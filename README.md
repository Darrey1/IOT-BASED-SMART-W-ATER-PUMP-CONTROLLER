# Smart Water Pump Controller

An Arduino-based water tank controller that measures level with an ultrasonic sensor, switches a pump automatically at configurable thresholds, and is fully controllable over SMS through a GSM module. No internet, no app, no smartphone required.

---

## What it does

- Measures tank level with an HC-SR04 ultrasonic sensor, filtered to reject bad readings
- Switches the pump on at low level and off at high level, with no user involvement
- Sends SMS alerts to the owner when the tank hits either threshold
- Accepts SMS commands back, so the pump can be controlled from any phone, anywhere
- Stores the alert number in EEPROM, so it survives a power cut
- Recovers its state on boot after a power interruption
- Shows level, pump state and system status on a 16x2 LCD, with LED and buzzer indication

## Why SMS

The site this was built for has unreliable mains power and no dependable internet. A Wi-Fi or cloud based controller would be offline exactly when it matters most.

GSM only needs cellular coverage, which is available almost everywhere the pump is. SMS works on the cheapest phone on the network, so the owner does not need a smartphone or a data plan to see that the tank is low or to switch the pump on from another town.

---

## Hardware

| Component | Purpose |
|---|---|
| Arduino Nano | Control logic, 2KB SRAM |
| HC-SR04 ultrasonic sensor | Distance to water surface |
| SIM900A GSM module | SMS send and receive |
| 16x2 LCD | Level, pump state, system messages |
| Relay module | Pump switching |
| DC pump | Load |
| Red and green LEDs | Level and pump state indication |
| Buzzer | Audible alert at thresholds |

> **Power note.** The GSM module draws current spikes during transmission that the Arduino's regulator cannot supply. It needs its own supply with a shared ground. Powering it from the Arduino's 5V pin causes brownout resets that look like random crashes.

---

## Control logic

| Condition | Action |
|---|---|
| Level at or below 20% | Pump on, low alert SMS, red LED, buzzer |
| Level at or above 99% | Pump off, full alert SMS, green LED, buzzer |
| Between thresholds | Pump holds its current state |

The pump does not toggle at a single set point. It turns on at the low threshold and stays on until the high threshold is reached, which prevents rapid cycling when the water surface sits near a boundary.

### Sensor filtering

Ultrasonic sensors return occasional garbage, particularly off a moving water surface. Each measurement takes five readings and uses the median. A single bad reading cannot switch the pump.

### Boot recovery

Power cuts are expected. On boot the controller re-reads the tank, restores the alert number from EEPROM, and re-establishes the correct pump state for the current level before anything else runs.

---

## SMS commands

Text any of these to the SIM in the module. Matching is case insensitive.

| Command | Effect |
|---|---|
| `STATUS` | Replies with current level, pump state and mode |
| `PUMPON` | Forces the pump on |
| `PUMPOFF` | Forces the pump off |
| `AUTO` | Returns control to automatic threshold logic |
| `SETNUM +234...` | Changes the alert number, saved to EEPROM |
| `HELP` | Lists available commands |

---

## Engineering notes

Most of the real work on this project was in the GSM layer. These are the failures that took the longest to find, kept here because they are not obvious and are not in the datasheets.

**SMS sent into a void.** The original send routine issued `AT+CMGS` and then waited a fixed delay before writing the message body. The module replies with a `>` prompt when it is ready, and that prompt does not arrive on a predictable schedule. Writing the body early means the module discards it. There is no error returned, so the code reports success and nothing is delivered. The fix is to wait for the prompt character rather than for a timer.

**Commands worked once, then stopped.** The send routine drained the serial buffer, and the drain handler parsed incoming characters into the same shared line buffer that the command parser was using. Sending an SMS in response to a command therefore overwrote the command still being parsed. The first command always worked; nothing after it did. The fix was to stop the send path from re-entering the parse path.

**Incoming messages were invisible.** SIM800L pushes received messages directly as `+CMT:`. SIM900 firmware instead stores the message and sends `+CMTI: SM,n`, a notice that message *n* is waiting. Code written for one module silently receives nothing on the other, because it is waiting for a notification style the module never sends. Handling `+CMTI` means reading the stored message with `AT+CMGR` and then deleting it.

**The board kept resetting to the welcome screen.** Adding a couple of local buffers of 160 and 64 bytes inside a nested call chain overflowed the Nano's 2KB SRAM. There is no fault, no message, just a silent restart that looks like a power problem. On a board this small, buffer size is an architectural decision rather than a detail.

---

## Getting started

### Prerequisites

- Arduino IDE with the Nano board selected
- Libraries: `LiquidCrystal`, `SoftwareSerial`, `EEPROM`
- An active SIM with credit, registered and PIN disabled

### Configure

```c
const float TANK_HEIGHT_CM = 30.0;   // Sensor face to tank floor
const int   LOW_THRESHOLD  = 20;     // Percent, pump on
const int   HIGH_THRESHOLD = 99;     // Percent, pump off
const char  DEFAULT_PHONE[] = "+234XXXXXXXXXX";
```

`TANK_HEIGHT_CM` must be measured from the sensor face down to the empty tank floor, not the tank's own height. Getting this wrong scales every reading.

The default number is used only on first boot. After the first `SETNUM`, the EEPROM value always wins.

### Upload

Select Arduino Nano, choose the correct processor variant, and upload. Watch the LCD through the boot sequence: it reports GSM initialisation and readiness, so a module that is not registering on the network shows up immediately.

---

## Project status

Working hardware, built and demonstrated.

Not yet implemented:

- Multiple alert recipients
- Dry-run protection on the pump
- Level history or usage logging
- Configurable thresholds over SMS

---
