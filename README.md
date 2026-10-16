Here is a technical, architecture-focused README designed specifically to showcase the **Versal ACAP & AI Engine Architecture**, how the **AI Engine (AIE) array works**, and how **Vitis AI / DPU engine** is structured for the **VCK190**.

You can copy and paste this directly into your **`Versal_AI_Dev/README.md`** file.

---

```markdown
# Versal ACAP Architecture & Edge AI Development Hub

This repository serves as a technical hub for exploring, designing, and deploying compute-intensive applications on the **AMD Xilinx Versal Adaptive Compute Acceleration Platform (ACAP)**, specifically targeting the **VCK190 Evaluation Kit** (VCK190-Versal AI Core VC1902).

---

## 🏛️ Versal ACAP Hardware Architecture

Unlike traditional FPGAs, the Versal ACAP is a revolutionary heterogeneous compute architecture that integrates three distinct, tightly coupled domains connected via a high-bandwidth **Network-on-Chip (NoC)**:


```

┌────────────────────────────────────────────────────────────────────────┐
│                        VERSAL ACAP ARCHITECTURE                        │
├─────────────────────────┬──────────────────────┬───────────────────────┤
│    PROCESSING SYSTEM    │ PROGRAMMABLE LOGIC   │   AI ENGINE ARRAY     │
│          (PS)           │         (PL)         │        (AIE)          │
│                         │                      │                       │
│  • Dual-Core ARM A72    │  • Configurable CLBs │  • 2D Array Tiles     │
│  • Dual-Core ARM R5F    │  • DSP58 Blocks      │  • Vector Processors  │
│  • DDR4 / LPDDR4 Ctrl   │  • UltraRAM / BRAM   │  • Dedicated Memory   │
└───────────┬─────────────┴──────────┬───────────┴───────────┬───────────┘
│                        │                       │
└────────────────────────┼───────────────────────┘
▼
AXI Network-on-Chip (AXI NoC)

```

1. **Processing System (PS):** Dual-core ARM Cortex-A72 for high-level OS/baremetal host orchestration and dual-core ARM Cortex-R5F for real-time processing.
2. **Programmable Logic (PL):** Traditional FPGA fabric containing DSP58 blocks, UltraRAM, and CLBs for custom datamovers, stream interfaces, and hardware overlays.
3. **AI Engine (AIE) Array:** A dedicated 2D grid of high-performance vector processing tiles designed for compute-heavy matrix math, signal processing, and neural network primitives.

---

## 🧩 What is the AI Engine (AIE)?

The **AI Engine (AIE)** is a non-cached, VLIW (Very Long Instruction Word) / SIMD (Single Instruction Multiple Data) vector processor capable of performing multiple MAC (Multiply-Accumulate) operations per clock cycle.


```

```
              ┌─────────────────────────────────────────┐
              │              AI ENGINE TILE             │
              │                                         │
              │  ┌───────────────────────────────────┐  │
              │  │ Vector Processing Unit (VPU)      │  │
              │  │ (SIMD / VLIW - 512-bit Vector)    │  │
              │  └─────────────────┬─────────────────┘  │
              │                    │                    │
              │  ┌─────────────────┴─────────────────┐  │
              │  │ 32 KB Data Memory (4 Banks)       │  │
              │  └─────────────────┬─────────────────┘  │
              │                    │                    │
              │  ┌─────────────────┴─────────────────┐  │
              │  │ Stream Switch & AXI Interfaces    │  │
              │  └───────────────────────────────────┘  │
              └─────────────────────────────────────────┘

```

