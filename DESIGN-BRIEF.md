# Little Stack — Design Brief (draft v0.2)

*Little stack. Big SDR.*

## Goal

An open-source SDR, learning tool and development platform. It starts from HackRF One / HackRF Pro, fixes their weak points, and adds stacking, Raspberry Pi HAT compatibility and standalone (PortaPack-style) use.

## What this fixes

| HackRF One / Pro limit | This design |
|---|---|
| 8-bit samples | 12-bit full-duplex transceiver, 16-bit on the HF path |
| 20 MS/s, USB 2.0 | Up to 61.44 MS/s per channel; USB 3.0 and Ethernet |
| Half-duplex | Full duplex, 2×2 MIMO per module |
| One RF port (Opera Cake is an add-on) | Built-in antenna switch matrix, 4 or more ports per module |
| One's RF amp is easily damaged | Separate, protected RX and TX chains |
| One: plain crystal + Si5351C clock generator | TCXO, GPSDO option, low-jitter PLL, 10 MHz and PPS in/out |
| Small CPLD (One) / small FPGA (Pro) | Larger FPGA that does real DSP on the board |
| No switchable preselector | Switchable filter bank on every RX input |
| Expansion header leaves dev boards (ESP32 etc.) hanging off the side | Pi-pattern stacking, 40-pin GPIO passes through every layer |

## Architecture: the stack

```
[ UI board (optional) ]     new PortaPack: display, touch, encoder, audio, battery
[ GPS board (optional) ]    GPS-disciplined 10 MHz + PPS for the whole stack
[ Radio module N ]          full duplex, or RX-only scanner; repeat to add channels
[ Radio module 1 ]          FPGA + transceiver + front end + switch matrix
[ HF module (optional) ]    direct-sampling HF RX/TX
[ Brain board (optional) ]  Compute Module 5, Ethernet, USB, power
```

Standard Pi HATs fit at any level, because every board passes the 40-pin header through.

### Mechanical
- Every board uses the Raspberry Pi mounting-hole pattern (58 × 49 mm, M2.5) and puts the 40-pin header in the Pi position, so standard HATs stack on it.
- Target outline is 85 × 56 mm. The radio module may need to be larger; it keeps the hole pattern and header position either way. Check this in Phase 0.
- SMA ports on the board edges, shielding cans over the RF sections, a fixed standoff height per layer, and a thermal path (heat spreader, or a fan in the enclosure).

### Stack bus (define first; every board follows it)
- 40-pin Pi GPIO passed through unchanged.
- A separate high-density stack connector carrying:
  - 10 MHz reference, PPS, AD9361 multi-chip sync, trigger lines
  - high-speed serial lanes (FPGA SERDES) for sample data between modules and the brain
  - control: I²C/SPI, module address straps, interrupt, reset
  - power rails
- An ID EEPROM on every module (like the HAT EEPROM), so the stack detects what is fitted.

### Brain board (optional)
- Without it, each radio module runs from a PC over USB 3.0, like a HackRF.
- Raspberry Pi Compute Module 5: Linux on the device, native GPIO/HAT compatibility, Gigabit Ethernet, display output, USB host.
- PCIe link from the CM5 to the radio FPGA.
- Power: USB-C PD input (a stack can draw 20 W or more), battery input from the UI board, power sequencing, current limit per module.
- Ethernet: 1 GbE carries about 27–36 MS/s of streamed IQ. Leave room for 2.5/10 GbE later.

### Radio module (repeatable)
- **Transceiver:** one full-duplex chip per module, chosen in Phase 0 against the price target:
  - AD9361: RX 70 MHz–6 GHz, TX 47 MHz–6 GHz, 2×2 MIMO, 12-bit, up to 56 MHz bandwidth / 61.44 MS/s. About $280 per chip.
  - AD9363: the cheaper chip in the PlutoSDR. Officially 325 MHz–3.8 GHz and 20 MHz bandwidth.
  - LMS7002M: 2×2, 12-bit, 100 kHz–3.8 GHz. Low cost, but loses 3.8–6 GHz.
- **Price target:** a full-duplex radio module that sells for no more than a HackRF Pro (about $400).
- **FPGA:** has PCIe, SERDES and DSP blocks (Artix-7 or ECP5-5G class; choose in Phase 0, preferring open-toolchain support), with a DDR3 buffer.
- **USB 3.0 device port** (FX3 class), so one module can run straight from a PC like a HackRF, without the brain board.
- **FPGA DSP:** FFT sweep on the board for fast scanning (sends spectra, not raw IQ), decimation/channelizer, triggers.
- **Clock:** TCXO of 0.5 ppm or better, footprint for a GPSDO/OCXO, low-jitter PLL in place of an Si5351-class generator, 10 MHz and PPS in/out.
- **RX chain, per channel:** input protection (ESD + PIN-diode limiter) → switchable preselector filter bank → LNA with bypass → AD9361. It survives strong signals and accidental TX into the port.
- **TX chain, per channel:** its own path with its own PA, a harmonic filter, a directional coupler with a power detector (forward/reflected power, so it reads SWR), and thermal and VSWR protection. RX and TX never share an amplifier.
- **Antenna switch matrix (Opera Cake style):** 4 or more SMA ports per module. Modes: frequency (picks the antenna by band), time (rotates through antennas for direction finding), manual. Each port has a software-switched, current-limited bias-tee.
- **Calibration source:** an internal noise/tone source routed through the switch matrix, for phase-coherent calibration across modules (direction finding, beamforming).

