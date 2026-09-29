# AIRV CVA6 MAC8IM Method 2 — User Guide

This guide explains how to configure the AIRV/CVA6 environment, compile
the MNIST application, run RTL simulation with Questa, execute the
design on the Zybo Z7-20 FPGA platform, and understand the
implementation of the cleaned MAC8IM Method 2 accelerator.

The implementation described here is the **cleaned MAC8IM Method 2 version**. The implementation section documents the final source code directly, using exact line numbers and code anchors.

------------------------------------------------------------------------

## 1. Project Structure

MAC8IM Method 2 is integrated into the AIRV/CVA6 project. The exact
top-level directory name can vary depending on where the version is
stored or cloned.

A typical AIRV project contains:

``` text
<PROJECTROOT>/
├── core/                         # CVA6 RTL and CV-X-IF logic
├── sw/
│   └── app/                      # Software applications, including MNIST
├── util/
│   ├── gcc-toolchain-builder/    # RISC-V GNU toolchain
│   ├── riscv-opcodes/            # RISC-V instruction definitions
│   └── openocd/                  # OpenOCD
└── setup.sh                      # Environment configuration
```

The `cleaned_code/` directory delivered with this guide contains the source files relevant to the MAC8IM Method 2 implementation.

------------------------------------------------------------------------

## 2. Main Requirements

The AIRV development flow used by this project requires:

-   Linux
-   Xilinx Vivado / Vitis 2024.1 for FPGA implementation
-   QuestaSim for RTL simulation
-   RISC-V GNU toolchain with MAC8IM assembler support
-   OpenOCD
-   Zybo Z7-20 for FPGA execution
-   Digilent JTAG-HS2 cable
-   USB-UART connection

For RTL-only simulation, the FPGA board is not required.

------------------------------------------------------------------------

## 3. Environment Setup

The AIRV project uses `setup.sh` to configure the project root and the
required tools.

The most important variable is:

``` bash
export PROJECTROOT=/absolute/path/to/your/MAC8IM/project
```

Do not copy a machine-specific path blindly. Set `PROJECTROOT` to the
absolute path of the complete CVA6 project into which the cleaned MAC8IM
files are integrated.

A typical setup also places Vivado, Questa, OpenOCD and the project
RISC-V toolchain on `PATH`.

Load the environment with:

``` bash
source /path/to/setup.sh
```

Then verify:

``` bash
echo $PROJECTROOT
which vivado
which vsim
which openocd
which riscv-none-elf-gcc
```

If several RISC-V toolchains are installed, give the MAC8IM-capable
project toolchain priority:

``` bash
export PATH="$PROJECTROOT/util/gcc-toolchain-builder/riscv_toolchain/bin:$PATH"
```

Then check again:

``` bash
which riscv-none-elf-gcc
```

The selected assembler/toolchain must contain the `mac8im` instruction
definition described later in this guide.

------------------------------------------------------------------------

## 4. Compile the MNIST Application

Load the environment:

``` bash
source /path/to/setup.sh
```

Then compile MNIST:

``` bash
cd $PROJECTROOT/sw/app
make mnist
```

A clean rebuild can be performed with:

``` bash
make clean
make mnist
```

The resulting application is normally:

``` text
mnist.riscv
```

The software uses the custom `mac8im` mnemonic, so compilation must use
the modified RISC-V assembler rather than an unrelated system toolchain.

------------------------------------------------------------------------

## 5. Run RTL Simulation with Questa

After loading the environment:

``` bash
cd $PROJECTROOT
make sim APP=mnist
```

The normal AIRV flow builds the RTL/testbench, compiles MNIST and runs
the RTL simulation.

Useful generated files can include:

``` text
vsim.wlf
trace_hart_0.log
uart
```

For MAC8IM debugging, the most important points to inspect are:

``` text
custom instruction decode
rs1 / rs2 / rd
x28 / x29
scoreboard dependency and forwarding
CV-X-IF rs[0:4]
coprocessor products
MAC result
result valid
CPU writeback
```

A useful debugging sequence is given in Section 11.

------------------------------------------------------------------------

## 6. FPGA Execution on Zybo Z7-20

### 6.1 Build and program the FPGA

