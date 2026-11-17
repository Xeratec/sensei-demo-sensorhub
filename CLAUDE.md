<!--
SPDX-FileCopyrightText: 2026 ETH Zurich and University of Bologna
SPDX-License-Identifier: CC-BY-4.0

Authors:
- Philip Wiese <wiesep@iis.ee.ethz.ch>
-->

# SENSEI Sensor-Hub Demo Firmware

Pure environmental-acquisition firmware for the SENSEI platform (nRF5340 +
GAP9 + UT Sensor Shield v1). Companion repo to `sensei-demo-multimodal`
(adds GAP9 vision) and `sensei-demo-yolov5` (vision only).

## What this repo does

- `src_NRF/` runs on the nRF5340 application core (Zephyr / nRF Connect SDK
  v2.6.1). A single main loop wakes every `SAMPLING_TIME = 5000 ms`
  (`src_NRF/config.h`), sequentially polls every sensor, packs the
  readings into `sensor_values_t` (96 B, 4-byte aligned,
  `src_NRF/main.c`), and emits a CSV line on USB-CDC ACM (115200 baud) or
  the BLE Nordic UART Service.
- `src_GAP9/` is **a stub**: an I²C-slave memory emulation only. There is
  no neural network or sensor processing on the GAP9 in this demo. Do not
  describe this repo as running inference.

## Sensors and addressing

`src_NRF/i2c_helpers.c` lists the bus layout:
- I²C-A: LIS2DUXS12 0x19, BME680 0x76, MAX77654 PMIC 0x48, GAP9 slave 0x0A.
- I²C-B: SCD41 0x62, SGP41 0x59, ILPS28QSW 0x5C, BH1730FVC 0x29, AS7331
  0x74, MAX-M10S GNSS 0x42.

Power gating is per-sensor through GPIO (`src_NRF/bsp/pwr_bsp.c`); SCD41,
SGP41, AS7331 and BH1730FVC are powered up only for the read.

## Build

- nRF: `west build -b nrf5340_senseiv1_cpuapp` from `src_NRF/`, then
  `west flash`. Requires `SENSEI_SDK_ROOT` env var.
- GAP9: `gap init && gap run` (or `gap flash`) from `src_GAP9/`.
- Root `Makefile` only carries `format` (clang-format + yapf).

## Gotchas

- The CSV schema in `sensor_values_t` is the canonical dataset format used
  by the journal paper's data-acquisition section — don't rename fields
  without updating downstream ingestion (`docs/plot_timeseries.py`).
- SAMPLING_TIME is wall-clock; the AS7331 measurement alone takes ~2 s
  (`2^time` ms, time=11), so the slack inside a 5 s window is small.
- `k_uptime_get_32()` wraps every ~49 days — use it only as a relative
  timestamp, not absolute time.