### HF module (optional layer)
- **RX:** 100 kHz to about 60 MHz by direct sampling, with a 14–16-bit ADC at about 100–130 MS/s (RX888 class), an HF preselector, anti-alias low-pass filter and attenuator.
- **TX:** an HF transmit path, using a DAC or a mixed-signal chip (AD9866 class, as on the Hermes-Lite 2) and a low-pass filter bank. Choose the chip in Phase 0.
- Its coverage overlaps the radio module around 47–70 MHz.

### GPS board (optional)
- GPS receiver and GPS-disciplined oscillator. Drives the 10 MHz and PPS lines on the stack connector for every module.
- It uses the stack connector, not only the Pi header, because a clean 10 MHz reference needs its own line.

### UI board: a new PortaPack (optional top layer)
- Needs the brain board.
- Gamified apps in the spirit of PortaPack Mayhem, plus guided lessons for learning RF.
- IPS touchscreen, rotary encoder and buttons, audio codec with speaker and mic/headphone jack, microSD.
- Battery pack (2S Li-ion), USB-PD charging, fuel gauge.
- Its software runs on the CM5 under Linux. Mayhem firmware won't run as-is, because it targets the HackRF's LPC4320 MCU.

## How stacking works
- Each radio module is full duplex on its own. Every module you add brings more channels: N full-duplex modules give 2N RX and 2N TX channels.
- RX-only scanner modules: the same PCB with the TX parts not fitted. A cheaper way to add scan speed and bandwidth.
- Uses: tune modules to adjacent bands for wider total bandwidth, scan N bands in parallel (N times the scan speed), and run coherent arrays (direction finding, beamforming, MIMO).
- Limit: the brain board's link caps the total stream to the host (PCIe about 400 MB/s, GbE about 110 MB/s). Bigger stacks rely on FPGA decimation and FFT to cut the data down.

## Look and feel
- The board should look good, not just work: parts aligned to a grid, passives in matching orientations, symmetric placement where the circuit allows, even via stitching and via fences in clean rows, smooth traces (arcs or 45° bends), one consistent silkscreen font.
- Suggested finish: matte black soldermask with gold (ENIG) pads.
- Optional: a different soldermask colour for each board in the stack, like the layers of a burger.
- Logo: a small stacked-burger icon on every board (buns, two patties, cheese oozing, sauce dripping, lettuce). It is PCB art, coloured with the board's own materials: white silkscreen, exposed gold copper, bare board and mask over copper. Preview: [art/burger-icon-preview.svg](art/burger-icon-preview.svg).

## Software
- SoapySDR driver, GNU Radio blocks, SDR++ and SDRangel support, libiio (Linux already has an AD9361 driver). Open FPGA reference design.
- For learning: a documented example for each DSP block (filters, FFT, demodulators), on both the FPGA and the CPU.

## Power and thermal (estimates; verify in Phase 0)
- One radio module should run from a PC's USB 3 port (4.5 W) where possible, with a USB-C PD input for full-power TX and for stacks.
- Radio module about 3–8 W depending on TX power, CM5 up to about 7 W, UI board about 1–2 W. A 2-module stack with the UI board draws about 20 W, so it needs USB-PD input (45 W or more) and a fan or heat spreader. USB bus power, as the HackRF uses, is not enough.

## Licensing
- Open hardware; the license is not yet chosen (CERN-OHL-S or GPL). HackRF's design files are GPL-2.0, so any file reused from them keeps that license.

## Plan
Each board gets its own Flux project, and every board follows the Phase 0 stack-bus spec.

0. **Architecture:** part selection, stack connector pinout, mechanical outline, power budget.
1. **Radio module:** the highest-risk board (RF + FPGA + DDR, 8+ layers, BGA packages).
2. **Brain board.**
3. **HF module.**
4. **UI board.**

Risk: 6 GHz RF boards with an FPGA and DDR need controlled impedance and usually several prototype spins. The Flux agent produces drafts; an experienced person should review them before any order.

## Decisions
- **Brain:** optional. A radio module runs from a PC over USB 3.0; the CM5 brain board adds Linux on the device, Ethernet and standalone use.
- **Standalone:** a new PortaPack-style UI board with gamified apps.
- **Stack modules:** full-duplex modules, plus cheaper RX-only scanner modules.
- **Transceiver:** full duplex from the start, one chip per module. Final chip picked in Phase 0 against the price target.
- **Repository:** public on GitHub.

### Why not stack two HackRF-style boards for full duplex
- Two half-duplex boards need two sets of transceiver, mixer, ADC/DAC and MCU, and still give 8-bit samples at 20 MS/s.
- One full-duplex chip covers both directions at 12 bits and up to 56 MHz bandwidth. Extra boards in the stack then go to scan speed and bandwidth.

## Still open
- Final transceiver chip (Phase 0).
