# 📚 VirtualLab-HIL — Component Encyclopedia & Technical Reference
**Comprehensive Electrical & Mathematical Specifications for All 45+ Components**

---

## 1. Passives & Discrete Components

### Resistor (`resistor`)
- **Pins**: `p`, `n`
- **Parameters**: `resistance` (Ohms, default $1000\Omega$), `tolerance` (%, default 5%), `powerRating` (Watts, default 0.25W).
- **MNA Matrix Stamp**:
  $$G = \frac{1}{R}, \quad A[n_p][n_p] += G, \quad A[n_n][n_n] += G, \quad A[n_p][n_n] -= G, \quad A[n_n][n_p] -= G$$

### Capacitor (`capacitor`)
- **Pins**: `p`, `n`
- **Parameters**: `capacitance` (Farads, default $1\mu\text{F}$), `initialVoltage` (Volts, default 0V).
- **Trapezoidal Integration Companion Model**:
  $$G_{eq} = \frac{2C}{h}, \quad I_{eq}(t) = -G_{eq} v_C(t-h) - i_C(t-h)$$
  Stamps conductance $G_{eq}$ into matrix $A$, and companion current source $I_{eq}$ into vector $z$.

### Inductor (`inductor`)
- **Pins**: `p`, `n`
- **Parameters**: `inductance` (Henries, default $0.01\text{H}$), `initialCurrent` (Amperes, default 0A).
- **Norton Companion Model**:
  $$G_{eq} = \frac{h}{L}, \quad I_{eq} = -I_{prev}$$
  Stamps conductance $G_{eq}$ into matrix $A$, and historical inductor current source into vector $z$.

### Transformers (`transformer`, `transformer_ct`)
- **Pins**:
  - Standard: `p1`, `p2` (Primary), `s1`, `s2` (Secondary).
  - Center-Tapped: `p1`, `p2` (Primary), `s1`, `ct`, `s2` (Secondary with Center-Tap).
- **Parameters**: `turnsRatio` (default 1.0), `primaryInductance` (default 0.1H).
- **MNA Stamping**:
  $$(V_{s1} - V_{s2}) - n(V_{p1} - V_{p2}) = 0$$
  Auxiliary branch current equations stamped into matrix with $10\text{ k}\Omega$ shunt primary magnetizing conductance preventing DC floating singularities.

### Ground (`ground`)
- **Pins**: `p`
- **Behavior**: Physical 0.00V potential reference node ($V_0 = 0$). Essential for Modified Nodal Analysis.

---

## 2. Power & Waveform Sources

### DC Voltage Source (`dc_voltage`)
- **Pins**: `p` (+), `n` (-)
- **Parameters**: `voltage` (Volts, default 5.0V).
- **MNA Stamp**: Augmented branch equation enforcing $V_p - V_n = V_{src}$.

### AC Voltage Source (`ac_voltage`)
- **Pins**: `p` (+), `n` (-)
- **Parameters**: `amplitude` (Volts, default 5.0V), `frequency` (Hz, **default 10.0 Hz**), `phase` (degrees, default 0), `offset` (Volts, default 0), `waveform` (`sine`, `square`, `triangle`, `sawtooth`).
- **Instantaneous Stamping**:
  $$v(t) = V_{offset} + V_{amp} \cdot \sin(2\pi f t + \phi)$$

### DC Current Source (`current_source`)
- **Pins**: `p` (out), `n` (in)
- **Parameters**: `current` (Amperes, default 0.001A = 1mA).

---

## 3. Semiconductors & Active ICs

### Diode (`diode`)
- **Pins**: `p` (Anode), `n` (Cathode)
- **Parameters**: `saturationCurrent` ($I_s$, default $10^{-14}\text{A}$), `ideality` ($n$, default 1.0), `breakdownVoltage` ($V_{br}$, default 50V).
- **Shockley Non-Linear Linearization**:
  $$I_D = I_s \left( e^{\frac{V_D}{n V_t}} - 1 \right), \quad g_d = \frac{dI_D}{dV_D} = \frac{I_D + I_s}{n V_t}$$
  Stamps dynamic conductance $g_d$ into $A$, and companion current into RHS vector $z$.

### Zener Diode (`zener`)
- **Pins**: `p` (Anode), `n` (Cathode)
- **Parameters**: `zenerVoltage` ($V_z$, default 5.1V), `zenerImpedance` ($Z_z$, default $10\Omega$).
- **Breakdown Model**:
  When $V_D < -V_z$, stamps low dynamic impedance $G_z = 1/Z_z$ and Norton breakdown current into matrix $A$ and vector $z$.

### BJT NPN & PNP (`bjt_npn`, `bjt_pnp`)
- **Terminals**: `base`, `collector`, `emitter`
- **Parameters**: `beta` ($\beta$, default 100), `earlyVoltage` ($V_A$, default 100V), `saturationCurrent` ($I_s$, default $10^{-14}\text{A}$).
- **Ebers-Moll with Early Effect Linearization**:
  $$I_C = I_s e^{\frac{V_{BE}}{V_t}} \left(1 + \frac{V_{CE}}{V_A}\right), \quad g_m = \frac{\partial I_C}{\partial V_{BE}}, \quad g_{ce} = \frac{\partial I_C}{\partial V_{CE}}$$

