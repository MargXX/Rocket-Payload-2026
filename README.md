# Payload Flight Logger v2 (RP2040, Ada)

New flight firmware for the Brown Rocketry IREC 2025-2026 payload hardware. Same board and wiring as the April 2026 flight; only the firmware changes. The goal is repeatable, higher-quality flight data and a clean before/after comparison against the first flight.

**Status:** design phase. No flight code yet.

See [`../terrarium_pico/README.md`](../terrarium_pico/README.md) for the original flight firmware, its recovered results, and the hardware notes this version builds on.

---

## Why a v2

The first flight logged successfully but left known gaps, all recorded in the original README:

- **Rate drift.** The loop ran work plus a flat 6 ms delay, so cadence was set by bus and flash timing. It averaged 6.57 Hz and drifted to 5 Hz, against a 119 Hz sensor output rate.
- **Aliased flight.** At 5 Hz the boost phase and deployment transients were barely sampled.
- **Blocking flash.** Sector erase (around 400 ms observed) stalled the whole loop.
- **No launch awareness.** One continuous low-rate log served both a 25 hour ground campaign and a 45 second flight.

v2 targets each of these directly.

---

## Planned architecture

### Timing: timer interrupt tick

- A hardware timer alarm fires at a fixed period near the LSM9DS1 output rate (119 Hz, about 8.4 ms).
- The ISR only sets a tick flag and re-arms the alarm. Next alarm = previous alarm + period, so the rate cannot drift.
- If the ISR fires while the previous tick is still unhandled, that is an overrun. Overruns are counted and logged, not hidden.

### Main loop: producer and consumer

```
loop
   sleep until interrupt (WFI)
   if tick flag set:
      clear flag
      read sensors over I2C          -- producer
      push record into ring buffer
      run launch trigger check
   run one step of flash consumer    -- never blocks
end loop
```

- **Producer:** I2C reads, record packing, push into the ring.
- **Consumer:** drains the ring to SPI flash one page at a time. Written as a small state machine that checks the flash busy bit and moves on if busy, so an erase never delays sampling.
- Producer and consumer run in the same context, so the ring buffer needs no locking. The only state shared with the ISR is the tick flag.

### Ring buffer

- Fixed array of 28-byte sample slots in RAM, with a head (next slot to write) and a tail (oldest sample not yet in flash).
- **Before launch:** when full, the oldest sample is overwritten. The ring holds the pre-launch history window.
- **After launch:** the ring absorbs flash stalls. Overwriting here would lose flight data, so a full ring is counted as an error.

### Launch trigger

- Acceleration magnitude above a threshold for N consecutive samples.
- Threshold chosen from the recovered April flight and ground campaign data, not guessed.
- The samples that satisfied the trigger are already in the ring, so the debounce costs nothing.

### Flight states

Draft, to be finalized:

```
BOOT -> ARMED (filling ring) -> LAUNCH DETECTED -> LOGGING -> DONE / FLASH FULL
```

### Driver separation

Unlike v1, code is split into layers. Each layer depends only on the one below.

| Layer | Contents |
| --- | --- |
| Application | Ring buffer, trigger detector, state machine, consumer |
| Device drivers | LSM9DS1, BMP390, W25Q128 |
| Peripherals | I2C, SPI, UART, timer |

The ring buffer and trigger logic contain no hardware access, so they can be tested on a host machine by replaying the recovered April flight log through them.

---

## Planned flash format changes

- **Config header page at boot:** sensor full-scale ranges and output rates, so a recovered log is self-describing.
- **Timestamp epoch counter:** `TIMERAWL` wraps about every 71 minutes. An epoch count incremented on wrap removes post-processing stitching.
- **No blind erase at boot:** a brownout reset mid-flight must not erase flight data. Boot checks flash contents before erasing anything.

---

## Open design questions

