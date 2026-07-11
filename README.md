# UARTSim

A C++ simulation of a UART (Universal Asynchronous Receiver/Transmitter) peripheral — modeling register-level behavior, bit-accurate frame construction, fixed-size circular FIFOs, and event-driven callbacks, without any real hardware.

## Overview

UARTSim reproduces the core mechanics of a real UART chip in software:

- Two independent `Uart` instances, connected to each other like devices on a physical serial link
- Configurable baud rate, parity (none/even/odd), data bits (7 or 8), stop bits (1 or 2), and loopback mode
- Hardware-style `CONTROL`, `STATUS`, and `BAUD_DIV` registers, updated exactly the way real UART registers are
- Fixed-size 16-byte circular FIFO buffers for both transmit and receive, with overflow detection
- A shared clock (`Bus`) that advances every attached UART by one cycle per `tick()`
- Event callbacks (`std::function`) for byte-received, buffer-overflow, and framing-error conditions, in addition to the traditional poll-based status API
- An interactive CLI harness that walks through configuring a UART, transmitting a message, and printing a full bit-level trace of every frame

## Why

Getting UART framing, buffering, and register behavior right in software — before it ever touches real hardware — is far cheaper than debugging it after board bring-up. This project is a self-contained testbench for exactly that: bit-level framing, FIFO buffering, and register/event modeling, all inspectable from the command line.

## Project structure

```
.
├── include/
│   ├── Bus.hpp             # shared clock; ticks every attached Uart
│   ├── Uart.hpp            # the UART peripheral: registers, buffers, framing, callbacks
│   └── CircularBuffer.hpp  # generic fixed-size ring buffer template
└── src/
    └── main.cpp            # interactive CLI harness / demo
```

## Architecture

No inheritance or virtual dispatch is used anywhere — every relationship is composition:

| Class | Responsibility |
|---|---|
| `CircularBuffer<T, Capacity>` | Generic, fixed-capacity ring buffer (`push`/`pop`, O(1), no dynamic allocation). Used as the underlying FIFO for both TX and RX. |
| `Uart` | Owns a `Config`, a `Registers` block, two `CircularBuffer<uint8_t, 16>` members (TX/RX), a `peer_` pointer to the connected UART, and three callback slots. |
| `Bus` | Holds a list of attached `Uart*` and advances all of them by one clock cycle per `tick()` call. |
| `main.cpp` | CLI harness — collects configuration, wires two UARTs together via a `Bus`, drives transmission, and prints diagnostics. |

### Register model

```
Registers { uint32_t control; uint32_t baudDiv; uint32_t status; }
```

**CONTROL** — bit0 `ParityEnable`, bit1 `ParityOdd`, bit2 `TwoStopBits`, bit3 `SevenDataBits`, bit4 `Loopback`
**STATUS** — bit0 `TxBusy`, bit1 `RxReady`, bit2 `BufferOverflow`
**BAUD_DIV** — `16,000,000 / baudRate` (simulated 16 MHz base clock)

### Frame format

Every transmitted byte is framed as: **Start bit (0) → data bits, LSB first → optional parity bit → stop bit(s) (1)**, matching real asynchronous serial framing.

### Event callbacks

```cpp
uart.setOnByteReceived([](std::uint8_t byte) { /* ... */ });
uart.setOnBufferOverflow([]() { /* ... */ });
uart.setOnFramingError([]() { /* ... */ });
```

- **Byte received** — fires the moment a byte lands in an RX buffer.
- **Buffer overflow** — fires when a TX or RX FIFO push fails because the buffer (16 bytes) is full.
- **Framing error** — fires on the receiving UART when its baud rate, data bits, or stop bits don't match the sender's.

## Building

Requires a C++17 compiler.

```bash
g++ -std=c++17 -Wall -Wextra -Iinclude src/main.cpp -o uartsim
```

## Running

```bash
./uartsim "Hello UART"
```

You'll be prompted for parity, stop bits, data bits, baud rate, and loopback mode — press Enter on any prompt to accept the shown default. The program then prints, for every character sent:

- A full packet trace (character, ASCII, binary, data bits, parity calculation)
- The assembled frame (`Start | Data | Parity | Stop`)
- Any status-register or callback events triggered on that cycle

...followed by a transmission/reception summary and a full register dump.

### Example output

```
-- TRANSMIT --
Character : 'H'
ASCII     : 72
Binary    : 0100 1000
Data bits : 0 0 0 1 0 0 1 0  (LSB first)
Frame     : 0 00010010 1        (Start | Data | Stop)
  [CALLBACK] UART1 received byte: 'H' (0x48)
  STATUS RX changed: [0000 0010] TX_BUSY=0 RX_READY=1 OVERFLOW=0
...
Received Message : "Hi"
```

## Topics covered

- Templates & generic programming (`CircularBuffer<T, Capacity>`)
- Pointers & references (`peer_`, `std::vector<Uart*>`, output parameters)
- Bit manipulation (register bitfields, LSB-first framing, parity calculation)
- Data structures (fixed-size circular FIFO with overflow detection)
- Encapsulation & composition (no inheritance/virtual dispatch)
- Lambdas & `std::function` (event callbacks)

## Authors

- Ali Hassan — 25ins-04020
- Inamullah Khan — 25ins-04008
- Abdullah Zeb — 25ins-04001