### MOSFET N & P Channel (`mosfet_n_enh`, `mosfet_p_enh`, `mosfet_n_dep`, `mosfet_p_dep`)
- **Terminals**: `gate`, `drain`, `source`
- **Parameters**: `vth` ($V_{th}$, default 2.0V N-channel / -2.0V P-channel), `kn` ($K_n$, default $0.002\text{ A/V}^2$), `lambda` ($\lambda$, default $0.02\text{ V}^{-1}$).
- **Square-Law Saturation Equations**:
  $$I_D = \frac{K_n}{2}(V_{GS} - V_{th})^2 (1 + \lambda V_{DS})$$

### Operational Amplifier (`opamp`)
- **Terminals**: `inp` (Non-inverting +), `inn` (Inverting -), `out` (Output), `vcc` (+ Rail), `vee` (- Rail).
- **Parameters**: `vcc` (Volts, default 15V), `vee` (Volts, default -15V), `openLoopGain` ($A_{ol}$, default 100,000), `rout` ($R_{out}$, default $50\Omega$).
- **Internal vs External Rail Modes**:
  - *Internal Rails Mode*: Rails are virtual. No ERC floating errors. Stamped as linearized transconductance VCVS clamped between $V_{ee}$ and $V_{cc}$.
  - *External Rails Mode*: Terminals `vcc` and `vee` require physical circuit connections. Disconnection triggers ERC floating terminal alert.

---

## 4. Digital Combinational & Sequential Logic

### Digital Gates
- **AND (`gate_and`)**, **OR (`gate_or`)**, **XOR (`gate_xor`)**, **NAND (`gate_nand`)**, **NOR (`gate_nor`)**, **XNOR (`gate_xnor`)**: Inputs `A`, `B`; Output `out`.
- **NOT / Inverter (`gate_not`)**: Input `A`; Output `out`.
- **MNA Interface**: Output drives a 50Ω CMOS totem-pole source stamped into analog matrix equations ($5.0\text{V}$ for HIGH, $0.0\text{V}$ for LOW).

### Combinational MSI Blocks
- **Multiplexers (`mux_2to1`, `mux_4to1`, `mux_8to1`)**: Inputs `I0..In`, Select lines `S0..Sm`, Enable `EN`, Outputs `Y`, `Ybar`.
- **Demultiplexers (`demux_1to2`, `demux_1to4`)**: Input `IN`, Select lines `S0..Sm`, Enable `EN`, Outputs `Y0..Yn`.
- **Priority Encoders (`encoder_4to2`, `encoder_8to3`)**: Active lines `D0..Dn`, Outputs `Y0..Ym`, Valid flag `V`.
- **Decoders (`decoder_2to4`, `decoder_3to8`, `decoder_bcd_to_7seg`)**: Address inputs `A0..An`, Outputs `Y0..Ym`.

### Sequential Logic & Storage
- **Latches**: `latch_sr`, `latch_d`, `latch_jk`.
- **Edge-Triggered Flip-Flops**: `ff_d` (Data), `ff_t` (Toggle), `ff_jk` (JK), `ff_sr` (Set-Reset). Terminals: `CLK`, `D`/`J`/`K`/`T`, `CLR`, `Q`, `Qbar`.
- **Binary Counter (`counter_4bit`)**: Synchronous 4-bit up-counter with `CLK`, `CLR`, `EN`, `Q0..Q3`, Terminal Count `TC`.

---

## 5. Virtual Instruments & I/O Transducers

| Component | Kind | Terminal Pins | Measurement Capabilities |
|---|---|---|---|
| **Oscilloscope** | `oscilloscope` | `p` (sense), `n` (gnd) | Dual-channel waveform capture, Vpp, Vrms, Frequency, FFT. |
| **Logic Analyzer** | `logic_analyzer` | `d0`–`d7` | 8-Channel digital timing diagrams with state/edge triggers. |
| **DC Voltmeter** | `voltmeter` | `p`, `n` | High-impedance ($10\text{ M}\Omega$) potential difference readout. |
| **DC Ammeter** | `ammeter` | `p`, `n` | Zero-burden ($0.001\Omega$) inline branch current readout. |
| **OhmMeter** | `ohmmeter` | `p`, `n` | Active $1\text{ mA}$ test current evaluating $R = V_{diff} / 0.001$. |
| **Audio Speaker** | `speaker` | `p`, `n` | Real-time Web Audio API tone synthesis tracking driving frequency. |
| **Logic Input** | `digital_input` | `out` | Clickable switch asserting 0 or 1 with 50Ω CMOS driver. |
| **Logic Probe** | `digital_output` | `in` | LED indicator displaying 0, 1, Z, or X with analog voltage fallback. |
