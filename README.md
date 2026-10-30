# Real-Time Embedded Control – Buggy Project

C firmware for the [MikroElektronika **BUGGY**](https://www.mikroe.com/buggy) robot platform, running on a Microchip dsPIC33E microcontroller. The buggy is controlled over UART from a PC, monitors its surroundings with an infrared distance sensor, and automatically enters an emergency stop when an obstacle gets too close. It also streams sensor and status telemetry back to the host.

Developed as a group project for the **Embedded Systems** course of Robotics Engineering at the University of Genoa.

## Features

- **Finite state machine** with three states: `WAIT_FOR_START`, `MOVING` and `EMERGENCY`
- **Real-time control loop at 500 Hz** (2 ms period) driven by a hardware timer
- **Cooperative scheduler** running periodic tasks (LED blinking, telemetry transmission) at independent rates
- **Obstacle detection** with an IR distance sensor read through the ADC, converted to centimetres with a polynomial approximation
- **Automatic emergency stop** when an obstacle is closer than 15 cm, with hazard lights and a 5 s obstacle-free recovery period
- **Differential-drive motor control** via 4 PWM channels (10 kHz) driving an H-bridge, with speed and yaw-rate mixing and output normalization
- **Interrupt-driven UART** with circular TX/RX buffers and a custom NMEA-like message parser
- **Accelerometer readout over SPI** with a 5-sample moving average filter
- **Battery monitoring** through the ADC
- **Debounced push-button input** (INT1 interrupt + Timer 3) to start/stop the buggy manually

## System Overview

```
                 UART (commands / telemetry)
      PC  <───────────────────────────────────>  dsPIC33E
                                                    │
        ┌──────────────┬───────────────┬────────────┼────────────────┐
        │              │               │            │                │
   IR distance     Battery ADC    Accelerometer   Push button     H-bridge
   sensor (ADC)    (ADC)          (SPI)           RE8 (INT1)    (4x PWM / OC)
```

### State machine

| State | Behaviour | Transitions |
|---|---|---|
| `WAIT_FOR_START` | Motors off | Button press or `$PCSTT` → `MOVING` |
| `MOVING` | Drives with the reference speed and yaw rate | Button press or `$PCSTP` → `WAIT_FOR_START`; obstacle < 15 cm → `EMERGENCY` |
| `EMERGENCY` | Motors off, left/right lights blinking, button disabled | No obstacle for 5 s → `WAIT_FOR_START` |

While in `EMERGENCY`, start/stop commands are rejected with a negative acknowledgement.

## UART Protocol

Messages use the format `$<TYPE>,<payload>*`.

**Commands (PC → board)**

| Message | Description |
|---|---|
| `$PCREF,<speed>,<yaw>*` | Set reference speed and yaw rate (both in the range −100…100, mixed and clamped by the PWM driver) |
| `$PCSTT,*` | Start moving |
| `$PCSTP,*` | Stop moving |

**Telemetry (board → PC)**

| Message | Rate | Description |
|---|---|---|
| `$MDIST,<cm>*` | 10 Hz | Distance measured by the IR sensor |
| `$MACC,<x>,<y>,<z>*` | 10 Hz | Averaged accelerometer readings |
| `$MBATT,<volts>*` | 1 Hz | Battery voltage |
| `$MEMRG,<0\|1>*` | On event | Sent when entering (`1`) or leaving (`0`) the emergency state |
| `$MACK,<0\|1>*` | On event | Positive (`1`) or negative (`0`) acknowledgement of start/stop commands |

## Project Structure

| File | Purpose |
|---|---|
| `main.c` | System initialization, FSM, periodic tasks, UART message handling, main loop |
| `scheduler.c/.h` | Simple heartbeat-based cooperative task scheduler |
| `timer.c/.h` | Timer setup and blocking period wait helpers |
| `pwm.c/.h` | Output Compare PWM setup and differential-drive motor mixing |
| `adc.c/.h` | ADC configuration for the IR sensor and battery channels |
| `uart.c/.h` | UART configuration and circular buffer implementation |
| `parser.c/.h` | Byte-by-byte state-machine parser for incoming messages |
| `spi.c/.h` | SPI driver and accelerometer access |
| `ES_project_Group6.mc3` | MPLAB Code Configurator project file |
| `nbproject/`, `Makefile` | MPLAB X project configuration |

## Hardware

- **[MikroElektronika BUGGY](https://www.mikroe.com/buggy)**: battery-powered four-wheeled robot platform with mikroBUS sockets for Click boards
- Microchip dsPIC33E microcontroller <!-- TODO: add exact part number and the Clicker 2 / development board used -->
- Analog IR distance sensor <!-- TODO: add the Click board / sensor model -->
- 3-axis SPI accelerometer <!-- TODO: add the Click board / sensor model -->
- H-bridge motor drivers controlling the left and right motors
- Battery voltage monitored through a 1:3 divider on the ADC input
- Push button on pin RE8, status LED on RA0, side lights on RB8 and RF1

## Build and Flash

1. Install [MPLAB X IDE](https://www.microchip.com/mplab/mplab-x-ide) and the **XC16** compiler.
2. Clone the repository and open the project folder in MPLAB X.
3. Build the project (`Production → Build Main Project`).
4. Connect the programmer/debugger and flash the board (`Make and Program Device`).
5. Open a serial terminal on the board's UART port and send commands, for example:

```
$PCREF,50,0*
$PCSTT,*
```

## Authors
- [Paolo Laishram](https://github.com/paololais)
- [Andrea Loforese](https://github.com/AndLof)
- [Filippo Salterini](https://github.com/FilippoSalterini)
