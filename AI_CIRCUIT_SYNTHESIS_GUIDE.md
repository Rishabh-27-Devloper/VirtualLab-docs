# 🤖 VirtualLab-HIL — Gemini AI Circuit Synthesis Guide
**Natural Language to Functional Circuit Schematics via Google Gemini 3.6 Flash**

---

## 1. Overview & Architecture

The Gemini AI Circuit Assistant allows engineers and students to describe any electronic circuit in plain English. The AI synthesizes the electronic topology, assigns realistic component parameters, and compiles the schematic into clean auto-routed canvas nodes and wires.

```
┌────────────────────────┐      Prompt      ┌─────────────────────────┐
│ User Natural Language  ├─────────────────►│   Gemini 3.6 Flash      │
│ "Op-Amp Inverting Amp" │                  │   (JSON Circuit Spec)   │
└────────────────────────┘                  └────────────┬────────────┘
                                                         │ Netlist JSON
                                                         ▼
                                            ┌─────────────────────────┐
                                            │ Circuit Layout Compiler │
                                            │ (Pin Alias & Auto-Route)│
                                            └────────────┬────────────┘
                                                         │ React Flow Nodes & Edges
                                                         ▼
                                            ┌─────────────────────────┐
                                            │  Interactive Canvas     │
                                            └─────────────────────────┘
```

---

## 2. API Key Configuration & Dual Execution Pipeline

The AI Circuit Assistant operates in two interchangeable modes:

### Mode 1: Browser-Direct (Recommended)
- Users enter their personal Google Gemini API key directly in the AI Circuit Modal.
- The key is saved locally in encrypted browser storage (`localStorage`) and communicates directly with the Google AI API endpoint.
- Zero server queue latency.

### Mode 2: Cloud Backend Proxy with Key Pool & Failover
- If no user key is provided, requests are routed to the FastAPI cloud gateway on Render.
- Features automatic multi-key failover with exponential backoff and rate-limit cooldown recovery across shared keys.

---

## 3. Prompt Crafting Best Practices

To obtain clean, mathematically accurate circuit layouts, follow these guidelines:

### Guideline 1: Specify Excitation & Power
Explicitly name your voltage source types and operating values:
> *"Design a common-emitter amplifier with 10V DC VCC, Ground, and a 10Hz 50mV AC voltage source."*

### Guideline 2: Specify Target Component Values or Functions
> *"Use an NPN transistor with beta=100, collector resistor 1kΩ, base resistor 180kΩ, and 10µF coupling capacitors."*

### Guideline 3: Mention Measurement Probes
> *"Attach an oscilloscope probe to the output load resistor and another to the input signal source."*

---

## 4. Curated Prompt Library

### 1. Center-Tapped Full-Wave Rectifier
```
Create a full-wave rectifier using a center-tapped transformer (transformer_ct) with turns ratio 1.0, driven by a 10Hz 12V AC voltage source. Connect two diodes (diode) to the secondary terminals, tie the cathodes to a 100µF capacitor and 1kΩ load resistor, connect the center tap to Ground, and add an oscilloscope probe across the load.
```

### 2. 4:1 Multiplexer Data Selector
```
Construct a 4-to-1 multiplexer circuit using a mux_4to1 component. Connect four digital_input switches to inputs I0, I1, I2, and I3, two digital_input switches to select lines S0 and S1, and wire the output Y to a digital_output logic probe.
```

### 3. BCD to 7-Segment Display Decoder
```
Build a BCD to 7-segment decoder circuit using a decoder_bcd_to_7seg IC. Wire four digital_input switches to binary lines A, B, C, and D, and connect all seven segment outputs (a through g) to individual digital_output probes.
```

### 4. Op-Amp Inverting Amplifier
```
Create an inverting amplifier using an opamp with internal ±15V rails. Connect a 10kΩ resistor Rin to the inverting input, a 50kΩ resistor Rf in negative feedback between output and inverting input, ground the non-inverting input, drive Rin with a 1V 10Hz AC voltage source, and place oscilloscope probes at input and output.
```

### 5. 4-Bit Binary Ripple Counter
```
Create a 4-bit binary counter circuit using a counter_4bit component. Connect a clock_source with 5Hz frequency to CLK, a digital_input switch to CLR, and wire all four counter outputs Q0, Q1, Q2, and Q3 to digital_output probes.
```

---

## 5. Intelligent Layout Compiler & Auto-Routing

The circuit layout compiler (`src/services/circuitLayoutCompiler.ts`) performs intelligent grid alignment and semantic pin alias resolution:
- **Transformers**: Resolves `pri1`/`primary1` $\to$ `p1`, `sec1`/`secondary1` $\to$ `s1`, `ct`/`center` $\to$ `ct`.
- **Multiplexers**: Resolves `D0..D3` $\to$ `I0..I3`, `SEL0..SEL1` $\to$ `S0..S1`.
- **Decoders**: Resolves BCD data lines `IN0..IN3` $\to$ `A..D`, segment outputs `OUT_A..OUT_G` $\to$ `a..g`.
- **Separation & Spacing**: Enforces minimum horizontal spacing ($180\text{px}$) and vertical spacing ($120\text{px}$) to eliminate overlapping component cards.