Load the environment:

``` bash
source /path/to/setup.sh
cd $PROJECTROOT
```

Generate the FPGA image:

``` bash
make cva6_fpga
```

Program the board:

``` bash
make program_cva6_fpga
```

### 6.2 UART

Identify the UART device:

``` bash
ls -l /dev/serial/by-id/ | grep -i uart
```

Then open the detected device, for example:

``` bash
tio /dev/ttyUSB0
```

If access is denied and the machine policy allows it:

``` bash
sudo tio /dev/ttyUSB0
```

### 6.3 OpenOCD

In another terminal:

``` bash
source /path/to/setup.sh
cd $PROJECTROOT/sw/app
openocd -f openocd_digilent_hs2.cfg
```

A successful connection should detect the RISC-V hart and expose the GDB
server, normally on port 3333.

### 6.4 Load MNIST with GDB

In a third terminal:

``` bash
source /path/to/setup.sh
cd $PROJECTROOT/sw/app
riscv-none-elf-gdb mnist.riscv
```

Inside GDB:

``` gdb
target extended-remote :3333
load
c
```

The application output should appear on UART.

------------------------------------------------------------------------

## 7. MAC8IM Instruction Overview

MAC8IM Method 2 accelerates eight INT8 multiply-accumulate operations
with one custom instruction.

The architectural operation is conceptually:

``` text
rd =
    old rd
  + four packed products from rs1 × rs2
  + four packed products from x28 × x29
```

Operand mapping:

``` text
rs1      -> packed input values 0–3
rs2      -> packed weight values 0–3
old rd   -> accumulator
x28 / t3 -> packed input values 4–7
x29 / t4 -> packed weight values 4–7
```

The result returns through CV-X-IF and is written to `rd`.

The custom instruction uses:

``` text
opcode = 0001011
funct3 = 001
funct7 = 0000000
```

The corresponding match/mask values are:

``` text
MATCH_MAC8IM = 0x100b
MASK_MAC8IM  = 0xfe00707f
```

------------------------------------------------------------------------

## 8. Modified RISC-V Toolchain

The software toolchain must recognize the mnemonic:

``` text
mac8im
```

The relevant cleaned files are:

``` text
rv_i
riscv-opc.h
riscv-opc.c
```

The assembler-visible syntax remains:

``` text
mac8im rd, rs1, rs2
```

The additional x28/x29 values are implicit operands established by the
software convention and read explicitly by the modified CPU operand
path.

If the toolchain source is changed, rebuild the project-specific RISC-V
toolchain using the normal AIRV toolchain-builder flow and verify:

``` bash
which riscv-none-elf-gcc
```

Do not assume that a system-installed RISC-V assembler supports
`mac8im`.

------------------------------------------------------------------------

## 9. End-to-End Data Path

``` text
NetworkPropagate.c
       │
       │ mac8im rd, rs1, rs2
       │ + x28/x29 implicit operands
       ▼
GNU assembler / binutils
       │
       ▼
decoder.sv
       │
       │ MAC8IM -> CVXIF
       ▼
issue_read_operands.sv
       │
       ├── rs1
       ├── rs2
       ├── old rd
       ├── x28
       └── x29
       │
       ├── scoreboard.sv
       │     dependency / forwarding
       │
       └── issue_stage.sv
             wiring
       │
       ▼
cvxif_fu.sv
       │
       │ x_issue_req.rs[0:4]
       ▼
CV-X-IF
       │
       ▼
cvxif_example_coprocessor.sv
       │
       │ eight INT8 products
       │ + accumulator
       ▼
CV-X-IF result
       │
       ▼
rd writeback
```

------------------------------------------------------------------------

## 10. Quick Validation Procedure

After integrating the cleaned files into the complete project:

``` bash

# 1. Load environment

source /path/to/setup.sh

# 2. Verify tools

which riscv-none-elf-gcc
which vsim
which vivado
which openocd

# 3. Compile MNIST

cd $PROJECTROOT/sw/app
make clean
make mnist

# 4. Run RTL simulation

cd $PROJECTROOT
make sim APP=mnist
```

Compare the cleaned version with the known working MAC8IM Method 2
behavior:

