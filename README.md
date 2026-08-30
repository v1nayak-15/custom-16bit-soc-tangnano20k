# Custom 16-Bit Gate-Level SoC on Tang Nano 20K FPGA

A deterministic, gate-level 16-bit RISC microprocessor and System-on-Chip (SoC) designed from first principles, featuring memory-mapped I/O, custom physical GPIO drivers, and an integrated 400×300 RGB VGA display pipeline synthesized on the **Sipeed Tang Nano 20K FPGA**.

---

## Key Architectural Features

* **16-Bit Harvard Architecture:** Completely isolated 32K Instruction ROM and 16K Data RAM for deterministic, single-cycle instruction throughput.
* **Gate-Level Core:** Unified ALU, dual-purpose address/data register (`A`), computational accumulator (`D`), Program Counter (`PC`), and a discrete combinational Instruction Decoder.
* **Unified Memory-Mapped I/O:** Standard memory instructions (`LOAD`/`STORE`) address Main RAM, custom GPIO physical registers, a keyboard controller, and video RAM seamlessly.
* **Hardware Video Card:** Memory-mapped dual-port SRAM driving a **400×300 RGB VGA output** with custom horizontal and vertical sync generation—updating visuals without interrupting CPU clock cycles.
* **100% Deterministic Execution:** Bare-metal hardware implementation with zero operating system overhead, optimized for hard real-time control applications.

---

## Hardware Platform Specifications

* **FPGA Board:** Sipeed Tang Nano 20K
* **FPGA Chip:** Gowin GW2AR-LV18QN88C8/I7 (Arora Family)
* **Logic Resources:** 20,736 4-input LUTs (LUT4)
* **Internal Memory:** 828 Kbit Block SRAM (BSRAM) + 64 Mbit embedded SDRAM
* **Clock Source:** Onboard 27 MHz crystal oscillator
* **Interface / Flashing:** Onboard BL616 USB-JTAG/UART debugger via USB-C

---

## Memory Map

| Address Range | Module / Peripheral | Description |
| :--- | :--- | :--- |
| `0x0000 - 0x3FFF` | **Main RAM** | 16K general data storage & runtime memory |
| `0x4000 - 0x5FFF` | **Video Framebuffer (VRAM)** | Memory-mapped pixel array (400×300 RGB) |
| `0x6000` | **Keyboard Register** | User input latch register |
| `0x6001` | **GPIO Data Register** | Direct latch to external FPGA I/O pins |
| `0x6002` | **GPIO Direction Register** | I/O pin tri-state mode control |

---

## Project Structure

* `rtl/cpu/`: Verilog sources for the 16-bit processor datapath and control unit.
* `rtl/memory/`: Instruction ROM, Main RAM, and address decoding logic.
* `rtl/video/`: VGA timing engine, pixel generator, and dual-port video RAM.
* `rtl/periph/`: GPIO and peripheral driver blocks.
* `fpga/gowin/`: Gowin IDE project files and physical pin constraints (`.cst`) configured for the GW2AR-18 device.
* `software/`: Assembler scripts and bare-metal test programs.

---

## Building and Flashing

### Option A: Gowin EDA (Official GUI Flow)
1. Open `fpga/gowin/custom_soc_20k.gprj` in **Gowin EDA**.
2. Make sure the target device is set to **GW2AR-LV18QN88C8/I7**.
3. Run **Synthesize** and **Place & Route**.
4. Open **Gowin Programmer** and flash the bitstream (`.fs`) to SRAM or internal Flash over USB-C.
