# Custom 16-Bit SoC on Tang Nano 20K FPGA

A 16-bit Hack-style processor (built up from logic gates, nand2tetris style) extended into a small System-on-Chip on the **Sipeed Tang Nano 20K**. It has SDRAM-backed data memory, a UART program loader, a memory-mapped HDMI/DVI video output, an I2C ADC interface (ADS1115) and a bank of external LEDs.

---

## Architecture

* **16-bit Harvard CPU:** `A` and `D` registers, 15-bit `PC`, and a combinational ALU using the standard Hack instruction format (`A`-instructions and `C`-instructions with dest/comp/jump fields).
* **Program memory:** 32K x 16-bit block RAM (`program_ram.v`), written at runtime over UART. No `.hex` file is needed to build.
* **Data memory:** 16K words at `0x0000-0x3FFF`, stored in the on-board 64 Mbit SDRAM through a custom controller (`sdram.v`). Each 16-bit word is two byte accesses, with a refresh scheduler and watchdog in `memory.v`.
* **Wait-state handling:** the CPU only advances when the memory system is `ready` (SDRAM idle, no refresh due, GPU not busy, read data valid), so slow peripherals never corrupt execution.
* **Single clock domain:** the CPU, memory and peripherals all run from the 27 MHz board clock. The SDRAM clock is a fixed 180-degree shifted copy produced by an `ODDR`.
* **Video / GPU:** a memory-mapped GPU (`gpu_top.v`) with a command renderer draws into a 160x120 RGB332 framebuffer in block RAM. It is scaled 3x to 480x360, centred in a 720x480 frame at a 27 MHz pixel clock, and sent out over HDMI/DVI using TMDS encoders and `OSER10` serializers (135 MHz serial clock from a Gowin rPLL).
* **Peripherals:** UART receiver, UART program loader, ADS1115 I2C master, 16 external LEDs, 6 on-board debug LEDs.

---

## Hardware Platform

* **Board:** Sipeed Tang Nano 20K
* **FPGA:** Gowin GW2AR-LV18QN88C8/I7 (Arora family)
* **Memory used:** block SRAM for program memory, embedded 64 Mbit SDRAM for data memory
* **Clock:** onboard 27 MHz oscillator
* **Programming / UART:** onboard BL616 over USB-C

---

## Memory Map (word addresses)

| Address | Peripheral | Access | Description |
| :--- | :--- | :--- | :--- |
| `0x0000 - 0x3FFF` | Data RAM (SDRAM) | R/W | 16K x 16-bit general memory |
| `0x6000 - 0x600A` | GPU | R/W | See GPU registers below |
| `0x6010` | ADC channel select | R/W | bits [1:0] choose AIN0-AIN3 |
| `0x6011` | ADC control | R/W | write bit 0 = start conversion; read bit 0 = busy, bit 1 = data valid |
| `0x6012` | ADC data | R | 16-bit signed conversion result |
| `0x6014` | External LEDs | R/W | 16-bit LED output register |
| `0x7FFF` | Keyboard | R | key code (currently tied to 0, USB keyboard removed) |

---

## GPU

Registers (word addresses):

| Address | Register | Notes |
| :--- | :--- | :--- |
| `0x6000` | CONTROL | bit 0 = GPU enable |
| `0x6001` | STATUS | bit 0 = renderer busy (read only) |
| `0x6002` | COMMAND | writing issues the command; one command can be held pending while the renderer is busy |
| `0x6003 / 0x6004` | X / Y | start coordinate (8-bit) |
| `0x6005 / 0x6006` | X2 / Y2 | end coordinate (8-bit) |
| `0x6007` | COLOR | RGB332 (`RRRGGGBB`) |
| `0x6008 - 0x600A` | SIZE, PARAM0, PARAM1 | reserved parameters |

Commands implemented in `gpu_renderer.v`: `1` = DRAW_PIXEL, `2` = CLEAR, `3` = FILL_RECT. Coordinates outside 160x120 are clamped or ignored.

---

## Loading Programs over UART

UART is 115200 baud, 8N1, received on the BL616 serial bridge (pin 70). All multi-byte values are little endian.

| Byte | Command |
| :--- | :--- |
| `0x4C` (`L`) | Load: followed by a 16-bit word count, then that many 16-bit words. The CPU is stopped, the program is written from address 0, then the CPU is reset and started automatically. |
| `0x52` (`R`) | Reset and run the loaded program |
| `0x53` (`S`) | Stop the CPU |

Example (4 words): `4C 04 00 C8 EF 02 00 10 EC 02 60 ...`

---

## ADS1115 ADC

I2C at 100 kHz, address `0x48` (ADDR tied to GND), single-shot conversion at 128 SPS, +-4.096 V range, single-ended inputs. SCL/SDA are on pins 41/42; use external pull-ups to **3.3 V** (not 5 V). The master does not implement clock stretching or NACK recovery.

---

## Repository Layout

```
hdl/                  nand2tetris chip implementations (ALU, Bit, Register, RAM8 ... RAM16K)
rtl/top.v             top-level module
rtl/cpu/              alu.v, cpu.v, computer.v
rtl/memory/           memory.v, program_ram.v, sdram.v
rtl/video/            gpu_top.v, gpu_regs.v, gpu_renderer.v, framebuffer_bram.v,
                      Gowin_rPLL.v, tmds_encoder.v, tmds_serializer.v, video_timing.v
docs/                 development notes (USB keyboard removal, ADS1115 integration)
rtl/periph/           uart_rx.v, uart_loader.v, ads1115_i2c_master.v
fpga/gowin/           tangnano20k.cst (pin constraints)
```

---

## Building and Flashing (Gowin EDA)

1. Create a Gowin project for device **GW2AR-LV18QN88C8/I7** and add every `.v` file under `rtl/` (`video_timing.v` is not instantiated; `gpu_top.v` has its own timing counters).
2. Add `fpga/gowin/tangnano20k.cst` as the physical constraints file.
3. Set `top` as the top module, then run **Synthesize** and **Place & Route**.
4. Flash the generated `.fs` bitstream with **Gowin Programmer** over USB-C.
5. Send a program over the UART with the protocol above.