``` text
MNIST output
predicted class
cycle count
instruction count
custom-instruction waveform
CV-X-IF operands
coprocessor result
CPU writeback
```

For FPGA validation:

``` bash
cd $PROJECTROOT
make cva6_fpga
make program_cva6_fpga
```

Then run the UART + OpenOCD + GDB sequence from Section 6.

------------------------------------------------------------------------

## 11. Recommended Debugging Order

When MAC8IM produces a wrong result, debug in this order:

``` text
1. Check generated disassembly for mac8im
   ↓
2. Check decoder.sv
   opcode / funct3 / MAC8IM operation
   ↓
3. Check issue_read_operands.sv
   rs1 / rs2 / rd / x28 / x29
   ↓
4. Check scoreboard.sv
   dependency and forwarding
   ↓
5. Check cvxif_fu.sv
   x_issue_req.rs[0:4]
   ↓
6. Check coprocessor operand bytes
   ↓
7. Check eight products
   ↓
8. Check accumulator input
   ↓
9. Check MAC result
   ↓
10. Check CV-X-IF result/writeback
```

This order helps distinguish software encoding errors, CPU operand-path
errors, forwarding hazards and arithmetic errors.

------------------------------------------------------------------------

## 12. Troubleshooting

### Wrong RISC-V compiler

Check:

``` bash
which riscv-none-elf-gcc
```

If necessary:

``` bash
export PATH="$PROJECTROOT/util/gcc-toolchain-builder/riscv_toolchain/bin:$PATH"
```

### Assembler does not recognize `mac8im`

Verify that the modified `rv_i`, `riscv-opc.h` and `riscv-opc.c` changes
are present in the toolchain source and that the rebuilt project
toolchain is the one selected by `PATH`.

### Questa does not start

Check:

``` bash
which vsim
echo $MGLS_LICENSE_FILE
echo $LM_LICENSE_FILE
```

The exact license configuration depends on the machine/environment.

### UART has no output

Check the actual serial device:

``` bash
ls -l /dev/serial/by-id/
```

Verify permissions and, if necessary, power-cycle the board before
repeating the OpenOCD/GDB sequence.

### OpenOCD cannot detect the target

Check:

``` text
board power
JTAG-HS2 connection
USB permissions / udev rules
existing OpenOCD processes
```

Find old processes with:

``` bash
ps aux | grep openocd
```

### Simulation and FPGA results differ

First verify that both flows use the same `mnist.riscv` and the same
MAC8IM RTL revision. Then inspect the operand mapping and
result/writeback path.

------------------------------------------------------------------------

---

## 13. Implementation Guide — MAC8IM Method 2

This section describes the final cleaned MAC8IM Method 2 source code directly.

All line numbers refer to the files in `cleaned_code/` delivered with this guide. Each entry explains what was modified or added at that location and why it is needed by MAC8IM.

### 13.1 `NetworkPropagate.c`

This file integrates MAC8IM into the MNIST CNN software.

#### Lines 31–62 — `macsOnRange_mac8im_contiguous()`

Adds the main contiguous MAC8IM helper.

The function processes eight input/weight pairs per MAC8IM operation. Two packed 32-bit input groups and two packed 32-bit weight groups represent eight INT8 values.

The second packed input and weight values are passed through the fixed registers used by MAC8IM Method 2:

```text
x28 / t3 -> second packed input group
x29 / t4 -> second packed weight group
```

Any remaining elements that do not form a complete group of eight are processed with the scalar loop.

#### Lines 64–109 — `macsOnRange_mac8im_conv2()`

Adds the Conv2-specific MAC8IM helper.

This path handles the memory organization used by Conv2. The second packed input load uses the layer-specific offset:

```text
24(%[p_in])
```

When the required input address is not suitable for the packed path, the function falls back to scalar MAC operations.

#### Lines 111–158 — `macsOnRange_mac8im_fc2()`

Adds the FC2-specific MAC8IM helper.

The function uses MAC8IM for groups of eight values when the checked address satisfies the required alignment condition. Otherwise, it preserves a scalar fallback.

#### Lines 196–345 — accelerated convolution path

Adds the MAC8IM-oriented convolution implementation that processes two kernel rows together.

