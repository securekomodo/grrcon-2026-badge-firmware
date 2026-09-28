# GrrCON 2026 (15th) Anniversary Badge Firmware

Unmodified firmware read off the GrrCON 2026 Hak4Kidz challenge badge, with the notes you need to talk to the chip and put the original image back.

**Writeup:** [Hacking the GrrCON 2026 (15th) Anniversary Badge](https://www.redlinecybersecurity.com/blog/hacking-the-grrcon-2026-badge) on the Redline Cyber Security blog covers the whole thing: finding the serial pins with a multimeter, getting a console at 9600 baud, the hidden pin-12 trigger, the secret code, then dumping and reflashing the chip over UPDI with a plain USB-serial adapter.

| MCU         | Microchip ATtiny1616 (tinyAVR 1-series, SOIC-20) |
| ----------- | ------------------------------------------------ |
| Signature   | `0x1E9421`                                       |
| Flash       | 16 KB                                            |
| EEPROM      | 256 B                                            |
| Fuses       | `SYSCFG0 = 0xF6` — UPDI enabled, **not** locked  |
| Console     | USART0, 9600 8N1, TX on pin 9 (PB2), RX on pin 8 (PB3) |
| Programming | UPDI on pin 16 (PA0)                             |

## Files

| File | Description |
|---|---|
| `firmware/flash_original_16k.bin` | Full 16 KB flash image, as read |
| `firmware/eeprom_original_256.bin` | 256 B EEPROM (`GRRE` marker + stage byte) |
| `firmware/CHECKSUMS.sha256` | SHA-256 of the two images |

## Pinout

The `TX` arrow on the silkscreen points at pin 9 and is labelled from the chip's point of view (Table 5-1 in the datasheet).

```text
        ATtiny1616  SOIC-20
      +------\_/-------+
 VDD  | 1           20 |  GND
 PA4  | 2           19 |  PA3    LED
 PA5  | 3           18 |  PA2    USART0 RxD (alt)
 PA6  | 4           17 |  PA1    USART0 TxD (alt)
 PA7  | 5           16 |  PA0    RESET / UPDI
 PB5  | 6           15 |  PC3    input (pull-up)
 PB4  | 7           14 |  PC2
 PB3  | 8  <- RxD   13 |  PC1
 PB2  | 9  <- TxD   12 |  PC0    input (pull-up)  <- trigger
 PB1  | 10          11 |  PB0
      +----------------+
```

The micro USB port labelled `RECHARGE` is connected to nothing. It is a troll.

## Wiring

Power the badge from a regulated 3.3 V rail rather than the coin cells: a 3.3 V adapter against the 4.59 V battery rail corrupts both the console and UPDI, and a switched rail makes reboots painless.

### Serial console

```text
 badge pin 1  (VDD)  <---- regulated 3.3 V rail
 badge pin 8  (RxD)  <---- FTDI TXO
 badge pin 9  (TxD)  ----> FTDI RXI
 badge pin 12 (PC0)  ----  tap to GND to start the challenge
 badge pin 20 (GND)  ----- GND
```

Hold the button while powering on and the badge prints `Entering Sentinel Watch Mode...`. Ground pin 12 and it runs `probe... / contact... / handshake accepted.` and drops to the prompt. Never wire the badge's TX to the adapter's `TXO`: the idle-high output back-feeds the chip through its ESD diode.

### UPDI (SerialUPDI)

```text
  FTDI TXO --[ 5.1k ]--+---- PA0 / pin 16  (UPDI)
  FTDI RXI ------------+
  FTDI GND ----------------- GND / pin 20
  badge VDD / pin 1 <------- regulated 3.3 V rail
```

The series resistor lets the adapter's `TXO` and the chip share the one-wire bus. 220 Ω was too small (the adapter overpowers the chip's low); 4.7 k to 5.1 k works.

## What is in the image

The flash is an Arduino sketch built on megaTinyCore. `strings` over the 16 KB gives every message the badge can print: the boot banner, the prompt, `ACCESS GRANTED` / `ACCESS DENIED`, and the two links. There is no flag in the firmware; the badge sends you to the CTF server once you pass the gate.

The EEPROM holds the badge's state:

| Address | Bytes | Meaning |
|---|---|---|
| `0x00–0x03` | `47 52 52 45` (`GRRE`) | Magic marker, EEPROM initialised |
| `0x04` | `01` | Stage / solved flag, set after passing the gate |
| `0xEA` | `01` | Persistent counter |
| everything else | `FF` | Erased |

Because the stage byte lives in EEPROM, progress survives a power cycle.

## Dump

Read with `pymcuprog` over a 3.3 V USB-serial adapter wired as SerialUPDI.

```
pymcuprog ping -d attiny1616 -t uart -u /dev/ttyUSB0 -c 57600
pymcuprog read -d attiny1616 -t uart -u /dev/ttyUSB0 -c 57600 -m flash  -f flash_original_16k.bin
pymcuprog read -d attiny1616 -t uart -u /dev/ttyUSB0 -c 57600 -m eeprom -f eeprom_original_256.bin
```

## Restore

```
pymcuprog erase  -d attiny1616 -t uart -u /dev/ttyUSB0 -c 57600
pymcuprog write  -d attiny1616 -t uart -u /dev/ttyUSB0 -c 57600 -m flash -f firmware/flash_original_16k.bin
pymcuprog verify -d attiny1616 -t uart -u /dev/ttyUSB0 -c 57600 -m flash -f firmware/flash_original_16k.bin
```

Erase first: flash bits only clear (1 → 0), so writing over the old image without an erase fails verify. Check the images against `CHECKSUMS.sha256` before flashing.

## References

- [ATtiny1614/1616/1617 datasheet (DS40002204A, PDF)](https://ww1.microchip.com/downloads/en/DeviceDoc/ATtiny1614-16-17-DataSheet-DS40002204A.pdf)
- [Microchip ATtiny1616 product page](https://www.microchip.com/en-us/product/attiny1616)
- [megaTinyCore ATtiny x16 pin reference](https://github.com/SpenceKonde/megaTinyCore/blob/master/megaavr/extras/ATtiny_x16.md)
- [pymcuprog](https://github.com/microchip-pic-avr-tools/pymcuprog)
