---
title: "Building an Industrial Water Pump Controller with Arduino Opta and Home Assistant"
date: 2026-03-28T10:00:00-07:00
author: "Odis Harkins"
authorLink: "/aboutme/"
tags: ["Arduino", "Home Automation", "MQTT", "Home Assistant", "PLC"]
categories: ["Home Automation", "Development"]
draft: false
---

When our well pump started acting up, I figured it was a perfect excuse to build something more than a simple on/off switch. I wanted real-time monitoring, intelligent fault protection, and full Home Assistant integration — so I built an industrial-grade controller using the **Arduino Opta**.

## Why the Arduino Opta?

The [Arduino Opta](https://www.arduino.cc/en/hardware/opta) sits in a unique space: it's a proper industrial PLC form factor with DIN rail mounting, relay outputs, and built-in Ethernet, but it's fully programmable with the Arduino ecosystem. That last part is what sold me — I didn't want to learn ladder logic, I wanted to write C++ and have it work.

A key caveat: the Opta must run on the **Mbed OS core**, not the newer Zephyr core. The Ethernet library has compatibility issues with Zephyr, so stick with Mbed if you want networking.

## System Architecture

The core design principle is **network independence**. The pump has to keep running even if the MQTT broker goes down, the network drops, or Home Assistant restarts. All the critical control logic — sensor reads, fault detection, relay switching — runs locally. MQTT is purely a reporting and remote-command layer layered on top.

```
[Pressure Sensor] ──┐
[Current Sensor]  ──┤
[Depth Sensor 1]  ──┤──► Arduino Opta ──► [Pump Relay]
[Depth Sensor 2]  ──┤              │──► [Fault Indicator Relay]
[Mode Selector]   ──┤              │──► [Status LEDs x4]
[Reset Button]    ──┘              │
                                   └──► Ethernet ──► MQTT ──► Home Assistant
```

## Three Operating Modes

The controller has a physical 3-position mode selector switch on the panel:

**OFF** — Pump is disabled, all fault checking suspended. Use this for maintenance or winterizing. Faults auto-clear in this mode.

**ON** — Manual/priming mode. The pump runs continuously and low-pressure faults are ignored. This is invaluable when you're filling a dry system and pressure will naturally be low at startup. High-pressure cutout still functions as a hard safety limit.

**AUTO** — Full protection mode. All fault systems are active, and the pump starts automatically when selected (assuming no active faults).

## Sensor Inputs and Fault Protection

The system monitors four analog inputs:

| Input | Sensor | Range |
|---|---|---|
| A0 | Pressure | 0–87 PSI (0–10V) |
| A1 | Current | 0–20A (0–10V) |
| A3 | Tank depth 1 | 0–2m (4–20mA) |
| A4 | Tank depth 2 | 0–2m (4–20mA) |

The digital inputs (A5–A7) handle the reset button and mode selector. **One critical hardware note:** the Opta's analog input pins do not support internal pull-ups. You must wire **external 10K pull-up resistors** on these pins or you'll get floating inputs and erratic behavior. Learned that the hard way.

### Fault Logic

Rather than tripping immediately on a threshold crossing, every fault uses a **sustained violation window** to avoid false trips during normal transients:

- **Low pressure** (<30 PSI, AUTO mode only) — triggers after sustained low reading
- **High pressure** (>60 PSI) — active in all running modes; in ON mode causes an auto-restart rather than a permanent latch
- **Overcurrent** (>10A) — requires 6 consecutive seconds above threshold
- **Undercurrent** (≤3A while pump commanded on) — also requires 6 seconds; catches cavitation or a broken shaft

After any fault clears via the reset button, there's a 6-second grace period before fault checking re-enables. This prevents the controller from immediately re-latching if the underlying condition takes a moment to resolve.

The four front-panel LEDs map directly to fault types, so a glance at the panel tells you exactly what's wrong without needing a phone or laptop.

## Home Assistant Integration via MQTT

The controller publishes sensor data every second and uses **MQTT auto-discovery**, so Home Assistant picks up all the entities automatically with no manual configuration:

**Published topics:**
- `waterpump/pressure` — live PSI reading
- `waterpump/amps` — live current draw
- `waterpump/depth1` / `waterpump/depth2` — tank levels
- `waterpump/status` — pump running state
- `waterpump/fault` — active fault description
- `waterpump/mode` — current operating mode
- `waterpump/availability` — online/offline heartbeat

**Subscribed topics:**
- `waterpump/cmd/reset` — remote fault reset
- `waterpump/cmd/mode` — remote mode change

The availability topic is particularly useful — if the controller loses power or crashes, Home Assistant marks all entities as unavailable rather than showing stale data.

## What I'd Do Differently

A few lessons from the build:

1. **Calibrate sensors on the bench first.** The 4–20mA depth sensors needed trimming before install. Much easier to do at a desk than crouched in a pump house.

2. **Use shielded cable for the analog runs.** The pump motor created enough electrical noise to cause occasional jitter on the current sensor before I switched cables.

3. **Add a local display.** The serial debug output at 115200 baud is great during development, but I'm planning to add a small OLED to show live pressure and current without needing a laptop nearby.

## The Code

All source code is available in my [home-automation repository](https://github.com/oharkins/home-automation/tree/main/Arduino_Opta/Water_Pump_Controller). The README includes full wiring diagrams, threshold configuration, and troubleshooting steps.

The non-blocking architecture means the main loop runs fast enough to stay responsive on all inputs while still handling Ethernet and MQTT without the timing issues you'd get from `delay()`-based code.

If you're managing a well pump, irrigation system, or any other critical water infrastructure and want something smarter than a pressure switch, the Opta is worth a look. The PLC form factor means it lives happily in an electrical panel, and the Arduino ecosystem means you're not locked into proprietary tooling.