The kernel loop advances with:

```text
sy += 2
```

and calls the Conv2-specific MAC8IM helper for the packed path.

The existing scalar `macsOnRange()` path remains available for cases that cannot use the packed contiguous access.

#### Line 450 — contiguous convolution MAC8IM call

The contiguous range uses:

```c
macsOnRange_mac8im_contiguous(...)
```

so groups of eight MAC operations are executed by the custom instruction.

#### Line 547 — fully connected MAC8IM call

The contiguous fully connected range is redirected to:

```c
macsOnRange_mac8im_contiguous(...)
```

This allows FC computation to reuse the same eight-way packed MAC helper.

#### Lines 631 and 648 — FC2 MAC8IM calls

FC2 uses:

```c
macsOnRange_mac8im_fc2(...)
```

for its accelerated ranges while preserving the FC2-specific fallback behavior.

#### Line 749 — Conv2 implementation selection

The network invokes:

```c
convcellPropagate2(...)
```

for Conv2 so that the MAC8IM-specific two-row convolution implementation is used.

---

### 13.2 `cv32a6_ima_sv32_fpga_config_pkg.sv`

#### Line 21 — enable CV-X-IF

The configuration sets:

```systemverilog
CVA6ConfigCvxifEn = 1
```

This enables the CV-X-IF interface used to send MAC8IM from CVA6 to the coprocessor.

---

### 13.3 `cva6.sv`

#### Line 164 — five GPR read ports

The integer register-file read-port configuration is set to:

```systemverilog
NrRgprPorts = 5
```

MAC8IM Method 2 needs five register values:

```text
rs1
rs2
old rd
x28
x29
```

---

### 13.4 `ariane_pkg.sv`

#### Line 75 — global GPR-port count

Sets:

```systemverilog
NR_RGPR_PORTS = 5
```

This keeps the CVA6 package configuration consistent with the five-source MAC8IM operand path.

#### Lines 446–448 — `MAC8IM` operation

Adds `MAC8IM` to the CPU operation enumeration.

This operation identifier is used after decode to distinguish the custom instruction inside the pipeline.

#### Lines 578–580 — `operand_d` and `operand_e`

Extends `fu_data_t` with two additional operands.

The five CV-X-IF values are transported as:

```text
operand_a -> rs1
operand_b -> rs2
imm       -> old rd / accumulator
operand_d -> x28
operand_e -> x29
```

---

### 13.5 `decoder.sv`

#### Lines 1189–1203 — MAC8IM instruction decode

Adds the custom instruction decode for:

```text
opcode = 0001011
funct3 = 001
funct7 = 0000000
```

The instruction is assigned to:

```systemverilog
fu = CVXIF
op = MAC8IM
```

The decoder extracts `rs1`, `rs2`, and `rd`. The `rd` value is later read as the initial accumulator as well as being the architectural destination.

Unsupported values in this custom decode path remain illegal.

---

### 13.6 `issue_read_operands.sv`

This is one of the main CPU-side MAC8IM modifications.

#### Lines 43–50 — rs4 and rs5 interface

Adds the additional register-source interfaces:

```text
rs4
rs5
```

These ports are used for the fixed x28 and x29 operands.

#### Lines 97–103 — additional operand storage

Adds storage for:

```text
operand_d
operand_e
```

These values carry x28 and x29 toward CV-X-IF.

#### Lines 123–124 — forwarding state

Adds:

```text
forward_rs4
forward_rs5
```

so the two additional source registers can use the same dependency/forwarding mechanism as normal operands.

#### Lines 136–139 — execution-data output

Connects the additional operand state into `fu_data_o`.

This allows the values to reach `cvxif_fu.sv`.

#### Lines 186–201 — MAC8IM register mapping

For MAC8IM, the register sources are configured as:

```text
rs3 = rd
rs4 = x28
rs5 = x29
```

`rs3` supplies the old destination value as the accumulator.

x28 and x29 provide the second packed input/weight pair.

For instructions that do not use these extra operands, rs4 and rs5 are mapped to x0.

#### Lines 243–246 — accumulator dependency check

Extends the third-source dependency logic so MAC8IM checks whether the old `rd` value is available before issue.

