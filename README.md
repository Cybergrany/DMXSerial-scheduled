# DMXSerial

[![arduino-library-badge](https://www.ardu-badge.com/badge/DMXSerial.svg?)](https://www.ardu-badge.com/DMXSerial)
[![Arduino Library Checks](https://github.com/mathertel/DMXSerial/actions/workflows/arduino-checks.yml/badge.svg)](https://github.com/mathertel/DMXSerial/actions/workflows/arduino-checks.yml)


This is a library for sending and receiving DMX codes using the Arduino platform.
You can implement DMX devices and DMX controllers with this library.

The DMX communication implemented by the DMXSerial library relies completely on the hardware support of a builtin USART / Serial port interface
by using interrupts that handle all I/O transfer in the background.

There is a full 512 byte buffer allocated to support all possible DMX values of a line at the same time. 

Since Version 1.5.0 the original ATmega168 based implementation was refactored and enhanced
to support also other processor architectures like the ATMEGA4809.

## Scheduled controller mode

This fork adds `DMXControllerScheduled` for projects that need DMX output to
share a small microcontroller with another timing-sensitive protocol.

The original `DMXController` mode is unchanged: after `DMXSerial.init(DMXController)`
the library continuously sends DMX frames in the background. That is the normal
DMX behaviour and remains the right choice when DMX is the main job of the
controller.

`DMXControllerScheduled` initializes the same DMX hardware and shadow buffer, but
it does not continuously restart frames. A frame is transmitted only when the
application calls `sendFrame()` or `sendFrameBlocking(timeoutMs)`. When that
frame completes, TX interrupts are stopped and the direction pin is returned to
receive/tri-state (`DmxModeIn`) until the next explicit frame request.

The purpose is to create predictable idle windows for protocols such as PJON
`SoftwareBitBang`, where long runs of continuous DMX TX interrupts can otherwise
delay packet receive/ACK handling.

### Basic one-frame send

```cpp
#include <DMXSerial.h>

void setup() {
  DMXSerial.init(DMXControllerScheduled);
  DMXSerial.maxChannel(32); // Keep frames short when only low fixture channels are used.

  DMXSerial.write(1, 255);
  DMXSerial.write(2, 128);

  // Start one frame and return immediately. isSending() stays true until the
  // frame has completed and scheduled mode has released the driver.
  DMXSerial.sendFrame();
}

void loop() {
  if (!DMXSerial.isSending()) {
    // Safe place for other protocol work.
  }
}
```

### Blocking one-frame send

```cpp
bool sent = DMXSerial.sendFrameBlocking(10);
```

`sendFrameBlocking(timeoutMs)` waits until the frame completes or the timeout
expires. A `false` return means either no frame was started or the wait timed
out. It does not abort a frame that is already being transmitted; check
`isSending()` before opening a timing-sensitive receive window.

### Burst plus keepalive pattern

Some fixtures expect a regular DMX signal, so scheduled mode should normally be
paired with a small burst after changes and a periodic keepalive frame while
idle. Keep `maxChannel()` as low as your fixture patch allows so each scheduled
frame is short.

```cpp
#include <DMXSerial.h>

constexpr uint8_t kBurstFrames = 3;
constexpr uint16_t kMaxChannels = 32;
constexpr unsigned long kKeepaliveMs = 250;

uint8_t pendingFrames = 0;
unsigned long lastFrameAt = 0;

void queueDmxChange(uint16_t channel, uint8_t value) {
  DMXSerial.write(channel, value);
  pendingFrames = kBurstFrames;
}

void setup() {
  DMXSerial.init(DMXControllerScheduled);
  DMXSerial.maxChannel(kMaxChannels);
}

void loop() {
  const unsigned long now = millis();

  if (DMXSerial.isSending()) {
    return;
  }

  if (pendingFrames > 0 || now - lastFrameAt >= kKeepaliveMs) {
    if (DMXSerial.sendFrame()) {
      lastFrameAt = now;
      if (pendingFrames > 0) {
        --pendingFrames;
      }
    }
    return;
  }

  // Other protocol receive/service work can run here between DMX frames.
}
```

### PJON SoftwareBitBang coexistence

For PJON `SoftwareBitBang`, keep the receive/service work outside active DMX
frames. Decode incoming PJON packets quickly, update the DMX shadow buffer with
`DMXSerial.write()`, then return so PJON can ACK before starting another DMX
frame from the main loop.

```cpp
void loop() {
  if (DMXSerial.isSending()) {
    return;
  }

  bus.receive(1000); // microsecond-scale receive window, tune for your bus.

  if (pendingDmxCommand) {
    applyCommandToDmxBuffer();
    pendingFrames = 3;
  }

  serviceScheduledDmxFrames();
}
```

Scheduled mode is intentionally opt-in. Existing sketches that use
`DMXController`, `DMXReceiver`, or `DMXProbe` should keep their previous
behaviour.


## Supported Boards and processors

The supported processors and Arduino Boards are:
* Arduino UNO, Ethernet, Fio, Nano and Arduino Pro, Arduino Pro Mini (ATmega328 or ATmega168)
* Arduino Mega2560 (ATmega2560)
* Arduino Leonardo (Atmega32U4)
* Arduino nano Every (ATMEGA4809)
* ATmega8 boards - experimental

Other compatible boards my work as well.

You can find more detail on this library at http://www.mathertel.de/Arduino/DMXSerial.aspx.


## Compile for other Serial ports

The original library was written for Arduino 2009 and Arduino UNO boards that use the ATmega328 or ATmega168 chip.
These chips only have a single Universal Synchronous and Asynchronous serial Receiver and Transmitter (USART) a.k.a. the Serial port in the Arduino environment.
The DMXSerial library uses this port for sending or receiving DMX signals by default.

This conflicts a little bit with the usage of the programming mode the Arduino offers through then USB ports or serial interfaces but it is possible to build compatible DMX shields that don’t interfere with this usage if done right. The DMXShield is an example of this.

By using the Arduino Leonardo, MEGA or Every board you can write debugging information to the Serial port in your sketch for debugging or data exchange purpose as the processor supports multiple ports.
So maybe a DMX diagnostic sketch and debugging output in your code can profit from that.


**Arduino Leonardo, Arduino Esplora**

When compiling for a Arduino Leonardo board the DMXSerial library will choose the **Serial1** port by default for DMX communication.

The Arduino Leonardo board uses a chip that has a USB 2.0 port available in addition to the USART. Therefore the “Serial” port is routed through the USB port that behaves like a serial port device and not the built-in USART. If you like to address the real Serial port you have to go with the Serial1.

When you look at the hardware and the registers definitions and manual for the processor the USART0 still exists but the definitions for addressing the registers have changed
(for example USART_RX_vect is now USART0_RX_vect). Therefore some adjustments had to be done. 


**Arduino MEGA 2560**

When using the chip on the Arduino MEGA 2560 board you have more than one USART available and maybe you wish to use port 1 instead of port 0.
This can be done by enabling the definitions for port1 in the library in file `src\DMXSerial_avr.h` just uncomment the line

```CPP
#define DMX_USE_PORT1
```

**MEGAAVR processors like 4809 in Arduino nano Every**

For this processor the USART1 is used for DMX.


**ATmega8 boards**

I added the definitions of boards based on the ATmega8 too for experimental purpose. If you find problems with these boards, leave me a note please.



## Supported Shields

A suitable hardware is the Arduino platform plus a shield for the DMX physical protocol implementation.
You can find such a shield at: http://www.mathertel.de/Arduino/DMXShield.aspx.

Without or some modification this library can also used with other DMX shields
that use the Serial port for connecting to the DMX bus.


## License

Copyright (c) 2005-2020 by Matthias Hertel,  http://www.mathertel.de/

The detailed Software License Agreement can be found at: http://www.mathertel.de/License.aspx
