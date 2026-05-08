---
layout: page
title: "Opto-Isolated PWM Motor Driver"
permalink: /projects/icarus-fc/optoisolator-driver
noindex: true
sitemap: false
---

Discrete opto-isolated low-side driver for a small brushed-DC ducted fan — the first hardware stage of the [icarus-fc 1D hover stabilizer](https://github.com/fuzzygreenblurs/icarus-fc), built on a solderboard and validated open-loop on the bench before MCU integration.

<style>
.demo-row {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  gap: 1.5rem;
  flex-wrap: wrap;
  margin: 2rem auto 0.5rem;
}
.demo-card {
  position: relative;
  display: inline-block;
  width: 360px;
  max-width: 100%;
  text-decoration: none;
}
.demo-card img {
  width: 100%;
  display: block;
  border-radius: 8px;
  box-shadow: 0 4px 14px rgba(0,0,0,0.18);
  transition: transform 0.15s ease;
}
.demo-card:hover img { transform: scale(1.02); }
.demo-card .play-overlay {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 68px;
  height: 48px;
  background: rgba(0,0,0,0.78);
  border-radius: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.15s ease;
  pointer-events: none;
}
.demo-card:hover .play-overlay { background: #ff0000; }
.demo-card .play-overlay::before {
  content: "";
  display: block;
  border-style: solid;
  border-width: 11px 0 11px 18px;
  border-color: transparent transparent transparent #fff;
  margin-left: 4px;
}
.demo-card .yt-badge {
  position: absolute;
  bottom: 0.55rem;
  right: 0.55rem;
  background: rgba(0,0,0,0.78);
  color: #fff;
  font-size: 0.72rem;
  font-weight: 600;
  letter-spacing: 0.04em;
  padding: 0.18rem 0.42rem;
  border-radius: 3px;
  pointer-events: none;
}
.demo-caption {
  text-align: center;
  font-size: 0.92rem;
  opacity: 0.7;
  margin-top: 0.25rem;
  margin-bottom: 2.5rem;
  font-style: italic;
}
</style>

<div class="demo-row" markdown="0">
  <a class="demo-card" href="https://youtube.com/shorts/NCCM40ZPoXY?feature=share" target="_blank" rel="noopener" aria-label="Demo 1: driver running open-loop">
    <img src="https://img.youtube.com/vi/NCCM40ZPoXY/hqdefault.jpg" alt="Demo 1">
    <span class="play-overlay" aria-hidden="true"></span>
    <span class="yt-badge">YouTube</span>
  </a>
  <a class="demo-card" href="https://youtube.com/shorts/bAPke4g3cEE?feature=share" target="_blank" rel="noopener" aria-label="Demo 2: driver running open-loop">
    <img src="https://img.youtube.com/vi/bAPke4g3cEE/hqdefault.jpg" alt="Demo 2">
    <span class="play-overlay" aria-hidden="true"></span>
    <span class="yt-badge">YouTube</span>
  </a>
</div>

<p class="demo-caption">Driver running open-loop with the ducted fan — click to watch (YouTube Shorts)</p>

### Overview of circuit

![Motor control circuit schematic](/assets/images/optoisolator/motor_control_circuit.png)

The motor circuit must protect the microcontroller from the large *L · di/dt* voltage spikes that come off a PWM-driven DC motor. The **4N35 optoisolator** completely isolates the MCU from the motor. The **1N4001 snubber diode** provides a path to ground for reverse-polarity spikes coming off the motor, and the capacitor in parallel with the 1N4001 provides a path to ground for higher-frequency noise.

Some of the components in this circuit require some experimentation/trial and error. The resistor attached to the base of the 4N35 should be set for best fall-time, probably ~1 MΩ. The capacitor in parallel with the motor should be ceramic (electrolytics are too slow) and should start with a value ~0.1 µF. If there is too much spike noise on the analog input, this value can be increased.

Note that we're driving the gate of the power MOSFET at **12 V**, and the motor at **5–6 V**.

### Pinouts and PWM frequency

The pinouts for the 4N35 optoisolator and the IRLB8721 power MOSFET (TO-220AB package) are shown in the schematic above. Note that it is the **bandwidth of the 4N35 that constrains the PWM frequency** — the bandwidth for this device is low, so the design uses a PWM frequency of about **1 kHz**.

### Bench setup

A function generator stands in for the MCU's PWM output during open-loop validation; the oscilloscope probes the input PWM and the gate-drive node downstream of the optoisolator.

![Kuman dual-channel function generator and Hantek DSO2D15 oscilloscope on the bench](/assets/images/optoisolator/IMG_6961.JPG)

![Function generator configured for a 1 kHz square wave on CH2 (PWM stand-in for MCU)](/assets/images/optoisolator/IMG_6962.JPG)

![Variable bench supply at 12.0 V / 0.20 A driving the gate-drive side of the solderboard](/assets/images/optoisolator/IMG_6959.JPG)

![Solderboard build: 4N35 optoisolator, IRLB8721 N-MOSFET, 1N4001 snubber, 0.1 µF ceramic in parallel with motor](/assets/images/optoisolator/IMG_6965.JPG)

### Waveforms

![Scope capture: 1 kHz square wave on the output stage of the driver](/assets/images/optoisolator/IMG_6958.JPG)

### Source files

- Schematic and design notes: [`1d_stabilizer/notes/motor_control_circuit.png`](https://github.com/fuzzygreenblurs/icarus-fc/blob/main/1d_stabilizer/notes/motor_control_circuit.png)
- Bench-test photos: [`1d_stabilizer/results/open_loop_testing/`](https://github.com/fuzzygreenblurs/icarus-fc/tree/main/1d_stabilizer/results/open_loop_testing)
- Reference reading: [DC motor circuit layout](https://github.com/fuzzygreenblurs/icarus-fc/blob/main/1d_stabilizer/notes/dc_motor_circuit_layout.pdf), [PWM (Pico)](https://github.com/fuzzygreenblurs/icarus-fc/blob/main/1d_stabilizer/notes/PWM_pico.pdf), [brushed DC principles](https://github.com/fuzzygreenblurs/icarus-fc/blob/main/1d_stabilizer/notes/brushed_dc_principle.pdf)