This prevents MAC8IM from using a stale accumulator.

#### Lines 254–268 — x28/x29 dependency handling

Adds dependency checks for rs4 and rs5.

If x28 or x29 is waiting for a previous instruction, the value is forwarded when available. Otherwise, issue stalls until the correct value is ready.

#### Lines 279–287 — accumulator register-file path

Allows the five-port configuration to read the third GPR source and use it as the MAC8IM accumulator.

#### Lines 308–314 — rs4/rs5 forwarding selection

Adds forwarding selection for the x28 and x29 operand paths.

#### Lines 494–505 — five-port register address packing

Defines the five register-file read addresses.

For MAC8IM, the packed order is:

```text
{x29, x28, rd, rs2, rs1}
```

Therefore:

```text
rdata[0] -> rs1
rdata[1] -> rs2
rdata[2] -> old rd
rdata[3] -> x28
rdata[4] -> x29
```

#### Line 600 — five-port `operand_c` configuration

Enables the GPR-based third operand for the five-port MAC8IM configuration.

#### Lines 613–617 — register-file output mapping

Maps the additional register-file outputs to:

```text
operand_d_regfile
operand_e_regfile
```

#### Lines 627–630 — reset of additional operand registers

Resets the x28/x29 operand pipeline state.

#### Lines 641–644 — pipeline update

Registers the next x28/x29 operand values into the pipeline state.

#### Line 657 — supported port-count assertion

Extends the register-port configuration assertion to accept the five-port implementation.

---

### 13.7 `issue_stage.sv`

#### Line 100 — third-source width

Uses the five-port configuration when selecting the third-source register width.

This allows the MAC8IM accumulator value read from `rd` to remain XLEN-wide.

#### Lines 116–124 — rs4/rs5 signal declarations

Adds the address, data, and valid signals for the two extra source registers.

#### Lines 162–170 — scoreboard connections

Connects rs4 and rs5 to the scoreboard.

This allows x28/x29 dependencies to be detected and forwarded.

#### Lines 214–221 — read-operands connections

Connects rs4 and rs5 between the issue stage and `issue_read_operands.sv`.

---

### 13.8 `scoreboard.sv`

#### Lines 43–51 — rs4/rs5 scoreboard ports

Adds address, data, and valid interfaces for rs4 and rs5.

#### Lines 318–321 — forwarding request state

Adds forwarding request and valid state for the two additional source registers.

#### Lines 334–338 — writeback dependency detection

Checks whether an active writeback port contains a pending value required by rs4 or rs5.

#### Lines 351–355 — in-flight dependency detection

Checks scoreboard entries that are still in the pipeline for pending writes to x28 or x29.

#### Lines 366–372 — source-valid generation

Adds valid generation for rs4 and rs5 and uses the five-port configuration for the third source.

#### Lines 434–472 — rs4/rs5 forwarding arbiters

Adds:

```text
i_sel_rs4
i_sel_rs5
```

These arbiters select the newest available forwarded values for x28 and x29.

This is required because x28/x29 are real architectural registers and can have normal RAW dependencies.

---

### 13.9 `cvxif_fu.sv`

#### Lines 43–44 — five valid source operands

The CV-X-IF source-valid vector is extended to the five-source configuration:

```text
11111
```

#### Lines 60–63 — CV-X-IF operand packing

The five values are sent as:

```text
rs[0] = operand_a -> rs1
rs[1] = operand_b -> rs2
rs[2] = imm       -> old rd / accumulator
rs[3] = operand_d -> x28
rs[4] = operand_e -> x29
```

This is the final CPU-side operand mapping before the instruction reaches the coprocessor.

---

### 13.10 `cvxif_pkg.sv`

#### Line 15 — CV-X-IF source count

Defines:

```systemverilog
X_NUM_RS = ariane_pkg::NR_RGPR_PORTS
```

Because `NR_RGPR_PORTS` is five in this implementation, CV-X-IF carries five source-register values.

---

### 13.11 `cvxif_instr_pkg.sv`

#### Line 19 — instruction-table size

The supported CV-X-IF instruction count is increased to:

```text
3
```

