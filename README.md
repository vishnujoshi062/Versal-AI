# Versal ACAP Hardware & AI Engine Development Hub

A comprehensive development workspace for **AMD Xilinx Versal ACAPs** (specifically targeting the **VCK190 Evaluation Kit**). This repository hosts low-level **AI Engine (AIE)** baremetal designs, custom High-Level Synthesis (**HLS**) stream controllers, and high-level **Vitis AI** deep learning model compilation workflows.

---

## 🏛️ Versal ACAP Heterogeneous Architecture

The Versal Adaptive Compute Acceleration Platform (ACAP) integrates three distinct compute domains connected through a high-bandwidth, low-latency **AXI Network-on-Chip (NoC)**:

| Compute Domain | Primary Hardware Features | Key Use Cases |
| :--- | :--- | :--- |
| **Processing System (PS)** | Dual-core ARM Cortex-A72 & Dual-core ARM Cortex-R5F | OS execution (PetaLinux/Baremetal), system orchestration, and I/O management |
| **Programmable Logic (PL)** | Configurable Logic Blocks (CLBs), DSP58 blocks, UltraRAM/BRAM | Custom hardware overlays, AXI4-Stream datamovers, and protocol controllers |
| **AI Engine (AIE) Array** | 2D tile grid of SIMD/VLIW vector processing units | High-throughput vector math, matrix multiplication, and spatial signal processing |

---

## 🧩 The AI Engine (AIE) Array

The **AI Engine (AIE)** is a 2D array of non-cached, Very Long Instruction Word (VLIW) and Single Instruction Multiple Data (SIMD) vector processing tiles designed for compute-dense workloads.

### AIE Tile Anatomy
* **Vector Processor Unit (VPU):** Performs up to **32x 16-bit MACs** or **128x 8-bit MACs** per clock cycle per tile across 512-bit vector registers.
* **Local Data Memory:** Each tile contains **32 KB of multi-bank SRAM** for zero-latency, cache-miss-free memory access.
* **Inter-Tile Routing:** Dedicated streaming networks (cascade streams and direct North/South/East/West tile-to-tile routing) allow data transfer without accessing PL fabric or DDR memory.

### Operational Flow
1. **Adaptive Data Flow (ADF) Graphs:** C++ kernels are organized into data-flow graphs (`graph.h`) specifying execution dependencies and memory bindings.
2. **PL Stream Interfacing (MM2S / S2MM):** HLS datamovers stream data from DDR memory through AXI4-Stream PLIO ports into the AIE array.

---

## 🧠 Vitis AI Stack & DPU Architecture (`DPUCVDX8G`)

For deep learning workloads on the VCK190 board, **Vitis AI** utilizes the **`DPUCVDX8G`** (Deep Learning Processor Unit for Versal AI Core devices targeting INT8 operations).

### Heterogeneous Execution Model
Unlike previous FPGA-only DPU IP cores, `DPUCVDX8G` splits neural network execution across two domains:

* **AIE Array:** Executes high-density **2D/3D Convolutions, GEMM (General Matrix Multiplication)**, and heavy tensor operations.
* **PL Fabric:** Handles instruction fetch engines, **Pooling (Max/Avg)**, activation functions (ReLU, LeakyReLU), element-wise operations, and memory DMA control.

---

## 🛠️ Deep Learning Compilation & Execution Pipeline

```text
[ PyTorch / TensorFlow Model ]
              │
              ▼
  [ Vitis AI Quantizer ]   ──► Quantizes FP32 weights to INT8 (vai_q_pytorch)
              │
              ▼
   [ Vitis AI Compiler ]   ──► Compiles graph to DPU instructions (.xmodel)
              │
              ▼
 [ Versal Target Execution ] ──► Vitis AI Runtime (VART) executes on DPU + AIE
