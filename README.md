# Wearable PPG Sensor for Heart Rate Monitoring

**Electronic Workshop – II : Project 2**

A fully analog photoplethysmography (PPG) front-end that measures heart rate optically. It takes a raw photodiode signal in the microvolt range, runs it through five analog signal-conditioning stages, and produces a clean digital pulse that an Arduino Uno converts into BPM.

**Himangi Sahoo** (2024112020) · **Gauri Krishnan** (2024102073) · IIIT Hyderabad

---

## Table of Contents

- [Overview](#overview)
- [LED–Photodiode Circuit](#ledphotodiode-circuit)
- [Stage 1: Transimpedance Amplifier](#stage-1-transimpedance-amplifier)
- [Stage 2: Cascaded Sallen-Key Bandpass Filter](#stage-2-cascaded-sallen-key-bandpass-filter)
  - [High Pass Filter](#high-pass-filter)
  - [Low Pass Filter](#low-pass-filter)
- [Stage 3: Notch Filter](#stage-3-notch-filter)
- [Stage 4: Gain Amplifier](#stage-4-gain-amplifier)
- [Stage 5: Comparator](#stage-5-comparator)
- [Microcontroller](#microcontroller)
- [Final Circuit](#final-circuit)
- [PPG Waveform](#ppg-waveform)
- [Tools Used](#tools-used)

---

## Overview

Photoplethysmography is a simple, non-invasive optical technique for monitoring heart rate, respiratory rate, and oxygen saturation. An IR/green light source shines into the skin, and a photodetector measures the intensity of light reflected or transmitted back. This project builds a PPG circuit using an IR LED and photodiode in **reflective mode** to measure changing blood volume in the arteries, then processes that signal through five analog stages and a microcontroller to display heart rate in BPM.

The raw photodiode signal is extremely weak and noisy, so it passes through:

1. **Transimpedance Amplifier (TIA)**. Converts photocurrent to voltage.
2. **Sallen-Key Bandpass Filter** (cascaded high-pass + low-pass). Isolates the 0.5–5 Hz heartbeat band.
3. **Notch Filter**. Removes 50 Hz power-line interference.
4. **Gain Amplifier**. Boosts the signal to a usable amplitude.
5. **Comparator**. Converts the analog waveform to a clean digital pulse for the microcontroller.

---

## LED–Photodiode Circuit

The IR LED and photodiode sit side by side; the LED shines IR light through the finger tissue to the blood arteries, and as arterial blood volume changes with systolic/diastolic pressure, the amount of light reflected back to the photodiode changes correspondingly (per the Beer-Lambert law of light absorption). The photodiode converts the received light into a photocurrent, which is **inversely proportional to blood volume**: more blood means more absorption, means less light reaching the photodiode, means lower photocurrent. This current is fed into the TIA.

## Stage 1: Transimpedance Amplifier

The TIA converts the photodiode's current into a voltage signal using an op-amp in a negative-feedback current-to-voltage configuration, biased around a 2.5 V reference (`V_ref`) so the output sits within the 0–5 V range needed by the downstream microcontroller.

By virtual short (`V+ = V- = V_ref`), the photodiode sees a constant DC voltage at its node, giving the TIA low input impedance and good impedance matching. The output follows:

```
V_out = V_ref + I_in · R
```

With expected photocurrent around 0.1–1 µA and a target AC swing of ~50 mV, the feedback resistor works out to `R ≈ 500 kΩ`; a standard **560 kΩ** resistor was used. A parallel **10 pF** capacitor stabilizes the feedback loop and adds a low-pass corner around 28 kHz, far above the 0.5–5 Hz heartbeat band, so it only suppresses high-frequency noise.

In LTSpice, using a real PPG dataset as input, the TIA output landed in the µV range; on the physical hardware, the output amplitude was larger but still too weak to see the pulsatile component clearly on the oscilloscope, motivating the filter and gain stages that follow.

## Stage 2: Cascaded Sallen-Key Bandpass Filter

### Why Sallen-Key?

The Sallen-Key topology realizes a second-order active filter (high-pass, low-pass, band-pass, or notch) from an op-amp plus four impedance elements. Being second-order gives a steeper 40 dB/decade roll-off than a first-order filter's 20 dB/decade, which is useful for tightly bounding the narrow 0.5–5 Hz PPG band. Rather than a single band-pass stage, a high-pass and low-pass Sallen-Key filter were cascaded separately, so the two cutoff frequencies can be tuned and analyzed independently. The combined network is fourth-order overall, giving a 40 dB/decade roll-off at *both* the lower and upper cutoffs.

### High Pass Filter

Capacitors sit in the forward signal path, blocking DC and slow-moving components while passing higher frequencies. The cutoff frequency is:

```
f_L = 1 / (2π√(R1·R2·C1·C2))
```

**Implemented values:** `C11 = C10 = 10 µF`, `R23 = R9 = 50 kΩ` → **f_L ≈ 0.318 Hz**, chosen to sit just below the 0.5 Hz lower edge of the PPG band without excessive attenuation there.

LTSpice and hardware testing confirmed a clean signal at 5 Hz (passband) with visibly attenuated amplitude at 0.1 Hz (stopband).

### Low Pass Filter

Resistors sit in the forward path with capacitors shunting to the reference node, passing low frequencies and attenuating high ones. With non-inverting gain `K = 1 + R15/R16`, the cutoff frequency is:

```
f_H = 1 / (2π√(R1·R2·C1·C2))
```

**Implemented values:** `R13 = R12 = 20 kΩ`, `C7 = C8 = 1 µF`, `R15 = 5.6 kΩ`, `R16 = 10 kΩ` → `K = 1.56` and **f_H ≈ 7.96 Hz**, intentionally set a bit above the 5 Hz upper band edge so the PPG waveform isn't attenuated near its top edge while still cutting higher-frequency noise.

Testing at 3 Hz and 5 Hz showed the signal passing through, while 0.1 Hz and 20 Hz were both heavily attenuated, confirming the bandpass behavior once cascaded with the high-pass stage.

## Stage 3: Notch Filter

A **Twin-T notch filter** suppresses the ~50 Hz power-line interference that would otherwise corrupt the PPG waveform. It's built from two parallel T-networks (one low-pass, one high-pass) whose outputs are equal in magnitude but opposite in phase at the notch frequency, causing them to cancel. Op-amp buffer stages isolate the passive Twin-T network to sharpen and stabilize the notch.

For a balanced Twin-T design (`R, R, R/2` and `C, C, 2C`), the notch frequency is:

```
f_0 = 1 / (2π·R·C)
```

**Implemented values:** `R7 = R8 = 9.65 kΩ`, `C2 = C3 = 330 nF`, `R4 = 4.7 kΩ ≈ R/2`, `C4 = 660 nF = 2C` → **f_0 ≈ 50 Hz**. Measured Bode plots confirmed the notch centered at 50.1 Hz, effectively rejecting the power-line noise.

## Stage 4: Gain Amplifier

The notch filter output feeds a non-inverting gain stage that boosts the PPG signal enough to be observed and processed downstream:

```
V_out = (1 + R2/R1)(V_in - V_ref) + V_ref
A_v = 1 + R2/R1
```

**Implemented values:** `R2 = 560 kΩ`, `R1 = 33 kΩ` → **A_v ≈ 17** (theoretical). Measured on hardware with a real PPG input, the practical gain came out to **≈ 15**, closely matching theory.

## Stage 5: Comparator

The comparator converts the amplified analog PPG waveform into a clean rail-to-rail digital pulse, by comparing it against a reference voltage:

```
V_PPG > V_ref  ⇒  V_out ≈ V_CC
V_PPG < V_ref  ⇒  V_out ≈ V_EE
```

With `V_CC = 5V`, `V_EE = 0V`, `V_ref = 2.5V`, this acts as a simple analog-to-digital conversion, producing a square wave whose edges the microcontroller can easily detect. In both LTSpice simulation and hardware testing with a live PPG signal, the comparator output was a clean periodic pulse train at ~1.25 Hz, consistent with a resting heart rate.

## Microcontroller

An **Arduino Uno** takes the comparator's digital pulse as input, measures the time period `ΔT` between pulses, and computes heart rate as:

```
BPM = 60 / ΔT
```

For a 1.25 Hz pulse train, this gives `BPM = 60 × 1.25 = 75`, within the typical 60–100 BPM resting range. The result is printed live to the Arduino Serial Monitor.

## Final Circuit

All five stages were assembled into a single end-to-end circuit and verified both in LTSpice (using a recorded PPG dataset as input; note the resulting waveform is naturally non-uniform, reflecting the real dataset rather than a circuit error) and on physical hardware, built on a breadboard with an Arduino Uno for BPM calculation.

Averaging over 4 cycles on the oscilloscope gave a measured heart rate of **89 BPM**, matching the value the Arduino computed and printed to the Serial Monitor.

## PPG Waveform

The PPG waveform has a large DC component (from constant absorption by tissue, venous blood, and non-pulsatile arterial blood) with a small AC pulsatile component riding on top (from the periodic change in arterial blood volume through the cardiac cycle). During systole, increased blood volume absorbs more light, reducing the photocurrent and dipping the TIA output; during diastole, blood volume drops, more light reaches the photodiode, and the output rises. This produces the characteristic dip-and-rise PPG pulse shape.

---

## Tools Used

| Tool | Purpose |
|---|---|
| **LTSpice** | Circuit simulation, transient and AC/Bode analysis for every stage |
| **Digital Storage Oscilloscope (Keysight)** | Hardware waveform verification and Frequency Response Analysis |
| **Arduino Uno** | Digital pulse capture and BPM calculation |
