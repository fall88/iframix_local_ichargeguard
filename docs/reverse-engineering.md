# Reverse engineering the stock firmware

How the pin map in the main README was worked out: dump the flash, load the
app into Ghidra, and let Claude Code drive Ghidra through GhidraMCP to find,
decompile and rename the relevant functions.

No vendor firmware is published here. To follow along, dump your own device.

## 1. Dump and split the firmware

Back up the whole 4 MB flash first. It is also your way back to stock.

```
esptool.py --chip esp32c2 --port /dev/ttyUSB0 read_flash 0x0 0x400000 stock.bin
```

Extract the application partition (about 1.7 MB here) and split the ESP-IDF
app image into its load segments. Each segment has a load address, and Ghidra
needs them as separate memory blocks:

| Segment | Load address | Contents |
| --- | --- | --- |
| 0 | `0x3C100020` | Flash-mapped read-only data (strings, constants) |
| 1 | `0x3FCAE6C0` | DRAM initial data |
| 2 | `0x3FCB3720` | DRAM initial data |
| 3 | `0x40380000` | IRAM code |
| 4 | `0x42000020` | Flash-mapped code (most of the application) |
| 5 | `0x4038C288` | IRAM code |
| 6 | `0x4038CDC0` | IRAM code |

## 2. Set up Ghidra

1. Create a project and import each segment as a raw binary at its load address.
2. Language: **RISC-V, 32-bit, little-endian (RV32IMC)**.
3. Apply ROM symbols from ESP-IDF's linker scripts
   (`components/esp_rom/esp32c2/ld/esp32c2.rom*.ld`). They name the ROM helpers
   the app calls, for example `memcpy` at `0x4000048C`, `strcmp` at `0x400004A0`,
   `__divdf3` at `0x400008F4` and `ets_delay_us` at `0x40000044`.

## 3. Drive Ghidra from Claude Code with GhidraMCP

[GhidraMCP](https://github.com/LaurieWired/GhidraMCP) (release 1.4) has two parts:

- a Ghidra plugin that serves the open program over HTTP on port 8080;
- `bridge_mcp_ghidra.py`, an MCP server that connects that HTTP API to Claude Code.

Register the bridge in Claude Code as a stdio MCP server:

```json
"ghidra": {
  "type": "stdio",
  "command": "~/ghidra-mcp-venv/bin/python",
  "args": ["~/GhidraMCP-release-1-4/bridge_mcp_ghidra.py",
           "--ghidra-server", "http://127.0.0.1:8080/"]
}
```

**Gotcha:** if `/mcp` reports `Failed to reconnect to ghidra: CONNECTION_CLOSED`
while Ghidra is running, the bridge is crashing at startup. Version 2.x of the
`mcp` Python package renamed `FastMCP`, which bridge 1.4 imports. Pin it:

```
~/ghidra-mcp-venv/bin/pip install 'mcp<2'
```

Claude could then list strings, follow cross-references, decompile, rename
functions and leave comments directly in the Ghidra project.

## 4. Find the GPIO driver functions

ESP-IDF compiles its log strings into the firmware, so error messages point
straight at driver functions:

| Function | Address | Found through |
| --- | --- | --- |
| `gpio_set_level` | `0x42077FE6` | Uses both "GPIO output gpio_num error" and "gpio_set_level" |
| `gpio_config` | `0x42078116` | Uses the "gpio_config" string and the `GPIO[%lu]\| InputEn: ...` log format |

Ghidra had not defined the "InputEn" format as a string, so a string search for
it found nothing. Searching for the function-name string "gpio_config" did. The
helpers inside `gpio_config` (input, open-drain, output, pull-up, pull-down,
interrupt setup) are called in the same order as in the ESP-IDF 5.2 source,
which makes them easy to name.

`gpio_set_level` is reached through ten two-instruction wrappers, each setting
one pin high or low:

```
c.li  a1, 1        # level
c.li  a0, 2        # GPIO number
j     gpio_set_level
```

Following the callers of each wrapper showed what every pin does.

## 5. What the code does with the pins

- `gpio_outputs_init` configures GPIO2-6 as plain outputs (mask `0x7C`, no
  pulls, no interrupts).
- A 1-second `led_timer` callback refreshes the status LEDs and applies a
  fail-safe: if WiFi has been down for more than 5 seconds, it forces charging on.
- The stock firmware's MQTT code also sets GPIO2, and forces charging on when
  its connection drops.
- `adc_timer_callback` averages 16 samples of ADC1 channels 0 and 1 every
  second and scales them (the ROM float routines `__mulsf3`, `__divdf3` and
  `__muldf3` gave the constants away):
  - current = mean mV (GPIO0) / 1000 x 0.83
  - voltage = mean mV (GPIO1) / 1000 x 4.0
- The firmware never calculates a battery percentage.

### LED strip: WS2812 over SPI

The ESP32-C2 has no RMT peripheral, the block usually used for WS2812 timing,
so the vendor drives the strip with SPI:

- `spi_bus_initialize(SPI2_HOST, ...)` with MOSI on GPIO7 and no clock, MISO or
  chip-select pins;
- 8 MHz clock, one SPI byte per WS2812 bit: `0xF0` = 1, `0xC0` = 0;
- 26 LEDs x 24 bits = a 624-byte frame, followed by an 80 us low reset.

## 6. Tools used

- Ghidra with GhidraMCP 1.4
- Claude Code (Anthropic) as the MCP client
- ESP-IDF `riscv32-esp-elf-objdump` for code Ghidra had not defined as functions
- `esptool.py` for dumping and flashing
- Python and Pillow for decoding constants and cleaning photo metadata