```

### Key Components of an AIE Tile:
* **Vector Processor (VPU):** Fixed-point and floating-point SIMD vector units capable of 32x 16-bit MACs or 128x 8-bit MACs per cycle per tile.
* **Local Data Memory:** 32 KB of local SRAM per tile split into 4 memory banks, allowing simultaneous multi-directional access without cache misses.
* **Inter-Tile Memory Sharing:** Each tile can access the local memory of neighboring tiles in north, south, east, and west directions with zero latency penalty.
* **AIE Interconnect & Streaming:** Uses dedicated streaming interconnects (cascade streams, AXI-Stream) to push data directly across adjacent tiles without routing through the PL.

### How the AIE is Used in Practice:
* **Custom Compute Graphs (ADF Graph):** Writing vectorized C++ kernels mapped across the AIE array using adaptive data flow graphs (`graph.h`).
* **High-Bandwidth Datamovers (MM2S / S2MM):** HLS-based streaming engines in the PL fetch tensor data from DDR memory and stream it into the AIE Shim PLIO ports.
* **DSP & Wireless Signal Processing:** Filtering, Fast Fourier Transforms (FFT), matrix multiplication, and beamforming algorithms.

---

## 🧠 Vitis AI Stack & DPU Architecture on VCK190

**Vitis AI** is AMD Xilinx's development platform for inferencing deep learning models on Xilinx adaptive SoCs and Alveo acceleration cards.

Rather than manually writing low-level C++ code for every neural network layer, Vitis AI abstracts the underlying hardware using a soft-IP core called the **Deep Learning Processor Unit (DPU)**.

### The DPU Architecture on VCK190: `DPUCVDX8G`

For the Versal VC1902 ACAP (VCK190), Vitis AI utilizes the **`DPUCVDX8G`** (DPU Core for Versal Devices targeting 8-bit INT8 quantization):


```

┌────────────────────────────────────────────────────────────────────────┐
│                        DPUCVDX8G ARCHITECTURE                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   ┌───────────────────────────┐       ┌────────────────────────────┐   │
│   │   AI ENGINE (AIE) ARRAY   │       │     PROGRAMMABLE LOGIC     │   │
│   │                           │       │            (PL)            │   │
│   │  • Convolution Compute    │       │  • Element-wise Engines    │   │
│   │    Engine (Conv / GEMM)   │◄─────►│  • Pooling / Activation    │   │
│   │  • High-throughput Vector │ Stream│  • Memory Controller &     │   │
│   │    Matrix MAC Ops         │       │    Instruction Scheduler   │   │
│   └───────────────────────────┘       └────────────────────────────┘   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

```

#### How `DPUCVDX8G` Works on Versal:
1. **AIE + PL Co-Execution:** Unlike previous Zynq UltraScale+ DPUs (which executed entirely in PL fabric), the Versal `DPUCVDX8G` splits the workload:
   * **AIE Array:** Executes high-density 2D/3D Convolution, GEMM (General Matrix Multiply), and heavy matrix compute.
   * **PL Fabric:** Handles instruction fetch, activation layers (ReLU, LeakyReLU), pooling (Max/Avg), element-wise operations, and DMA memory arbitration.
2. **Instruction Engine:** The DPU reads specialized hardware instructions compiled by the Vitis AI Compiler (`vai_c_xcompiler`) from DDR.
3. **High Efficiency:** Delivers massive INT8 throughput per watt for real-time computer vision, object detection, and multi-stream CNN inference.

---

## 🛠️ Complete Compilation & Execution Workflow

```text
[ PyTorch / TensorFlow / ONNX Model ]
                  │
                  ▼
   [ Vitis AI Quantizer (vai_q) ] ──► (Converts FP32 to INT8)
                  │
                  ▼
   [ Vitis AI Compiler (vai_c) ]  ──► (Generates compiled .xmodel)
                  │
                  ▼
 [ Versal VCK190 Target (VART/XIR) ] ──► (Runs on DPU + AIE Array)

```

1. **Quantization (`vai_q_pytorch` / `vai_q_tensorflow`):** Converts 32-bit floating-point (FP32) network weights into fixed 8-bit integer (INT8) representation without accuracy degradation.
2. **Compilation (`vai_c_xcompiler`):** Maps the network graph operators to the target architecture (`DPUCVDX8G`) and generates an executable `.xmodel`.
3. **Execution (VART Runtime):** The ARM host CPU uses the Vitis AI Runtime (VART) API to pass image/video buffer tensors to the DPU overlay for hardware-accelerated execution.

---



```