- [ ] Ring size: pre-launch window length vs. worst-case flash stall. Which one sets the size?
- [ ] Full vs. empty rule for the ring (spare slot or count)
- [ ] Trigger threshold and N, from recovered data
- [ ] Where the trigger check runs and how detection latency is affected
- [ ] Consumer state machine states and transitions
- [ ] Tick flag: boolean or counter, and how it is declared for safe ISR sharing in Ada
- [ ] How `light-cortex-m0p` exposes the vector table and handler attachment
- [ ] Whether the 4 I2C transactions fit comfortably in one 8.4 ms period (estimate, then measure)
- [ ] Boot-time flash check policy

---

## Measurements to capture

- I2C read time per tick (spare GPIO high/low around the reads, measured on a scope or logic analyzer)
- Tick jitter (GPIO toggle in the ISR)
- Overrun count over a full ground run
- Achieved logging rate vs. v1 (6.57 Hz mean, 5 Hz final)

---

## Hardware

Unchanged from the April 2026 flight.

| Component | Part | Interface | Address / Pins |
| --- | --- | --- | --- |
| Microcontroller | Raspberry Pi Pico H (RP2040) | | |
| IMU | ST LSM9DS1 (accel / gyro / mag) | I2C | AG `0xD6`, Mag `0x3C` (8-bit form) |
| Barometer | Bosch BMP390, Adafruit breakout | I2C | `0xEE`, SDO pulled high |
| Flash | Winbond W25Q128, Adafruit 5634 breakout | SPI | CS on GP17 |
| Power | 3x AA in series to VSYS | | |

### Pin assignments

| Function | GPIO |
| --- | --- |
| I2C SDA | GP0 |
| I2C SCL | GP1 |
| UART TX (debug) | GP8 |
| UART RX (debug) | GP9 |
| SPI MISO | GP16 |
| SPI CS (manual GPIO) | GP17 |
| SPI SCK | GP18 |
| SPI MOSI | GP19 |
| Timing debug GPIO | TBD (free: GP20, GP21, GP22) |

### Wiring

```
Power:
  Pico 3V3_OUT --> LSM9DS1 VCC, BMP390 VIN, W25Q128 VIN
  Pico GND     --> LSM9DS1 GND, BMP390 GND, W25Q128 GND

I2C (shared bus):
  GP0 (SDA) --> LSM9DS1 SDA + BMP390 SDA
  GP1 (SCL) --> LSM9DS1 SCL + BMP390 SCL

SPI flash:
  GP16 (MISO) <-- W25Q128 DO
  GP19 (MOSI) --> W25Q128 DI
  GP18 (SCK)  --> W25Q128 CLK
  GP17 (CS)   --> W25Q128 CS

UART debug (115200 baud, UART1):
  GP8 (TX) --> USB-UART RX
  GP9 (RX) <-- USB-UART TX
  GND      --- USB-UART GND

SWD:
  Probe SWCLK --> Pico SWCLK pad
  Probe SWDIO --> Pico SWDIO pad
  Probe GND   --- Pico GND
```

LSM9DS1 interrupt pins are not wired. Not needed for this design; reserved for a later iteration (data-ready or sensor FIFO).

---

## Scope

**In for this flight:** timer tick ISR, ring buffer with pre-launch history, non-blocking flash consumer, accel trigger, config header page, timestamp epoch counter.

**Later:** LSM9DS1 FIFO and data-ready interrupt, second core, DMA, register-level I2C.

---

## Build and flash

```
alr build
openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg \
  -c "adapter speed 5000" \
  -c "program bin/<main>.elf verify reset exit"
```

On macOS the USB-UART appears as `/dev/cu.usbserial-*` or `/dev/cu.usbmodem*` (check with `ls /dev/cu.*`):

```
screen /dev/cu.<device> 115200
```

---

## References

- [RP2040 datasheet](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf) (TIMER chapter for alarms and interrupts)
- Cortex-M0+ Devices Generic User Guide (exceptions, NVIC, `WFI`)
- [LSM9DS1 datasheet](https://www.st.com/resource/en/datasheet/lsm9ds1.pdf)
- [BMP390 datasheet](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmp390-ds002.pdf)
- [W25Q128 datasheet](https://www.winbond.com/resource-files/w25q128jv%20revf%2003272018%20plus.pdf)