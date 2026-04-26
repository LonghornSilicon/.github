<div align="center">

# Longhorn Silicon

**Chips, designed at Texas.**

*The first student-led silicon design team at The University of Texas at Austin.*

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Process Node](https://img.shields.io/badge/process-TSMC%2016nm-orange.svg)](https://longhornsilicon.com)
[![University](https://img.shields.io/badge/university-UT%20Austin-bf5700.svg)](https://utexas.edu)
[![Contact](https://img.shields.io/badge/contact-info%40longhornsilicon.com-lightgrey.svg)](mailto:info@longhornsilicon.com)

[Website](https://longhornsilicon.com) · [Contact Us](mailto:info@longhornsilicon.com) · [Join the Team](#join-us)

</div>

---

## Who We Are

Longhorn Silicon is a student-run semiconductor design organization at UT Austin. We design, verify, and tape out real silicon — not toy projects. Our mission is to train the next generation of chip architects by building hardware that solves hard problems in modern AI inference.

> *Silicon is the new frontier. We're training the people who'll shape it.*

We work across the full stack: architecture exploration, RTL design, functional verification, physical design, and post-silicon validation. Our research informs real design decisions with measurable impact on area, power, and performance.

---

## Flagship Project — Lambda

**Lambda** (Longhorn Accelerator for Matrix-Based Dataflow & Attention) is our first production chip: a purpose-built AI inference accelerator targeting on-device LLMs up to 200M parameters.

| Attribute | Specification |
|-----------|--------------|
| Process node | TSMC 16nm FinFET (University Program via imec) |
| Form factor | M.2 module |
| Memory interface | LPDDR5 |
| Target workload | Autoregressive LLM inference |
| Frontend HDL | SystemVerilog + UVM |
| Backend EDA | Cadence / Synopsys |

Lambda treats KV cache management as a **first-class hardware function** — moving it off the GPU's HBM and onto a dedicated coprocessor with its own compression, importance scoring, and tiered memory hierarchy. The result: 3–4× more effective context in the same memory bandwidth.

---

## Architecture Overview

Lambda is organized around four tightly coupled functional blocks:

### KV Cache Engine (KVE)

A large on-chip memory store (SRAM + optional eDRAM) that holds the most recently used Key and Value vectors for transformer attention. The KVE sits between the attention compute core and off-chip DRAM, intercepting every KV write and read.

**Compression modes supported:**

| Mode | Technique | Bits/Element | Notes |
|------|-----------|:---:|-------|
| Quantization | GEAR-style 4-bit + low-rank residual | 4 | Error correction via sparse outlier matrix |
| Aggressive quant | RotateKV-style coordinate rotation | 2 | Rotation eliminates outliers before quantization |
| Sparse coding | Dictionary / Lexico-style atom decomposition | variable | 4–8 atoms per vector; encoder/decoder in hardware |

The quantizer hardware is a comparator tree mapping each float to the nearest codebook entry; the dequantizer is a small lookup table on the read path — both are single-cycle in the critical path.

### Token Importance Unit (TIU)

As each decode step processes attention, the TIU accumulates a running importance score per cached token — tracking how much attention weight each past position has received across all heads.

Rather than naively evicting cold tokens (which causes hallucinations), the TIU enables **mixed-precision retention**:

```
Hot tokens   → SRAM, full precision (8–16 bit)
Warm tokens  → eDRAM, demoted to 4-bit
Cold tokens  → evicted or compressed to 2-bit in DRAM
```

Hardware implementation: one accumulator per cached token; the attention core streams a row of scores per step and the TIU adds them to the running tally. A comparator tree then classifies tokens into tiers and signals the memory controller with eviction/demotion commands.

At 16nm we can afford the accumulator array and comparator logic to **track thousands of tokens** simultaneously, and can implement learned scoring metrics beyond raw attention-weight sums.

### Memory Hierarchy Controller (MHC)

Three-level hierarchy, each tier tuned for a different cost/capacity trade-off:

| Level | Technology | Role |
|-------|-----------|------|
| L1 | SRAM | Active working set: query/output buffer, hottest KV entries, current-layer activations |
| L2 | 3T eDRAM (gain-cell) | Bulk on-chip KV store; ~2× denser than SRAM; requires periodic refresh |
| Off-chip | LPDDR5 | Cold/compressed KV, model weights |

The MHC implements **selective refresh** (inspired by SHIELD): data with short lifetimes — such as the query/output buffer rewritten every decode step — can have refresh disabled, saving ~35% of eDRAM energy. The controller tracks per-region data lifetimes to make this safe.

### LASSO — KV Cache Compression Coprocessor

An independent research silicon in the **SKY130** open-source PDK (Efabless chipIgnite program). LASSO is our test vehicle for compression pipelines before they land in Lambda. It implements the full compress/decompress datapath in real ASIC silicon, giving us measured PPA numbers rather than estimated ones.

---

## Repository Map

| Repository | Language | Description |
|-----------|----------|-------------|
| [`architecture`](https://github.com/LonghornSilicon/architecture) | Markdown / YAML | High-level system specs, PRD, design rationale, floorplan |
| [`kelle-simulator`](https://github.com/LonghornSilicon/kelle-simulator) | Python | Cycle-accurate simulator for Kelle (eDRAM + AERP KV-cache, MICRO 2025) |
| [`kelle-fpga`](https://github.com/LonghornSilicon/kelle-fpga) | C++ (HLS) | Vivado HLS prototype targeting Xilinx ZCU104 |
| [`kv-accelerator-comparison`](https://github.com/LonghornSilicon/kv-accelerator-comparison) | Python | Head-to-head evaluation: Titanus vs. Kelle on KV cache metrics |
| [`dont-waste-bits`](https://github.com/LonghornSilicon/dont-waste-bits) | Python | Adaptive KV-cache quantization for lightweight on-device LLMs |
| [`decode-pipeline-optimizer`](https://github.com/LonghornSilicon/decode-pipeline-optimizer) | Python | Hardware speculative decoding assist for improved decode throughput |
| [`turboquant`](https://github.com/LonghornSilicon/turboquant) | Python | KV cache compression research for LLM inference |
| [`fp4-multiplier`](https://github.com/LonghornSilicon/fp4-multiplier) | Python | Minimum-gate FP4 multiplier, fully verified |
| [`lasso`](https://github.com/LonghornSilicon/lasso) | Python / TeX | SKY130 ASIC compression coprocessor (Efabless chipIgnite) |
| [`SRAM_16384x32`](https://github.com/LonghornSilicon/SRAM_16384x32) | Verilog | 16K×32 SRAM macro |

---

## Toolchain

**Design & Verification**
- RTL: SystemVerilog
- Verification: UVM, Python (pytest, numpy, scipy)
- Simulation: Cadence Xcelium / open-source Verilator
- Synthesis & P&R: Cadence Genus + Innovus / Synopsys DC + ICC2
- FPGA prototyping: Vivado HLS → Xilinx ZCU104

**Open-Source Flow (LASSO)**
- PDK: SkyWater SKY130
- Flow: OpenLane
- Fabrication: Efabless chipIgnite

---

## Project Roadmap

```
Spring 2026   Charter & tooling establishment         ████████░░  In progress
Fall 2026     Architecture finalization               ░░░░░░░░░░  Upcoming
Spring 2027   RTL design freeze                       ░░░░░░░░░░  Upcoming
TBD           Tapeout (TSMC 16nm via imec)            ░░░░░░░░░░  Planned
Post-silicon  Bring-up & validation                   ░░░░░░░░░░  Planned
```

---

## Partners & Collaborators

- **Laboratory for Computer Architecture** — The University of Texas at Austin
- **imec / TSMC University FinFET Program** — Fabrication access
- **ChipAgents** — Industry collaboration

---

## Join Us

We are actively recruiting motivated students from UT Austin across all experience levels. Whether you have RTL experience or are learning your first Verilog module, there is a role for you.

**Open roles:** RTL design, verification, physical design, firmware, architecture research.

Reach out at **[info@longhornsilicon.com](mailto:info@longhornsilicon.com)** or visit **[longhornsilicon.com](https://longhornsilicon.com)**.

---

## License

Unless otherwise noted, code in this organization is released under the [MIT License](LICENSE).  
Hardware designs may carry additional terms — see individual repository licenses.

---

<div align="center">

*Built at UT Austin. Designed to last.*

</div>
