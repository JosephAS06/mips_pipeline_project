# MIPS 5-Stage Pipelined Processor (Verilog)

This project is an educational implementation of a 5-stage pipelined MIPS processor in Verilog. It is designed to deepen understanding of digital design, CPU architecture, pipelining, hazard resolution, and instruction-level parallelism.

The processor follows the MIPS32 Instruction Set Architecture (ISA) and implements key features such as:
- Instruction and data memory modules
- Register file
- ALU with overflow detection
- Immediate generation and sign-extension
- Pipeline registers (IF/ID, ID/EX, EX/MEM, MEM/WB)
- Hazard detection and forwarding units
- Program Counter (PC) logic and branch/jump support

This project does **not** aim to replicate or distribute any proprietary MIPS Technologies core. It is entirely developed from scratch and intended for **learning, exploration, and academic purposes only**.

---

## Educational Purpose

This repository is maintained as part of a personal academic initiative to practice RTL design and CPU architecture. The work is not affiliated with or endorsed by MIPS Technologies or Imagination Technologies.

Any resemblance to existing MIPS implementations is due to adherence to the ISA specification, which is publicly documented.

---

## Tools Used

- **Language:** Verilog (IEEE 1364-2005)
- **Simulation:** Icarus Verilog, GTKWave
- **Editor:** VS Code / Vim
- **Version control:** Git + GitHub
- **Documentation:** Markdown
- **License:** MIT (see `LICENSE` file)

---

## Project Structure

```text
mips_pipeline_project/
├── src/            # Verilog modules
├── testbench/      # Testbenches for each module
├── doc/            # Markdown documentation
├── programs/       # MIPS assembly test programs (converted to .hex)
├── waveforms/      # GTKWave dump files
├── LICENSE
└── README.md
