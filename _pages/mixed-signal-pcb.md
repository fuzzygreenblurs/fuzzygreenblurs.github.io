---
layout: page
title: "Mixed-Signal Hardware Design: Function Generator / Signal Analyzer"
permalink: /p/06b7b874f9bf/
noindex: true
sitemap: false
---

<style>
.msp-tabs nav.tab-nav {
  display: flex;
  flex-wrap: nowrap;
  gap: 0;
  border-bottom: 1px solid var(--border-color);
  margin: 0 0 2rem 0;
  position: sticky;
  top: 0;
  background: var(--body-bg);
  z-index: 10;
  padding: 0.5rem 0;
  overflow-x: auto;
  scrollbar-width: thin;
  -webkit-overflow-scrolling: touch;
}
.msp-tabs nav.tab-nav button {
  background: none;
  border: none;
  padding: 0.45rem 0.65rem;
  cursor: pointer;
  font: inherit;
  font-size: 0.85rem;
  white-space: nowrap;
  flex: 0 0 auto;
  color: var(--body-color);
  opacity: 0.55;
  border-bottom: 2px solid transparent;
  margin-bottom: -1px;
  transition: opacity 0.12s ease, border-color 0.12s ease;
}
.msp-tabs nav.tab-nav button:hover { opacity: 0.85; }
.msp-tabs nav.tab-nav button.active {
  opacity: 1;
  color: var(--heading-color);
  border-bottom-color: var(--heading-color);
  font-weight: 600;
}
.msp-tabs .tab-pane { display: none; }
.msp-tabs .tab-pane.active { display: block; }
.msp-tabs .tab-pane h2 { margin-top: 0.5rem; margin-bottom: 1.25rem; }
.msp-tabs .tab-pane h3 { margin-top: 2.75rem; margin-bottom: 0.85rem; }
.msp-tabs .tab-pane h4 { margin-top: 2rem; margin-bottom: 0.65rem; }
.msp-tabs .tab-pane h3 + p,
.msp-tabs .tab-pane h4 + p { margin-top: 0; }
.msp-tabs .tab-footer {
  margin-top: 3.5rem;
  padding-top: 1.25rem;
  border-top: 1px solid var(--border-color);
  display: flex;
  justify-content: space-between;
  font-size: 0.95rem;
}
.msp-tabs .tab-footer a { text-decoration: none; }
.msp-tabs .tab-footer span.spacer { flex: 1; }
.msp-callout {
  border-left: 3px solid var(--border-color);
  padding: 0.75rem 1rem;
  margin: 1.5rem 0;
  font-size: 0.92rem;
  opacity: 0.85;
}
.msp-appendix h3 { margin-top: 2rem; }
.msp-appendix ul li { margin-bottom: 0.35rem; }
.msp-appendix .tbd { opacity: 0.55; font-style: italic; }
</style>

<div class="msp-tabs" markdown="0">

<nav class="tab-nav" aria-label="Project sections">
  <button data-tab="intro">Introduction</button>
  <button data-tab="parts">Part Selection</button>
  <button data-tab="power">Power Supply</button>
  <button data-tab="mcu">MCU / USB / SWD</button>
  <button data-tab="adc">ADC / DAC</button>
  <button data-tab="appendix">Appendix</button>
</nav>

<section class="tab-pane" id="tab-intro" markdown="1">

## Introduction

<div class="msp-callout" markdown="1">
**Note:** This text is a personal writeup of design reasoning. The accompanying schematics, diagrams, and slide-style figures from the source course have been intentionally omitted from this public page pending permission and/or replacement with my own original captures.
</div>

### Attribution

