# BCA188 – Laboratory Activity 4: Analog Input, PWM, and DAC

## Hardware & Board Setup
* **Assigned Board:** ESP32-S3-DevKitC-1 (`ESP32-S3-WROOM` module)[cite: 1]
* **Pin Adaptation:** Because the ESP32-S3 does not expose `GPIO34`[cite: 1], the 10 kΩ potentiometer wiper was connected to **`GPIO4` (`ADC1_CH3`)**[cite: 1], and the PWM brightness LED was connected to **`GPIO19`** through a 330 Ω resistor[cite: 1].
* **Equipment Note (DAC Limitation):** Unlike the original ESP32 (which includes two 8-bit hardware DAC channels on `GPIO25` and `GPIO26`), the **ESP32-S3 lacks built-in hardware DAC peripherals** (`dacWrite()` is unsupported on this architecture)[cite: 1]. Per the laboratory equipment note, Example 5 is included as target firmware for an original ESP32 / external DAC setup, and **no PWM measurements from `GPIO19` are claimed as DAC voltages**. An original ESP32 or external DAC setup is requested to complete the physical DAC voltage measurements.

```text
3V3 (ESP32-S3)  ── Potentiometer Outer Pin 1
GPIO4 (ADC1_3)  ── Potentiometer Middle Pin (Wiper)
GND (ESP32-S3)  ── Potentiometer Outer Pin 2

GPIO19          ── [330 Ω Resistor] ── LED Anode (+) | LED Cathode (−) ── GND
```

---

## Hardware Wiring & Demonstration Media

### Breadboard Circuit Setup
![Lab 4 Circuit Setup](./lab4_img.heic)

---

## Measurement Tables

### Table 1: Potentiometer ADC Readings & PWM Duty (Examples 3 & 4)
* **ADC Configuration:** 12-bit resolution (`0–4095`), `ADC_11db` attenuation on `GPIO4`[cite: 1]
* **PWM Configuration:** 8-bit resolution (`0–255`), `5 kHz` frequency on `GPIO19`
* **Prediction Formula (Arduino integer scaling):** `Predicted Duty = floor((Raw * 255) / 4095)`

| Potentiometer Position | Raw ADC (`0–4095`) | Calibrated (`mV`) | Predicted PWM Duty (`0–255`) | Predicted Duty (`%`) | Observed PWM Duty (`0–255`) | Observed LED (`GPIO19`) |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **1. Minimum (0% Stop)** | `0`[cite: 2] | `0 mV`[cite: 2] | `0` | `0.0%` | `0` | Off |
| **2. Low Position** | `111`[cite: 4, 7] | `101 mV`[cite: 4] | `6` | `2.4%` | `6` *(6–7)*[cite: 7] | Very dim glow |
| **3. Low-Mid Position** | `555`[cite: 3, 8] | `477 mV`[cite: 3] | `34` | `13.3%` | `34`[cite: 8] | Dim brightness |
| **4. High-Mid Position** | `3341`[cite: 5, 9] | `2703 mV`[cite: 5] | `208` | `81.6%` | `208` *(207–208)*[cite: 9] | Bright |
| **5. Maximum (100% Stop)** | `4095`[cite: 6, 10] | `3121 mV`[cite: 6] | `255` | `100.0%` | `255`[cite: 10] | Full brightness |

### Table 2: Hardware DAC Output Voltage Settings (Example 5)
* **Prediction Formula:** `V_predicted = (Code / 255) * 3.3 V` (on a supported original ESP32 `GPIO25`)

| DAC Code (`0–255`) | Predicted Voltage (`V`) | Measured Voltage (`V`) | Hardware Status |
| :---: | :---: | :---: | :--- |
| **`0`** | `0.00 V` | `N/A` | ESP32-S3 lacks internal DAC (`GPIO25` DAC1 unavailable)[cite: 1] |
| **`64`** | `0.83 V` | `N/A` | Requires original ESP32 or external DAC setup |
| **`128`** | `1.66 V` | `N/A` | Requires original ESP32 or external DAC setup |
| **`192`** | `2.48 V` | `N/A` | Requires original ESP32 or external DAC setup |
| **`255`** | `3.30 V` | `N/A` | Requires original ESP32 or external DAC setup |

---

## Technical Explanations

### 1. Waveform Comparison: PWM (`GPIO19`) vs. DAC (`GPIO25`)
* **GPIO19 (LEDC PWM):** Generates a **5 kHz digital square wave** (200 µs period) switching rapidly between two discrete logic levels: `0 V` (`LOW`) and `3.3 V` (`HIGH`). Turning the potentiometer changes only the pulse width (HIGH on-time vs. LOW off-time), while peak amplitude remains fixed at `3.3 V`.
* **GPIO25 on Original ESP32 (Hardware DAC):** Generates a **steady, continuous DC voltage line** stepping through five discrete analog levels every two seconds, with no switching pulses.

### 2. Why PWM Is Not the Same Signal as a DAC Output
* **Switched Logic States vs. Continuous Analog Voltage:** A PWM pin only ever outputs `0 V` or `3.3 V`, relying on rapid switching and time-averaging so an LED or multimeter perceives an average intensity or voltage. By contrast, a hardware DAC uses an internal resistor-ladder network to output an actual steady DC voltage between `0 V` and the supply rail.
* **Downstream Circuit Behavior:** Because raw PWM still contains full-scale `3.3 V` switching edges, feeding it directly into analog inputs (such as audio amplifiers or analog sensor references) introduces switching noise unless smoothed by an external low-pass RC filter. A DAC output provides a clean DC voltage level directly.

### 3. Why an ADC Endpoint May Saturate
* **Narrower Measurable Range (`ADC_11db`):** With `ADC_11db` attenuation configured, the ADC's linear measurable ceiling is lower than the full `3.3 V` supply rail across the potentiometer (peaking around `3121 mV` on the tested board)[cite: 6].
* **Upper and Lower Rail Clipping:** Once the potentiometer wiper exceeds ~`3121 mV` near full clockwise rotation, the ADC reaches its maximum conversion ceiling early and **saturates at `4095`**[cite: 6]—meaning `4095` indicates "at or above the measurable ceiling" rather than an exact `3.30 V`. Similarly, voltages near `0 V` fall into the ADC's lower dead zone and clip to `0`[cite: 2] before reaching the mechanical stop.

---

## Comparison of Predicted and Observed Results

Across all five tested potentiometer positions on the **ESP32-S3-DevKitC-1**[cite: 1, 2], the observed PWM duty settings in Example 4 matched the integer-truncated predictions from `map(raw, 0, 4095, 0, 255)` (`0`, `6`, `34`, `208`, and `255`)[cite: 7, 8, 9, 10]. Minor single-count fluctuations (such as `Raw: 3340` producing `Duty: 207` and `Raw: 3341` producing `Duty: 208`[cite: 9]) occurred right at integer truncation boundaries due to normal LSB ADC noise. At the upper mechanical stop, the ADC saturated at `Raw: 4095` when the calibrated input reached `3121 mV`[cite: 6], confirming endpoint saturation below the `3.3 V` supply rail. Because the assigned ESP32-S3 lacks hardware DAC channels[cite: 1], Example 5 was documented as reference code without substituting PWM readings for DAC measurements.
