# UART_design-
# 8-N-1 UART Transceiver with Parity (SKY130)

[![Tiny Tapeout](https://img.shields.io/badge/Tiny_Tapeout-SKY130-blue)](https://tinytapeout.com/)
[![GitHub Actions Status](https://github.com/AsimOS11/UART_design-/actions/workflows/gds.yaml/badge.svg)](https://github.com/AsimOS11/UART_design-/actions)

## Overview
This project implements a complete, synthesizable Universal Asynchronous Receiver-Transmitter (UART) designed for the SKY130 process node via the Tiny Tapeout shuttle. It provides full-duplex serial communication with configurable error detection.

## Key Features
* **Standard Protocol:** 8 Data Bits, 1 Stop Bit (8-N-1 format).
* **Hardware Parity Checking:** Configurable Even or Odd parity bit generation and validation.
* **Metastability Protection:** 2-stage synchronization flip-flops on the asynchronous RX line.
* **Glitch Rejection:** Auto-centering receiver logic that samples the exact middle of the bit period.
* **Target Specifications:** Designed for a `50 MHz` system clock and a `9600` Baud Rate.

## Pinout Mapping

| Pin | Direction | Signal Name | Description |
| :--- | :--- | :--- | :--- |
| `ui_in[7:0]` | Input | `tx_data_in` | 8-bit parallel data to be transmitted. |
| `uo_out[7:0]` | Output | `rx_data_out` | 8-bit parallel data successfully received. |
| `uio_in[0]` | Input | `rx_in` | Serial data receive line (RX). |
| `uio_in[1]` | Input | `tx_start` | Pulse HIGH to initiate transmission. |
| `uio_in[2]` | Input | `parity_sel` | `0` = Even Parity, `1` = Odd Parity. |
| `uio_out[3]` | Output | `tx_out` | Serial data transmit line (TX). |
| `uio_out[4]` | Output | `rx_valid` | Pulses HIGH for 1 clock cycle upon successful reception. |
| `uio_out[5]` | Output | `parity_error` | Driven HIGH if a parity mismatch is detected in the received frame. |

## How to Test (Hardware Loopback)
To verify the transceiver on the physical demo board:
1. Connect a physical jumper wire from `uio_out[3]` (TX) to `uio_in[0]` (RX).
2. Set your desired 8-bit message using the input dip switches (`ui_in`).
3. Select your parity mode on `uio_in[2]`.
4. Press the button mapped to `uio_in[1]` to pulse `tx_start`.
5. The transmitted data will loop back into the receiver and immediately display on the 7-segment output (`uo_out`).

## Simulation
The included testbench (`tb_uart.v`) performs an automated hardware loopback test across 5 different vectors, verifying data integrity, parity generation, and parity error flagging.