#### Lines 44–57 — MAC8IM instruction entry

Adds the MAC8IM instruction pattern to the CV-X-IF coprocessor instruction table.

The entry accepts the instruction and enables result writeback.

The MAC8IM pattern corresponds to the custom instruction encoding used by the CPU decoder and toolchain.

---

### 13.12 `cvxif_example_coprocessor.sv`

#### Lines 145–170 — MAC8IM arithmetic datapath

This is the main MAC8IM arithmetic implementation.

The coprocessor receives five 32-bit operands:

```text
rs[0] -> first packed input
rs[1] -> first packed weights
rs[2] -> accumulator
rs[3] -> second packed input
rs[4] -> second packed weights
```

The two packed input words and two packed weight words are split into eight byte lanes.

The datapath computes eight products:

```text
p0
p1
p2
p3
p4
p5
p6
p7
```

and adds all eight products to the accumulator value.

Input bytes are extended so they retain the intended unsigned input interpretation, while the weight bytes use signed arithmetic.

The final 32-bit MAC result is returned through CV-X-IF.

The result-valid behavior is also tied to the availability of a request in the result FIFO rather than the artificial example delay used by the original demonstration coprocessor.

---

### 13.13 `rv_i`

#### Lines 27–29 — MAC8IM opcode definition

Defines the custom instruction mnemonic:

```text
mac8im
```

with the MAC8IM custom encoding.

This is the instruction definition used by the RISC-V opcode-generation/toolchain flow.

---

### 13.14 `riscv-opc.h`

#### Lines 24–26 — match and mask

Defines:

```text
MATCH_MAC8IM = 0x100b
MASK_MAC8IM  = 0xfe00707f
```

These constants identify the MAC8IM encoding.

#### Lines 2789–2790 — instruction declaration

Registers:

```c
DECLARE_INSN(mac8im, ...)
```

so the instruction is available to the GNU RISC-V opcode infrastructure.

---

### 13.15 `riscv-opc.c`

#### Lines 322–324 — assembler opcode-table entry

Adds the `mac8im` mnemonic to the RISC-V assembler opcode table.

The visible assembly syntax is:

```text
mac8im rd, rs1, rs2
```

The second packed input/weight pair is not written in the mnemonic because Method 2 uses the fixed x28/x29 convention.

---

### 13.16 `instr_decoder.sv`

No MAC8IM-specific modification is required in this file in the delivered cleaned source set.

The MAC8IM custom decode used by this implementation is handled in `decoder.sv`.

---

## 14. MAC8IM Register Mapping Summary

The complete five-source mapping is:

```text
CPU register        Pipeline field        CV-X-IF source       Meaning
-----------         --------------        -------------        -------
rs1                 operand_a             rs[0]                input 0–3
rs2                 operand_b             rs[1]                weight 0–3
old rd              imm / operand_c       rs[2]                accumulator
x28 / t3            operand_d             rs[3]                input 4–7
x29 / t4            operand_e             rs[4]                weight 4–7
```

The result returns through CV-X-IF and is written to `rd`.

---

## 15. Files to Read First

For a new developer, the recommended order is:

```text
1. NetworkPropagate.c
   -> understand how software prepares the packed operands

2. decoder.sv
   -> understand how MAC8IM is recognized

3. issue_read_operands.sv
   -> understand rd/x28/x29 mapping

4. scoreboard.sv
   -> understand dependencies and forwarding

5. cvxif_fu.sv
   -> understand the five CV-X-IF source values

6. cvxif_example_coprocessor.sv
   -> understand the eight-way MAC arithmetic

7. rv_i / riscv-opc.h / riscv-opc.c
   -> understand assembler support
```

---

## 16. Verification Note

The `cleaned_code/` directory contains the source files relevant to MAC8IM Method 2 rather than an independent complete CVA6 repository.

After integrating them into the complete AIRV project, perform a clean build and run the same MNIST test used for the working MAC8IM Method 2 version.

Check:

```text
MNIST output
predicted class
cycle count
instruction count
five CV-X-IF operands
eight MAC products
accumulator
result
CPU writeback
```

If the functional result differs, use the debugging sequence in Section 11 before changing the arithmetic implementation.
