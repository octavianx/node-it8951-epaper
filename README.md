# node-it8951

A Node.js driver for IT8951-based e-ink displays (Waveshare e-Paper HAT).

This is a port of the IT8951-ePaper [C driver from Waveshare](https://github.com/waveshareteam/IT8951-ePaper) combined with some concepts from the [Python PaperTTY IT8951 Driver](https://github.com/joukos/PaperTTY/blob/3ea8286903b98fac071285008b4cc05dd84c2121/papertty/drivers/driver_it8951.py)

## Key Features

* Fast, uses native GPIO libraries (rpio)
* Auto VCOM Detection
* Supports 1, 2, 4 Bits Per Pixel (BPP)
* Remappable Pins
* Configurable Buffer Size
* Support for 6" and 7.8" e-ink displays
* Raspberry Pi Zero 2W compatible

## Waveshare Official Compatibility

This driver has been verified against the [Waveshare official IT8951 driver](https://github.com/waveshareteam/IT8951-ePaper) (2026-01):

| Feature | This Driver | Waveshare Official | Status |
|---------|-------------|-------------------|--------|
| SPI Mode | `MODE0` | `BCM2835_SPI_MODE0` | Identical |
| BUSY Pin Logic | `0=busy, 1=idle` | `0=busy, 1=idle` | Identical |
| Reset Sequence | `HIGH→LOW→HIGH` | `HIGH→LOW→HIGH` | Identical |
| Init Flow | Reset→SysRun→GetInfo→I80CPCR→VCOM | Same | Identical |
| Clock Divider | `64` | `32` | More conservative (stable) |

### Improvements Over Original

| Improvement | Description |
|-------------|-------------|
| `spiSetDataMode(0)` | Explicit SPI MODE0 setting for stability |
| `CS pin HIGH` on init | Proper chip select initialization |
| `wait_for_ready()` before `set_vcom()` | Ensures device ready before VCOM write |
| `force6inch` option | Force 1448x1072 resolution when auto-detection fails |
| Slower SPI clock (64 vs 32) | Better stability on Pi Zero 2W |

## Supported Displays

| Display | Resolution | VCOM | Notes |
|---------|------------|------|-------|
| Waveshare 7.8" | 1872x1404 | 1380 | Check FPC cable for exact VCOM |
| Waveshare 6" HD | 1448x1072 | ~1530 | Use `force6inch: true` if needed |
| Kindle Paperwhite 6" | 1448x1072 | 1530 | Requires `force6inch: true` |

## Installation

```sh
npm install node-it8951
# or from GitHub
npm install github:octavianx/node-it8951-epaper
```

## Quick Start

```js
const IT8951 = require('node-it8951');

const display = new IT8951({
    MAX_BUFFER_SIZE: 32768,
    ALIGN4BYTES: true,
    VCOM: 1380  // Check your screen's FPC cable for the correct value
});

display.init();
display.clear();  // Clear to white
display.close();
```

## Example: 6" Display with Force Option

```js
const IT8951 = require('node-it8951');

const display = new IT8951({
    MAX_BUFFER_SIZE: 32768,
    ALIGN4BYTES: true,
    VCOM: 1530,        // Kindle Paperwhite voltage
    force6inch: true   // Force resolution to 1448x1072
});

display.init();
```

## Configuration Options

```js
const config = {
    MAX_BUFFER_SIZE: 32768,  // SPI buffer size (default: 4096, recommended: 32768)
    PINS: {
        RST: 11,   // Physical pin 11 (BCM 17)
        CS: 24,    // Physical pin 24 (BCM 8)
        BUSY: 18,  // Physical pin 18 (BCM 24)
    },
    VCOM: 1380,              // Display VCOM voltage (check FPC cable)
    BPP: 4,                  // Bits Per Pixel: 1, 2, or 4
    ALIGN4BYTES: true,       // Required for 1BPP on some devices
    SWAP_BUFFER_ENDIANESS: true,  // Little Endian format
    force6inch: false,       // Force 1448x1072 resolution
};
```

## API Reference

### Constructor
`new IT8951(config)` - Create a new driver instance with configuration options.

### Core Methods

| Method | Description |
|--------|-------------|
| `init()` | Initialize the display (required) |
| `draw(buffer, x, y, w, h, display_mode)` | Transfer buffer and refresh area |
| `displayArea(x, y, w, h, display_mode)` | Refresh a specific area |
| `clear(color=0xFF, display_mode)` | Clear screen (0xFF=white, 0x00=black) |
| `close()` | Shutdown driver and release GPIO |

### Display Modes

| Mode | Value | Description |
|------|-------|-------------|
| `INIT` | 0 | Fast, non-flashy, any gray to B/W |
| `DU` | 1 | Fast, non-flashy update |
| `GC16` | 2 | 16-level grayscale (flashy) |
| `GL16` | 3 | Little ghosting |
| `GLR16` | 4 | Heavy ghosting |
| `GLD16` | 5 | Flashy |
| `A2` | 6 | Flashy, any gray to any gray |
| `DU4` | 7 | 4-level gray, fast, minimal ghosting |

### Utility Methods

| Method | Description |
|--------|-------------|
| `wait(ms)` | Delay before next operation |
| `activate()` | Wake from standby |
| `standby()` | Enter low-power standby |
| `sleep()` | Enter deep sleep |
| `reset()` | Hardware reset |

## Example: Grayscale Gradient

```js
const IT8951 = require('node-it8951');

const display = new IT8951({ MAX_BUFFER_SIZE: 32768, ALIGN4BYTES: true, VCOM: 1380 });
display.init();

// Draw 16-level grayscale gradient
display.config.BPP = 4;
const colors = [0xFF, 0xEE, 0xDD, 0xCC, 0xBB, 0xAA, 0x99, 0x88,
                0x77, 0x66, 0x55, 0x44, 0x33, 0x22, 0x11, 0x00];
const segmentHeight = Math.trunc(display.height / colors.length);

for (let i = 0; i < colors.length; i++) {
    const buffer = Buffer.alloc(display.width * segmentHeight * display.config.BPP / 8, colors[i]);
    display.draw(buffer, 0, segmentHeight * i, display.width, segmentHeight);
}

display.wait(5000);
display.close();
```

## Important Notes

### 1BPP Mode
For 1BPP mode (black/white), some devices (e.g., Waveshare 6" HD) require:
- `ALIGN4BYTES: true` - Forces X and Width to be multiples of 32
- 8-bit buffers despite 1BPP definition (known workaround)

For devices supporting higher BPP, use those instead for better fidelity and performance.

See: https://www.waveshare.com/wiki/Template:EPaper_Codes_Descriptions-IT8951

### Root Permissions
This driver requires root access for `/dev/mem`. Run with `sudo`:
```sh
sudo node your-script.js
```

## Hardware Setup

Connect the IT8951 HAT to Raspberry Pi GPIO:

| IT8951 Pin | RPi Physical Pin | BCM |
|------------|------------------|-----|
| RESET | 11 | 17 |
| CS | 24 | 8 |
| BUSY | 18 | 24 |
| MOSI | 19 | 10 |
| MISO | 21 | 9 |
| SCLK | 23 | 11 |

## TODO
- [ ] Support rotation
- [ ] Test on more IT8951 devices
- [ ] Add TypeScript definitions

## Resources

- [Waveshare Official IT8951 Driver (C)](https://github.com/waveshareteam/IT8951-ePaper)
- [Python PaperTTY IT8951 Driver](https://github.com/joukos/PaperTTY)
- [IT8951 Specifications Document](https://www.waveshare.net/w/upload/1/18/IT8951_D_V0.2.4.3_20170728.pdf)
- [E-Paper Mode Declaration](http://www.waveshare.net/w/upload/c/c4/E-paper-mode-declaration.pdf)

## License

ISC