**This project does not claim to be a novel design** and is instead meant to be a discussion in hardware design, following the principles detailed in the [Phil's Lab Mixed-Signal Hardware Design](https://www.phils-lab.net/courses) course. Credit is attributed to Phil Salmony for providing the project objective, system level constraints and best practices to design for real-world hardware.

However, the course neglects to provide fundamental theoretical reasoning for many of these specific practices. This document attempts to fill in those gaps, acting as a self-contained justification for the entire system design, and further supplementing the course material.

### Objective

This document details a first-principles approach to designing an AC/DC mixed-signal printed circuit board (PCB) to analyze an incoming and/or generate an outgoing single-channel, low-voltage, audio-band frequency signal. Roughly speaking, this system mimics a simple function generator and analyzer (similar to a PC-based oscilloscope). This is a generic hardware design, serving as a useful starting point for test and measurement instrumentation electronics.

This design employs commercial off-the-shelf components and focuses primarily on macro design considerations, abstracting away the low-level specifics of individual ICs in favor of system-level reasoning to achieve the stated objective.

### System Requirements

1. **Low voltage supply:**
   - Voltage: 5V
   - Current: < 500 mA
   - Max power: 2.5 W

   *A standard USB-based power supply can meet these requirements. This USB line can also be used as a data line to perform control and data streaming.*

2. **Low (audio) signal frequency band: 20 Hz - 20 kHz**
   - ADC: required to support signal analyzer
   - DAC: required to support signal generator
   - Anti-aliasing: > 40 kHz sampling rate capability to meet Nyquist Criterion
   - Resolution: 12-bit sample size to minimize quantization error sufficiently
   - Bandwidth requirement: 0.48 Mbit/sec

   *Low frequency signals are selected as an initial constraint, requiring a lower sampling rate and therefore keeping the overall data bandwidth requirements smaller and easier to design against. USB 2.0 Full Speed mode can support <12 Mbit/sec data rates, more than sufficient for this requirement. Many STM32 microcontrollers already support this protocol without added configuration.*

3. **Processor: interfaces with data converters (ADC/DAC) and USB host (PC)**
   - FPGAs are ideal for high-bandwidth, parallelism and DSP workloads
   - Microcontrollers:
     - sufficient for low bandwidth, non-DSP workloads
     - easier to flash and debug (SWD or JTAG interfaces)
     - serial UART-based debugging is unnecessary
     - can interface over USB without added configuration
   - Data converters interface with the processor over a faster serial interface:
     - I²C: too slow for this application
     - LVDS or massive parallel buses are also an option but overkill here
     - SPI is sufficiently fast for this application

   *The ARM Cortex-M0 microcontroller line (least powerful in the Cortex-M line) is sufficient for this application. It will interface with the host over USB and with the ADC/DAC data converters over SPI. A simple SWD interface is selected as a debugging interface.*

4. **Power supply: subdivided to support the analog and digital circuitry individually.** Both regulators step down the 5V main USB supply to a dedicated 3.3V supply.
   - Analog circuitry: supplied by a simple linear (LDO) regulator
     - Not very tolerant to noise: avoid switching components in circuitry
     - Typically requires a small current draw (~10 mA)
     - Regulator efficiency is therefore not an issue: use a linear regulator
   - Digital circuitry:
     - More tolerant to noise than analog circuitry
     - Requires higher current draw (~100 mA)
     - Supply using a more power-efficient switching regulator (ex. buck converter)
   - Filtering: the USB power rail is prone to noise (depending on cable, host, etc.)
     - Input voltage is nominally +5V but can drop to +4.5V
     - Must watch out for regulator dropout voltages
     - Must consider EMI/ESD and ESR/ESL effects

5. **Additional front-end (facing the physical environment) circuitry:**
   - **ADC:**
     - High input impedance: >1 MΩ (similar to oscilloscopes)
     - This avoids disturbing or drawing much current from the measured circuitry
     - ESD protection: manual handling of connectors, probing signals, etc.
     - RF filtering to prevent aliasing and spurious noise in the input signal
   - **DAC:**
     - Low output impedance: <50 Ω
     - ESD protection: manual handling of connectors, driving loads, etc.
     - Reversed layout of the ADC pipeline
   - **Converter options:** single-ended-to-differential, differential-to-single-ended.

6. **Other considerations:**
   - Mechanical: PCB dimension constraints (mounting holes, connector placement, encasing)
   - Connectors: SMA, BNC, GPIO, etc.
   - Peripherals: use LEDs for debugging, power-on indication, user friendliness
   - Timing: crystal frequencies
   - ESD protection and filtering for EMC (electromagnetic compatibility)
   - Biasing circuitry: most electronics are powered by a single power supply, and both the digital and analog circuitry rely on this

<div class="tab-footer">
  <span class="spacer"></span>
  <a href="#parts" data-tab-link="parts">Part Selection &rarr;</a>
</div>

</section>

<section class="tab-pane" id="tab-parts" markdown="1">

## Part Selection

### Low Dropout (LDO) Regulator: HT75XX family

1. Low dropout (difference between input and output voltage): ~25 mV
2. Can support ~100 mA output current (greater than the ~50 mA requirement)
3. Only requires small input/output smoothing capacitors
4. Small SOT package available (SOT-23 vs SOT-89): easy to solder and debug

![HT75XX family — Holtek datasheet excerpt](/assets/images/mixed-signal-pcb/image9.png)

### Switching Regulator (DC-DC): TLV62569P buck converter

1. Highly efficient (95%) supply for digital circuitry exclusively
2. Steps down 5V to 3.3V, sourcing up to 100-250 mA
3. Feedback resistors allow for closed-loop control of variable loads
   - At higher switching frequencies, surrounding filter components are smaller, yielding smaller form factors

![TLV62569P functional block — TI datasheet](/assets/images/mixed-signal-pcb/image56.png)

![TLV62569P typical application — TI datasheet](/assets/images/mixed-signal-pcb/image59.png)

### Supporting components for analog circuitry

1. Op-amps used for active filters (combined with passives and buffers)
   - Single supply, rail-to-rail low voltage (0 - 3.3V inclusive) supply
   - BJT vs MOSFET: a FET has higher input impedance and lower bias current — good for a high-impedance input stage
   - Bandwidth: how much gain you can get based on input frequency. We have low bandwidth requirements.
   - Stability and decoupling capacitors are considered later on
2. Passive components
   - Minimize Johnson noise caused by selected non-ideal resistors
   - Johnson noise is a function of temperature and resistance value (reduce R values, within reason)

### Moving into Schematic Design

Section the schematic into:

- Power
- MCU
- ADC
- DAC

Divvy the schematic into logical blocks for readability and modular design. Each logical block has its own section dedicated to schematic design in this document.

<div class="tab-footer">
  <a href="#intro" data-tab-link="intro">&larr; Introduction</a>
  <a href="#power" data-tab-link="power">Power Supply &rarr;</a>
</div>

</section>

<section class="tab-pane" id="tab-power" markdown="1">

## Schematic: Power Supply

The overall power-supply schematic is composed of:

1. A linear (LDO) regulator across the top
2. A switching (buck) regulator across the bottom
3. A bias generator on the right
4. **V_BUS**: the +5V source from the USB Type-C connector

### Input Filter: π-filter network (CLC)

1. The inductor has a DC resistance of 0.15 Ω.
   - Without this resistance, the filter becomes a resonant circuit (very sharp peak at a specific frequency). The added resistance damps this resonant peak.
   - Cutoff frequency of about 1.6 MHz, which is high enough for our application — we just want to damp out high-frequency noise. Transients drawn by ICs require high frequencies, which we don't want to damp too much.
2. It can also support a maximum DC current of 500 mA.
   - Usually drawing much less current (~150 mA)
   - Design for redundancy to allow for spiking

#### Aside: RLC Filters

- Placing a capacitor in parallel with the V_in source makes the filter "bi-directional" (as in the π-filter approach).
- This is a 2nd-order filter, with sharper rolloff than RC filters.
- DC (low-frequency component) loss relies mainly on R. Higher R values lead to more DC loss, since damping causes the resonance peak to widen.

A bode plot of this filter shows that it is indeed a low-pass filter for high-frequency noise — perfect for a power filter. For high-speed digital circuitry, perform the *minimum* amount of filtering possible (just enough to pass EMI testing) because transients drawn by ICs require high-frequency currents which we don't want to damp out too much.

Without the R component, we'd see resonance (~40 dB) at about 1.5 MHz. Avoid these peaks in power supplies.

### LDO Regulator Configuration

The configuration matches the manufacturer-suggested layout, but with larger capacitors. Capacitance has tolerance up to 20%, and applying a DC bias across capacitors causes the effective capacitance to drop further, so we over-spec. The datasheet suggests 10 µF; we use 22 µF instead.

We have +5V along V_in and +3.3V (annotated VA for "analog source") generated at the output. The resistor on the input has a power rating of at least 1/8 W. Together with the input capacitor, this forms a simple RC filter.

#### Aside: RC Filters

- RC filters can be used to create LP or HP filters
- We can cascade them to form a bandpass (within reason — cascading causes impedance and loading problems)
- The cutoff frequency ("half-power bandwidth") is when the gain of the filter goes to $$1/\sqrt{2}$$. This is true for both LP and HP.
- Lower frequencies are relatively unaffected
- RC filters have a much slower rolloff compared to RLC. Use them only when noise filter requirements are much less stringent.

Returning to the RC filter at the input of the LDO: the cutoff frequency is relatively low, but it doesn't matter because the LDO is a low-frequency analog supply.

**Schematic note: ALWAYS NAME EVERY SINGLE NET (annotations to wires/networks). The default names are not readable.**

There is another reason for the RC filter feeding the input of the LDO: the switching regulator below can generate noise that traverses backward through the overall circuit, into the LDO and input supplies. **Any switching noise generated will be attenuated by the RC filter as well.**

The switching noise is likely lower frequency, which is why the RC filter will attenuate it — the π-filter at the USB input only attenuates very high-frequency (MHz-level) noise.

### Buck Converter Circuitry

- The BC datasheet specifies the associated components.
- The output voltage can be calculated using the equation given in the datasheet, based on the selected resistor feedback network. This is used to self-regulate the buck converter to continue to provide the correct voltage for a changing load.
- A maximum output current of 250 mA is specified.

#### Specifying the BC components

Different ICs include some of these parts internally, but for the TLV62569P the components needed are:

1. **Input capacitor** on the input rail (could be multiple caps for larger capacitance)
2. **Output inductor**
3. **Output capacitor** (might need multiple)
4. **Feedback resistor network** (R101 / R102)
   - Sometimes includes compensation capacitors to alter the output frequency response
   - The output voltage is divided through this network and fed back to a feedback pin, allowing the IC to stabilize the output value at the desired voltage
   - Different R values yield different output voltages for the same input voltage
5. Some BC ICs require an external diode or FET, but this specific IC is simple. Always check the datasheet.

**Selecting the correct sized inductor is a very important part of buck converter design.** The minimum inductance is given by a standard formula in most switching/BC datasheets, depending on:

- Output voltage
- Input voltage
- Switching frequency of the regulator
- Inductor ripple current (some constant times the maximum expected load)

**Takeaways:**

- Larger switching frequency → smaller L → smaller inductor package (good)
- Larger relative input-to-output voltage → larger L (not good); here +5V → +3.3V, so not very large
- We typically aim for ~20-30% ripple current compared to max output current

For our specific application:

- For V_in − V_out, consider the largest difference (when V_in is highest at +5V)
- F_sw = 1.5 MHz
- I_max = 250 mA → ripple current should be 0.25 × I_max = 62.5 mA
- Thus L_min = 12 µH
- Choose a standard value: L = 15 µH (accounting for inductor 20% tolerance)
- Ensure specified max DC current and saturated current are much higher than expected/peak values
- Always use shielded inductors where possible to minimize radiation

We derived this value based on the extreme case of 250 mA. However, our general case will be significantly less, driving down the ripple current and therefore driving up L. Recall that the ripple current can be represented as a fraction of the actual current, given by $$K \cdot I_L$$.

Lighter loads require larger inductors. We therefore choose **68 µH**, because we are expecting really light loads.

Next, consider the datasheet for the BC IC (TLV62569) for input and output capacitor selection:

- **Input capacitor**: ≥ 4.7 µF. We use 22 µF wherever possible for BOM consolidation.
- **Output capacitor**: 22 µF is reasonable per the datasheet table.

![TLV62569 input capacitor table — TI datasheet](/assets/images/mixed-signal-pcb/image31.png)

![TLV62569 output capacitor table — TI datasheet](/assets/images/mixed-signal-pcb/image54.png)

Finally, select the feedback resistor network using the equation in the datasheet:

![TLV62569 feedback formula — TI datasheet](/assets/images/mixed-signal-pcb/image61.png)

These resistors will have some tolerance — select 1% tolerance to achieve values closest to expected theoretical output.

The datasheet also recommends adding a DNP (Do Not Place) capacitor to the output. In the PCB design, pads will be available to optionally add this capacitor later. This is done so that if the BC is not stable, the cap can be manually soldered on. **Add DNP parts freely.**

#### Power-On LED

It is a good idea to add a PWR ON LED for testing downstream (especially for early-stage boards). The LED indicates that some sort of power circuitry is working.

Selecting the current-limiting resistor:

- LED forward voltage depends on color and current at that moment
- Choose a suitable forward current I_f (≤ 2 mA) for debugging — 10–20 mA for brighter LEDs
- For 2 mA, the corresponding forward voltage is ~2.9 V; we use <1 mA so V_f ≈ 2.8 V
- Current-limiting resistor: $$R = (V_{dd} - V_f) / I_f = (3.3\text{V} - 2.8\text{V}) / 1\text{mA} = 500\,\Omega$$
- Select 1 kΩ to be safe (yields ~0.5 mA)

### Bias Generator

We are running the analog circuitry from +3.3 V_A to GND, not +3.3 V_A to −3.3 V_A. This is considered a single supply. Op-amps do not work well with single supplies — when using a single supply into an op-amp, we need to bias their input (or one of their inputs) with ½ the supply voltage.

This way, the AC signal is super-imposed (0 V is at 1.65 V) over this "shifted" voltage, so it can swing between −1.65 V and +1.65 V instead.

The equally-sized resistors evenly divide the input voltage to generate the REF +1.65 V value (which acts as virtual 0 V). C107 (22 µF) together with the two resistors forms an RC filter with a calculated cutoff of 15 Hz.

We then use a voltage-follower configuration of an op-amp to yield the bias output VCOM. This is a common configuration in the analog frontend.

#### Voltage Followers in Depth

**Ideal Op-Amp Rules:**

- Under feedback, the op-amp will tend to match its input voltages so $$V_+ = V_-$$
- No current flows into the op-amp (infinite input impedance)
- The gain should be lossless (zero output impedance)

**Running from a single supply:**

- The op-amp must be biased at $$V_{cc}/2$$ to allow for full input/output swing
- AC signals are coupled onto the bias via coupling capacitors

The op-amp follower configuration has its non-inverting input connected directly to VREF and its inverting input connected to the output. This:

- Provides stable bias voltage
- Heavily filters the bias voltage (reduces noise from power supply)
- Yields high input impedance and low output impedance

#### Decoupling / Bypass Capacitors

A small capacitor is connected between ground and the power rail — a decoupling/bypass capacitor, typically placed as close as possible to the relevant power pin of the IC. **Decoupling/bypass capacitors** isolate IC supply pins from rail noise; **coupling/blocking capacitors** pass AC while blocking DC bias between stages.

<div class="tab-footer">
  <a href="#parts" data-tab-link="parts">&larr; Part Selection</a>
  <a href="#mcu" data-tab-link="mcu">MCU / USB / SWD &rarr;</a>
</div>

</section>

<section class="tab-pane" id="tab-mcu" markdown="1">

## Schematic: Microcontroller, USB, SWD, ESD Protection

We use the STM32F1 microcontroller alongside USB, SWD, and ESD-protection circuitry. The schematic is composed of:

1. Microcontroller circuitry on the left
2. USB circuitry on the right
3. RGB LED and Serial Wire Debug (SWD) circuitry on the bottom

### MCU Layout

#### Power Pins (top)

- **VDD**: power for digital circuitry
  - Per VDD pin, connect a 100 nF capacitor very close to the respective pin (reduces ESR/ESL effects)
  - Also include 1 larger 10 µF bulk decoupling capacitor close to the IC
- **VDDA**: power for analog circuitry (for possible ADC/DACs onboard the MCU)
  - If not using the analog components on the MCU, it's fine to tie this to 3.3 V
  - For demo purposes the supply is filtered per the recommended configuration in the ST datasheet (decoupling cap + π-filter)
- **VBAT**: can be tied to a battery to keep RTC running
  - We are not using RTC, so we tie it to 3.3 V

#### GND Pins (bottom)

Use a common ground so all VSS and VSSA pins are tied together. Don't split this just because we have VSS vs VSSA.

#### Configuration Pins

- **NRST**: active-low reset pin — can trigger a HW reset by pulling LOW
  - Pull this pin up using a 40 kΩ (or higher) resistor
  - Can also use a pushbutton to reset manually
  - We connect it to the SWD header instead (discussed later)
- **BOOT0**: common on all STM32 MCUs
  - Tells the chip on startup whether to enable the internal bootloader
  - LOW: run flashed program
  - HIGH: listen for debug/programming signals on various ports (USB/UART/etc) and will not run the main program until pulled LOW and the system is reset
  - Can be connected to a DIP switch for toggling
  - Typically pulled down using a 10 kΩ resistor; for non-SWD programming, jumper to the resistor pad and pull HIGH
- **HSE_IN & HSE_OUT**: pins for the High-Speed External crystal
  - STM32 has its own internal high- and low-speed crystals (not accurate)
  - For UART/USB it is better to use an external crystal for timing reference
  - Circuitry discussed in ST Application Note AN2867 (oscillator design guide for STM32)

#### Crystal Oscillator Design

From AN2867, the MCU side contains a driver (inverter) with a feedback resistor. We need to place an external feed resistor, crystal, and load capacitors.

- C_s is stray capacitance caused by loads, PCB routing, and traces (cannot choose this)
- We choose the crystal, feed resistor and load capacitors

**Crystal:**

- Frequency f₀ = 16 MHz with low tolerance (PPM)
- Load capacitance C₀ in the pF range
- ESR ≈ 80 Ω
- 4-pin packages typical: 2 GND pins, 2 I/O / bidirectional

**Stray capacitance:** typically 3-5 pF, sometimes higher.

For our crystal: f₀ = 16 MHz, load capacitance = 10 pF, ESR = 80 Ω. The added load capacitors work out to ~10–14 pF each.

Next, the **feed resistor R_ext** is specified. The feed resistor:

- Forms a low-pass filter with the load capacitors
- Limits harmonics — placing the cutoff above the fundamental frequency limits distortion
- Reduces the inverter drive strength so the crystal isn't overdriven
- Often is not even necessary — place a 0 Ω (DNP) resistor instead, then swap in a real resistor if timing issues arise

In our schematic, R201 = 0 Ω and the load capacitors C208/C209 = 10 pF.

#### Pinout Selection

This circuitry is sufficient to power the chip. Next, choose the pinout for the connected devices:

- **SPI1** (right): for the external DAC
- **SPI2** (left): for the external ADC
- **GPIOs**: DAC_NCLR, DAC_NLDAC
- **TIM4**: timer channels for the RGB LED (PWM with varying duty cycle for brightness)
- **USB_D**: differential USB lines
- **SWD pins**: SWDIO, SWCLK and SWO

### STM32CubeIDE: Pin Configuration

STM32CubeIDE is used for pinout planning and to expose the alternate functions available on each pin. This is convenient for PCB routing — pin assignments may shift over the design lifecycle based on routing convenience.

> File → New → STM32 project → MCU/MPU Selector → STM32F103CB

For SWD we enable Trace Asynchronous SW under the **SYS** tab and select 3 pins: SWDIO, SWCLK and SWO (optional, useful for live variable monitoring).

Next, set up the HSE via the **RCC** (clock) tab.

For RGB LED PWM, configure any free timer (TIM4 chosen here) for 3 PWM channels (R, G & B).

Enable SPI1 and SPI2 for the data converters. The STM32 acts as master:

- **SPI1**: ADC (Receive Only Master)
- **SPI2**: DAC (Transmit Only Master)

The Hardware NSS Output Signal option provides chip-select handling.

For USB, this STM32 has an integrated full-speed (12 Mbps) physical layer, so the associated pins simply need to be enabled in the USB OTG peripheral configuration.

This concludes pinout planning. Future configuration includes USB middleware selection and clock configuration (via the clock tree diagram, not essential for PCB design).

These pinouts are then transcribed to KiCad. Other components added include:

- Pull-up resistors for the SPI HW chip-select lines (in case the lines aren't driven by GPIO)
- USB differential pair lines named USB_D− and USB_D+ — KiCad uses this naming to recognize a differential pair, used later when routing as a controlled-impedance pair

See ST App Note AN4879 for USB hardware and PCB guidelines for STM MCUs. Per Table 3 (and footnote 2 for our MCU), the USB_DP pin must be pulled up to 3.3 V using a 1.5 kΩ resistor:

![ST AN4879 — USB pull-up table](/assets/images/mixed-signal-pcb/image49.png)

![ST AN4879 — pull-up footnote for STM32F103](/assets/images/mixed-signal-pcb/image1.png)

### RGB LED Layout

Following the same approach as the power-supply current-limiting resistor, we use 1 kΩ resistors per PWM channel. Each color has a different forward voltage; this doesn't matter much because PWM duty cycles can be tuned for desired brightness.

A decoupling capacitor is used because of the switching nature of duty cycles — good practice for any switching component.

### USB Connector Layout

The USB rail carries both power and data. The Type-C connector has many more pins than Type-A/B.

The USB connector circuitry is divided into 3 subsections: **Type-C connector**, **ESD protection**, **common-mode choke**.

See ST AN4879 for pull-up resistor guidance and per-MCU USB capabilities. The 1.5 kΩ pull-up on USB_D+ was discussed earlier; that pin can also be toggled via GPIO to trigger a reset, which helps when the device gets stuck during debugging.

1. **The USB supply is likely very noisy** (long cable, EM/ES pickup, possibly noisy host)
   - Use an RLC π-filter as in the power supply
2. **Use low-capacitance ESD protection** due to USB's high bandwidth
   - Place ESD protection as close to the connector as possible
   - **USBLC6-2SC6**: common ESD protection IC for USB 2.0
   - Add a small decoupling cap at the power pin (protects VBUS and data pins)
3. **Data line filtering**: optional for USB 2.0
   - We use a common-mode choke
   - USB is not entirely differential, so a common-mode choke is not always optimal — it improves EMC but can degrade signal integrity

USB Type-C requires specific considerations — see App Note TA0357.

Many Type-C connectors can deliver up to 100 W (20 A @ 5 V), which is overkill for our application. Add two separate 5 kΩ pull-down resistors on CC1 and CC2 (configuration channel) to signal to the host that a device is connected and to provide up to 500 mA @ +5 V.

When choosing the common-mode choke, ensure it is rated to the desired frequency — USB 2.0 is fast, so size it correctly to avoid impeding speed. The common-mode choke also has differential impedance: ensure it matches the bus's characteristic impedance. For USB, route traces as 90 Ω impedance pairs (discussed in the PCB design step). Note that USB is bidirectional, so the signal path works both ways.

### SWD Header

The standard SWD 10-pin header:

![SWD 10-pin connector pinout](/assets/images/mixed-signal-pcb/image50.png)
![SWD 10-pin connector outline](/assets/images/mixed-signal-pcb/image65.png)

- It really only needs 2 lines: CLK and DIO
- SWO and NRST are not strictly required
- SWO is included for variable tracking/plotting
- NRST allows a hard reset of the MCU
  - A 100 nF capacitor (C212) with R213 forms a low-pass RC filter to prevent spurious resets (recommended by ST)
- Pins 4 and 7 are unconnected
- Bidirectional **TVS (Transient Voltage Suppression) diodes** sit on the SWD lines as ESD protection
  - Placed close to the connector so human handling or built-up static can be dispersed safely
- Current-limiting resistors protect against shorting SWDIO to GND while it is driven HIGH (3.3 V)

#### What is SWD?

SWD is an in-circuit debug and programming method similar to JTAG:

> Host PC → USB cable → ST-Link (V2/V3) → ribbon adapter cable → SWD 10-pin header → MCU

It only requires 2 data lines (DIO, CLK) plus GND, and enables monitoring variables, setting breakpoints, flashing code, etc. **This is our preferred option.**

#### TVS Diodes and ESD Protection

We select the **PESD3V3L1BA** TVS diode:

- Bidirectional diode
- Max peak pulse power: 500 W
- Clamping voltage: 26 V (max voltage during over-voltage events)
- ESD protection: > 23 kV (per various ESD models)
- Small SMD packaging (minimizes capacitance)
- **Reverse stand-off voltage** (working voltage): 3.3 V — what the diode is nominally rated for
- Fairly low capacitance: 100 pF

For high-speed buses we want low capacitance — higher capacitance increases rise/fall times and limits bus speed.

![PESD3V3L1BA — Nexperia datasheet excerpt](/assets/images/mixed-signal-pcb/image45.png)

![PESD3V3L1BA — package outline](/assets/images/mixed-signal-pcb/image30.png)

TVS diodes are a simple way to account for ESD effects.

<div class="tab-footer">
  <a href="#power" data-tab-link="power">&larr; Power Supply</a>
  <a href="#adc" data-tab-link="adc">ADC / DAC &rarr;</a>
</div>

</section>

<section class="tab-pane" id="tab-adc" markdown="1">

## Schematic: ADC, DAC and Analog Circuitry

### ADC

ADC: TI's **ADC141S626**

- 14-bit converter
- 50–250 kSamples/sec
- Differential input

We follow the ADC datasheet to lay out the data acquisition circuitry:

![ADC141S626 typical application — TI datasheet](/assets/images/mixed-signal-pcb/image15.png)

In our case we use different supply voltages (the rail along the top) and a different voltage reference.

**Important connections:**

- **Power supply:**
  - Sources +3.3 V_A and +3V3 (digital) to the VA and VIO pins respectively
  - Both tied to a common ground (for PCB routing reasons)
- **Decoupling**: 100 nF close to pins, 10 µF bulk capacitors (always consider BOM consolidation)
- **Voltage reference**
- **Differential input** to the ADC
- **Digital output** D_OUT, feeds into the MCU over SPI
- The CLK signal is routed through a series resistor (DNP, ~22 Ω) to reduce EMI/ringing — can also be applied to data lines for similar effect

The ADC requires a fairly precise voltage reference at the VREF pin. A power supply regulator could supply this line, but the regulator voltage varies with load, is not particularly stable, and would be far noisier than a dedicated analog voltage reference. **A dedicated analog voltage reference drastically improves ADC measurement accuracy.**

The voltage reference IC used is the **REF3033AIDBZT** (REF30XX family):

- 50 ppm/°C
- Low current draw (50 µA)
- Easy SOT-23-3 package
- Outputs 3.33 V (labeled ADC_VREF), indicated by the part number (3033)
- Requires 100 nF decoupling and 10 µF bulk decoupling capacitors at the output

Laying out the input circuitry:

- The input 5 V rail is very noisy
- This is also a steady, slow analog reference
- Perform additional heavy filtering using a π-filter
  - 10 µF caps and series resistor: very low cutoff of ~319 Hz
  - The IC takes up to 45 mA input current → spec the resistor power rating accordingly

### ADC Frontend

The ADC frontend feeds the ADC. Its role is to provide an impedance conversion and an anti-alias filter feeding into the ADC. The ADC has a low impedance (~2 kΩ) and we want a much higher input impedance (e.g. an oscilloscope is typically ~1 MΩ).

Considering the left side of the circuit (starting from the BNC connector):

- BNC with ESD protection (PESD component, same as the digital section)
- Input voltage must be in the range of ½ supply voltage (3.3 V), so −1.65 V to +1.65 V
  - More sophisticated designs (oscilloscopes) handle much wider input ranges via variable gain selection / attenuation
  - Our simple design is limited to supply rails

*The DAC schematic, full PCB layout, routing and fabrication notes are in progress and will be added to this page.*

<div class="tab-footer">
  <a href="#mcu" data-tab-link="mcu">&larr; MCU / USB / SWD</a>
  <a href="#appendix" data-tab-link="appendix">Appendix &rarr;</a>
</div>

</section>

<section class="tab-pane msp-appendix" id="tab-appendix" markdown="1">

## Appendix: Theory & References

This section collects deeper-theory writeups and external references used to motivate the design choices throughout the project. Entries marked *TBD* are stubs to be filled in as the supporting writeups are authored.

### Filtering & Signal Conditioning

- RC filters — derivation of LP/HP cutoff, cascade impedance/loading effects. <span class="tbd">TBD</span>
- RLC filters — damped resonance, π-filter bidirectionality, 2nd-order rolloff. <span class="tbd">TBD</span>
- Anti-aliasing & the Nyquist criterion. <span class="tbd">TBD</span>
- [RLC online calculator](https://www.daycounter.com/Calculators/RLC-Filter-Calculator.phtml)

### Power Electronics

- LDO regulators — dropout voltage, headroom, stability. <span class="tbd">TBD</span>
- Buck converter design — inductor sizing, ripple current, feedback compensation. <span class="tbd">TBD</span>
- DC bias derating of MLCC capacitors. <span class="tbd">TBD</span>
- Decoupling vs bulk capacitance — placement, ESR/ESL. <span class="tbd">TBD</span>

### Op-Amps & Analog

- Single-supply biasing for op-amps. <span class="tbd">TBD</span>
- Voltage follower — input/output impedance derivation. <span class="tbd">TBD</span>
- Johnson noise vs resistor selection. <span class="tbd">TBD</span>
- Decoupling/bypass vs coupling/blocking capacitors. <span class="tbd">TBD</span>

### Digital & Communication

- SPI — clocking, NSS, master/slave handshaking. <span class="tbd">TBD</span>
- USB 2.0 — full-speed differential signaling, pull-ups, host enumeration. <span class="tbd">TBD</span>
- SWD vs JTAG — pin reduction and protocol overview. <span class="tbd">TBD</span>
- Crystal oscillator design — load capacitance, drive strength, R_ext. <span class="tbd">TBD</span>

### PCB Design Topics

- ESD protection (TVS diodes, clamping voltage, IEC 61000-4-2 model). <span class="tbd">TBD</span>
- EMI/EMC filtering at connector boundaries. <span class="tbd">TBD</span>
- Controlled-impedance differential pair routing (90 Ω for USB). <span class="tbd">TBD</span>
- Common-mode chokes — operation and trade-offs. <span class="tbd">TBD</span>

### Standards & Application Notes

- [ST AN2867 — Oscillator design guide for STM32](https://www.st.com/resource/en/application_note/an2867-oscillator-design-guide-for-stm8afals-stm32-mcus-and-mpus-stmicroelectronics.pdf)
- [ST AN4879 — USB hardware and PCB guidelines for STM32 MCUs](https://www.st.com/resource/en/application_note/an4879-usb-hardware-and-pcb-guidelines-using-stm32-mcus-stmicroelectronics.pdf)
- ST TA0357 — USB Type-C considerations. <span class="tbd">link TBD</span>
- [Phil's Lab — Mixed-Signal Hardware Design Course](https://www.phils-lab.net/courses)

### Datasheets

- [HT75xx LDO regulator family](https://www.holtek.com/productdetail/-/vg/ht75xx-1)
- [TLV62569P buck converter (TI)](https://www.ti.com/product/TLV62569)
- [REF3033 voltage reference (TI)](https://www.ti.com/product/REF3033)
- [ADC141S626 14-bit ADC (TI)](https://www.ti.com/product/ADC141S626)
- USBLC6-2SC6 ESD protection (ST). <span class="tbd">link TBD</span>
- PESD3V3L1BA TVS diode (Nexperia). <span class="tbd">link TBD</span>
- STM32F103CB MCU (ST). <span class="tbd">link TBD</span>

<div class="tab-footer">
  <a href="#adc" data-tab-link="adc">&larr; ADC / DAC</a>
  <a href="#intro" data-tab-link="intro">Back to Introduction</a>
</div>

</section>

</div>

<script>
(function () {
  var root = document.querySelector('.msp-tabs');
  if (!root) return;
  var buttons = root.querySelectorAll('nav.tab-nav button');
  var panes = root.querySelectorAll('.tab-pane');
  var validTabs = Array.prototype.map.call(buttons, function (b) { return b.dataset.tab; });

  function activate(name, opts) {
    if (validTabs.indexOf(name) === -1) name = validTabs[0];
    buttons.forEach(function (b) { b.classList.toggle('active', b.dataset.tab === name); });
    panes.forEach(function (p) { p.classList.toggle('active', p.id === 'tab-' + name); });
    if (location.hash !== '#' + name) {
      history.replaceState(null, '', '#' + name);
    }
    if (!opts || !opts.preserveScroll) {
      window.scrollTo(0, 0);
    }
  }

  buttons.forEach(function (b) {
    b.addEventListener('click', function () { activate(b.dataset.tab); });
  });
  root.querySelectorAll('[data-tab-link]').forEach(function (a) {
    a.addEventListener('click', function (e) {
      e.preventDefault();
      activate(a.dataset.tabLink);
    });
  });
  window.addEventListener('hashchange', function () {
    activate((location.hash || '#intro').slice(1), { preserveScroll: true });
  });

  var initial = (location.hash || '#intro').slice(1);
  activate(initial, { preserveScroll: true });
})();
</script>
