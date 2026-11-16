# esp-idf-lib/max31855

[![Build examples](https://github.com/esp-idf-lib/max31855/actions/workflows/build.yml/badge.svg)](https://github.com/esp-idf-lib/max31855/actions/workflows/build.yml)
[![Build docs](https://github.com/esp-idf-lib/max31855/actions/workflows/build-docs.yml/badge.svg)](https://github.com/esp-idf-lib/max31855/actions/workflows/build-docs.yml)
[![Validation](https://github.com/esp-idf-lib/max31855/actions/workflows/validate-component.yml/badge.svg)](https://github.com/esp-idf-lib/max31855/actions/workflows/validate-component.yml)

Driver for the MAX31855 cold-junction compensated thermocouple-to-digital
converter.

* [Documentation](https://esp-idf-lib.github.io/max31855/)
* [Repository](https://github.com/esp-idf-lib/max31855)
* [Issues](https://github.com/esp-idf-lib/max31855/issues)
* [Discussions and questions](https://github.com/esp-idf-lib/core/discussions)
* [Component page at the ESP Component Registry](https://components.espressif.com/components/esp-idf-lib/max31855)

## Features

- Read-only SPI interface, mode 0, up to 5 MHz.
- 14-bit thermocouple temperature, 0.25 °C per bit, and 12-bit cold junction
  temperature, 0.0625 °C per bit.
- Thermocouple faults are reported: short to VCC, short to GND, open circuit.
- One transaction reads both temperatures and the fault flags.
- Works with the K-, J-, N-, T-, S-, R- and E-type versions of the chip.

## Wiring

The chip talks over SPI in one direction only: it drives SO, and the host drives
SCK and CS. MOSI is not connected.

| MAX31855 | ESP32                     |
|----------|---------------------------|
| VCC      | 3.3 V (3.0–3.6 V)         |
| GND      | GND                       |
| SCK      | `CONFIG_EXAMPLE_SCLK_GPIO`|
| CS       | `CONFIG_EXAMPLE_CS_GPIO`  |
| SO       | `CONFIG_EXAMPLE_MISO_GPIO`|
| T+ / T-  | the thermocouple          |

Put a 0.1 µF capacitor between VCC and GND close to the chip. Keep the chip away
from heat sources: it measures the temperature of its own cold junction, and a
warm board adds an offset to the reading. The thermocouple polarity matters,
T+ and T- cannot be swapped.

## Interface

The chip is read-only: the host pulls CS low and clocks 32 bits out of SO. The
first 14 bits are the thermocouple temperature (sign and 13 magnitude bits,
0.25 °C per bit), then a reserved bit, then the fault bit, then the 12-bit cold
junction temperature (sign and 11 magnitude bits, 0.0625 °C per bit), then two
reserved bits and the three fault flags.

The conversion runs continuously in the background, one cycle takes 70 ms
typically and 100 ms at most, and the result is updated only while CS is high.
Reading the chip more often than that returns the same value twice, and the
first conversion after the power is applied takes 200 ms.

## Faults

`max31855_get_temperature()` returns `ESP_ERR_INVALID_RESPONSE` when the
thermocouple is open, shorted to VCC or shorted to GND. In that case the
thermocouple temperature is not written at all — the data in the frame is
meaningless (`+2047.75 °C` for an open thermocouple) — while the cold junction
temperature is still valid and is written.

The flags are mandatory arguments, so the caller always knows what happened:

```c
float tc, cj;
bool scv, scg, oc;

esp_err_t err = max31855_get_temperature(&dev, &tc, &cj, &scv, &scg, &oc);
if (err == ESP_ERR_INVALID_RESPONSE)
{
    if (oc) ESP_LOGW(TAG, "the thermocouple is not connected");
    if (scv) ESP_LOGW(TAG, "the thermocouple is shorted to VCC");
    if (scg) ESP_LOGW(TAG, "the thermocouple is shorted to GND");
    ESP_LOGI(TAG, "the board is at %.2f °C", cj);
}
```

## Usage

```c
#include <max31855.h>

max31855_t dev;

// the SPI bus itself is initialized by the application
ESP_ERROR_CHECK(max31855_init_desc(&dev, SPI2_HOST, 1000000, GPIO_NUM_5));

float tc, cj;
bool scv, scg, oc;
ESP_ERROR_CHECK(max31855_get_temperature(&dev, &tc, &cj, &scv, &scg, &oc));

ESP_ERROR_CHECK(max31855_free_desc(&dev));
```

## API

| Function | Description |
|----------|-------------|
| `max31855_init_desc()` | Add the device to an initialized SPI bus |
| `max31855_free_desc()` | Remove the device from the bus |
| `max31855_get_temperature()` | Read the temperatures and the fault flags |

The driver sets the SPI mode to 0 and the CS setup time to one clock, because
the datasheet requires CS to fall at least 100 ns before the first clock edge.
Other fields of `dev.spi_cfg` can be filled before `max31855_init_desc()`, the
driver keeps them.

## Thread safety

The transactions are serialized by the SPI driver, so several tasks can read the
same device. Do not call `max31855_free_desc()` while another task is reading:
the removal is not serialized with the transfers.

## Example

See [examples/default](examples/default).

## Installation

```sh
idf.py add-dependency esp-idf-lib/max31855
```

## Support

For questions and discussions about the component, please use
[Discussions](https://github.com/esp-idf-lib/core/discussions)
at [esp-idf-lib/core](https://github.com/esp-idf-lib/core).

## Contributing

Please read [CONTRIBUTING.md](https://github.com/esp-idf-lib/core/blob/main/CONTRIBUTING.md)
at [esp-idf-lib/core](https://github.com/esp-idf-lib/core).
