# AIRV CVA6 MAC8IM Method 2 --- User Guide

This guide explains how to configure the AIRV/CVA6 environment, compile
the MNIST application, run RTL simulation with Questa, execute the
design on the Zybo Z7-20 FPGA platform, and understand the
implementation of the cleaned MAC8IM Method 2 accelerator.

The implementation described here is the **cleaned MAC8IM Method 2
version**. The implementation section compares this final cleaned
version directly with the supplied **Baseline**. The intermediate
uncleaned MAC8IM development version is intentionally not documented.

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

The `cleaned_code/` directory delivered with this guide contains the
source files that differ from, or are relevant to, the Baseline MAC8IM
Method 2 implementation.

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

## 13. Notes on Line Numbers

The implementation guide below compares:

``` text
Baseline
    ↓
Final Cleaned MAC8IM Method 2
```

It does **not** compare the intermediate uncleaned MAC8IM version with
the cleaned version.

All cleaned line numbers were calculated after the cleanup was finished.

For pure insertions, the guide uses:

``` text
After Baseline Lx -> Cleaned Ly-Lz
```

because the inserted code has no corresponding Baseline line.

If the source is reformatted later, line numbers can move. Use both the
line number and the described symbol/function as the anchor.

------------------------------------------------------------------------

# 14. Implementation Guide --- Baseline to Cleaned MAC8IM

## 14.1 Architecture summary

MAC8IM Method 2 executes the custom instruction through CV-X-IF.

Software flow:

`NetworkPropagate.c` → `mac8im` instruction → CVA6 decoder → five source
operands → scoreboard/forwarding → CV-X-IF → coprocessor MAC8 datapath →
result/writeback.

One MAC8IM operation is:

`rd = rd + Σ(i=0..3) rs1[i]×rs2[i] + Σ(i=0..3) x28[i]×x29[i]`

Each packed register contains four 8-bit values.

Register roles:

  Operand         Role
  --------------- -------------------
  `rs1`           input bytes 0--3
  `rs2`           weight bytes 0--3
  previous `rd`   accumulator
  `x28` / `t3`    input bytes 4--7
  `x29` / `t4`    weight bytes 4--7

This is why the implementation expands the GPR read path to five
operands.

------------------------------------------------------------------------

## 14.2 Detailed file-by-file modifications

### 14.2.1 `NetworkPropagate.c`

### MAC8IM software helpers

  ----------------------------------------------------------------------------
  Baseline            Cleaned MAC8IM      Modification
  ------------------- ------------------- ------------------------------------
  After L28           L31--62             Adds
                                          `macsOnRange_mac8im_contiguous()`.
                                          Processes eight MACs per iteration.
                                          `t3/t4` carry the first four packed
                                          values and `t1/t2` the second four;
                                          `mac8im` accumulates into `sum`.
                                          Remaining elements use the scalar
                                          loop.

  After L28           L64--109            Adds `macsOnRange_mac8im_conv2()`.
                                          Provides the Conv2-specific two-row
                                          access pattern. The second packed
                                          input uses offset `24(%[p_in])`;
                                          unaligned input falls back to eight
                                          scalar MACs.

  After L28           L111--158           Adds `macsOnRange_mac8im_fc2()`.
                                          Uses MAC8IM when the checked address
                                          is aligned and preserves a scalar
                                          eight-element fallback otherwise.
  ----------------------------------------------------------------------------

### Convolution integration

  ---------------------------------------------------------------------------------
  Baseline                 Cleaned MAC8IM      Modification
  ------------------------ ------------------- ------------------------------------
  L67--208                 L196--345           Adds a MAC8IM-oriented convolution
  (`convcellPropagate1`)                       loop that processes two kernel rows
                                               at a time (`sy += 2`) and calls
                                               `macsOnRange_mac8im_conv2()` for the
                                               contiguous case. Scalar
                                               `macsOnRange()` remains as the
                                               fallback for non-contiguous/wrapped
                                               accesses.

  Baseline `macsOnRange()` L450                In the other convolution path,
  call around L169                             replaces the contiguous scalar range
                                               call with
                                               `macsOnRange_mac8im_contiguous()`.

  L467                     L749                Changes the Conv2 network call from
                                               `convcellPropagate1(...)` to
                                               `convcellPropagate2(...)`, selecting
                                               the dedicated Conv2 implementation.
  ---------------------------------------------------------------------------------

### Fully connected integration

  ----------------------------------------------------------------------------
  Baseline            Cleaned MAC8IM      Modification
  ------------------- ------------------- ------------------------------------
  L265                L547                Replaces the contiguous FC range
                                          with
                                          `macsOnRange_mac8im_contiguous()`.

  L349                L631                Replaces the FC2 contiguous range
                                          with `macsOnRange_mac8im_fc2()`.

  L366                L648                Replaces the FC2 wrapped/per-line
                                          range with
                                          `macsOnRange_mac8im_fc2()`.
  ----------------------------------------------------------------------------

Whitespace-only end-of-file changes are not architecturally relevant.

------------------------------------------------------------------------

### 14.2.2 `cv32a6_ima_sv32_fpga_config_pkg.sv`

  Baseline   Cleaned MAC8IM   Modification
  ---------- ---------------- ----------------------------------------------
  L21        L21              `CVA6ConfigCvxifEn` changes from `0` to `1`.

**Purpose:** enables CV-X-IF so CVA6 can offload MAC8IM to the
coprocessor.

------------------------------------------------------------------------

### 14.2.3 `cva6.sv`

  Baseline   Cleaned MAC8IM   Modification
  ---------- ---------------- ----------------------------------------
  L164       L164             `NrRgprPorts` changes from `2` to `5`.

**Purpose:** MAC8IM needs `rs1`, `rs2`, old `rd`, x28 and x29 at the
same instruction interface.

------------------------------------------------------------------------

### 14.2.4 `ariane_pkg.sv`

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  L75                     L75                     `NR_RGPR_PORTS` changes
                                                  from `2` to `5`.

  After L445              L446--448               Adds `MAC8IM` to the
                                                  operation enumeration.

  After L574              L578--580               Adds `operand_d` and
                                                  `operand_e` to
                                                  `fu_data_t`.
  -----------------------------------------------------------------------

**Purpose:** defines MAC8IM as a CVA6 operation and provides storage for
the fourth and fifth source values.

------------------------------------------------------------------------

### 14.2.5 `decoder.sv`

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  After L1189             L1189--1203             Adds custom opcode
                                                  `7'b0001011`. Selects
                                                  `CVXIF`, extracts
                                                  `rs1`, `rs2`, and `rd`,
                                                  selects the
                                                  third-source path for
                                                  the accumulator,
                                                  recognizes
                                                  `funct3 == 3'b001`, and
                                                  assigns operation
                                                  `MAC8IM`. Other funct3
                                                  values are illegal.

  -----------------------------------------------------------------------

**Purpose:** this is the CPU-side instruction decode entry point for
MAC8IM.

------------------------------------------------------------------------

### 14.2.6 `issue_read_operands.sv`

This file contains the largest CPU microarchitecture change because it
turns the operand path into a five-source path.

### New rs4/rs5 interface and storage

  -----------------------------------------------------------------------
  Baseline              Cleaned MAC8IM        Modification
  --------------------- --------------------- ---------------------------
  After L42             L43--50               Adds
                                              `rs4_o/rs4_i/rs4_valid_i`
                                              and
                                              `rs5_o/rs5_i/rs5_valid_i`
                                              ports.

  L89--91               L97--103              Adds `operand_d_regfile`,
                                              `operand_e_regfile`, and
                                              their `_n/_q` state.

  After L110            L123--124             Adds `forward_rs4` and
                                              `forward_rs5`.

  After L121            L136--139             Routes `operand_d_q` and
                                              `operand_e_q` into
                                              `fu_data_o`.
  -----------------------------------------------------------------------

### Register mapping and hazard handling

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  L168--171               L186--201               Initializes rs4/rs5
                                                  forwarding; for MAC8IM
                                                  maps `rs3 = rd`,
                                                  `rs4 = x28`, and
                                                  `rs5 = x29`. Non-MAC8IM
                                                  instructions map
                                                  rs4/rs5 to x0.

  L213--214               L243--246               Extends the
                                                  third-source clobber
                                                  check to MAC8IM and
                                                  checks the old `rd`
                                                  value when MAC8IM is
                                                  issued.

  L222--225               L254--268               Adds dependency
                                                  handling for x28/x29.
                                                  If a source is
                                                  clobbered, it is
                                                  forwarded when valid;
                                                  otherwise issue stalls.

  L236--241               L279--287               Changes the
                                                  third-source GPR path
                                                  to the five-port
                                                  configuration and
                                                  allows MAC8IM to use
                                                  `operand_c_regfile` as
                                                  the accumulator value.

  After L261              L308--314               Adds forwarding mux
                                                  selection for rs4 and
                                                  rs5.
  -----------------------------------------------------------------------

### Register-file read ports

  --------------------------------------------------------------------------------
  Baseline           Cleaned MAC8IM     Modification
  ------------------ ------------------ ------------------------------------------
  L442--443          L494--505          Adds the five-port `raddr_pack`. MAC8IM
                                        read order is `{x29, x28, rd, rs2, rs1}`.
                                        A three-port fallback remains for the
                                        existing path.

  L538               L600               Changes the `operand_c` GPR generate
                                        condition to the five-port configuration.

  L551               L613--617          Changes `operand_c_regfile` selection for
                                        five ports and connects
                                        `rdata[3]`/`rdata[4]` to
                                        `operand_d_regfile`/`operand_e_regfile`.

  After L560         L627--630          Resets `operand_d_q` and `operand_e_q`.

  After L570         L641--644          Registers `operand_d_n` and `operand_e_n`.

  L583               L657               Extends the supported GPR-port assertion
                                        to include five ports.
  --------------------------------------------------------------------------------

**Result:** the execution data sent toward CV-X-IF now contains all five
MAC8IM values.

------------------------------------------------------------------------

### 14.2.7 `issue_stage.sv`

  ----------------------------------------------------------------------
  Baseline               Cleaned MAC8IM         Modification
  ---------------------- ---------------------- ------------------------
  L100                   L100                   Changes `rs3_len_t`
                                                selection from the
                                                three-port condition to
                                                the five-port condition
                                                so the MAC8IM
                                                accumulator uses XLEN
                                                width.

  After L115             L116--124              Adds rs4/rs5 address,
                                                data and valid
                                                interconnect signals.

  After L152             L162--170              Connects rs4/rs5 to the
                                                scoreboard instance.

  After L195             L214--221              Connects rs4/rs5 to
                                                `issue_read_operands`.
  ----------------------------------------------------------------------

**Purpose:** physically connects the new operand signals between Issue
Stage, Scoreboard and Read Operands.

------------------------------------------------------------------------

### 14.2.8 `scoreboard.sv`

### Interface and dependency requests

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  After L42               L43--51                 Adds rs4 and rs5
                                                  address/data/valid
                                                  ports.

  L309--311               L318--321               Extends forwarding
                                                  request and valid state
                                                  with `rs4_fwd_req`,
                                                  `rs5_fwd_req`,
                                                  `rs4_valid`, and
                                                  `rs5_valid`.

  After L323              L334--338               Checks writeback ports
                                                  for pending values
                                                  targeting rs4/x28 or
                                                  rs5/x29.

  After L335              L351--355               Checks in-flight
                                                  scoreboard entries for
                                                  rs4/rs5 dependencies.

  L346--349               L366--372               Uses the five-port
                                                  condition for rs3
                                                  validity and adds
                                                  rs4/rs5 validity
                                                  outputs.
  -----------------------------------------------------------------------

### Forwarding arbitration

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  After L410              L434--472               Adds two `rr_arb_tree`
                                                  instances, `i_sel_rs4`
                                                  and `i_sel_rs5`, to
                                                  select forwarded values
                                                  for the two new
                                                  sources.

  -----------------------------------------------------------------------

**Purpose:** x28 and x29 are architectural registers. MAC8IM therefore
needs normal RAW-dependency protection and forwarding, not only extra
physical read ports.

------------------------------------------------------------------------

### 14.2.9 `cvxif_fu.sv`

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  L43--44                 L43--44                 Changes the CV-X-IF
                                                  source-valid generation
                                                  from the 3-source case
                                                  to the 5-source case
                                                  and asserts `5'b11111`.

  L60--61                 L60--63                 In the five-source
                                                  case, sends `operand_d`
                                                  to `rs[3]` and
                                                  `operand_e` to `rs[4]`,
                                                  in addition to the
                                                  existing first three
                                                  operands.
  -----------------------------------------------------------------------

The resulting CV-X-IF mapping is:

-   `rs[0] = operand_a` → rs1
-   `rs[1] = operand_b` → rs2
-   `rs[2] = imm` → old rd / accumulator
-   `rs[3] = operand_d` → x28
-   `rs[4] = operand_e` → x29

------------------------------------------------------------------------

### 14.2.10 `cvxif_pkg.sv`

  --------------------------------------------------------------------------
  Baseline              Cleaned MAC8IM        Modification
  --------------------- --------------------- ------------------------------
  L15                   L15                   Keeps `X_NUM_RS` tied to
                                              `ariane_pkg::NR_RGPR_PORTS`;
                                              with the MAC8IM configuration
                                              this now evaluates to five.

  --------------------------------------------------------------------------

The functional effect comes from `NR_RGPR_PORTS = 5` in `ariane_pkg.sv`.

------------------------------------------------------------------------

### 14.2.11 `cvxif_instr_pkg.sv`

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  L19                     L19                     Increases `NbInstr`
                                                  from `2` to `3`.

  After L44               L44--57                 Adds the MAC8IM
                                                  custom-0 instruction
                                                  pattern. The response
                                                  accepts the instruction
                                                  and enables writeback;
                                                  dual-write, dual-read,
                                                  load/store and
                                                  exception flags remain
                                                  disabled.
  -----------------------------------------------------------------------

**Purpose:** tells the example CV-X-IF coprocessor that this custom
instruction is supported.

------------------------------------------------------------------------

### 14.2.12 `cvxif_example_coprocessor.sv`

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  L145--147               L145--170               Replaces the baseline
                                                  example
                                                  arithmetic/result logic
                                                  with the MAC8IM
                                                  datapath. Reads five
                                                  32-bit operands,
                                                  extracts eight byte
                                                  lanes, computes
                                                  `p0…p7`, adds them to
                                                  the accumulator,
                                                  returns `mac_result`,
                                                  and makes the result
                                                  valid whenever the
                                                  request FIFO is
                                                  non-empty.

  -----------------------------------------------------------------------

Important arithmetic behavior:

-   input bytes from rs1/x28 are explicitly prefixed with `0` before
    signed multiplication, preserving them as non-negative 8-bit input
    values;
-   weight bytes from rs2/x29 are interpreted as signed;
-   the previous `rd` value is added as `acc_val`.

------------------------------------------------------------------------

## 3.13 Toolchain instruction definition

### `rv_i`

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  L27--29                 L27--29                 Renames the custom
                                                  mnemonic from `mac4` to
                                                  `mac8im`. Encoding
                                                  remains `31..25=0`,
                                                  `funct3=1`, custom-0
                                                  opcode.

  -----------------------------------------------------------------------

### `riscv-opc.h`

  --------------------------------------------------------------------------
  Baseline              Cleaned MAC8IM        Modification
  --------------------- --------------------- ------------------------------
  L24--25               L24--26               Replaces
                                              `MATCH_MAC4/MASK_MAC4` with
                                              `MATCH_MAC8IM/MASK_MAC8IM`.
                                              Match remains `0x100b`; mask
                                              remains `0xfe00707f`.

  L2788                 L2789--2790           Replaces
                                              `DECLARE_INSN(mac4, ...)` with
                                              `DECLARE_INSN(mac8im, ...)`.
  --------------------------------------------------------------------------

### `riscv-opc.c`

  -----------------------------------------------------------------------
  Baseline                Cleaned MAC8IM          Modification
  ----------------------- ----------------------- -----------------------
  L322--324               L322--324               Replaces assembler
                                                  table entry `mac4` with
                                                  `mac8im`, using
                                                  `MATCH_MAC8IM` and
                                                  `MASK_MAC8IM`. Operand
                                                  syntax remains
                                                  `"d,s,t"`.

  -----------------------------------------------------------------------

**Result:** software can emit `mac8im rd, rs1, rs2`; x28/x29 remain
implicit operands supplied by the software convention.

------------------------------------------------------------------------

### 14.2.14 `instr_decoder.sv`

**No change.**

The Baseline and Cleaned MAC8IM copies are identical. This file is
included in the delivered code set for completeness but is not part of
the MAC8IM modification.

------------------------------------------------------------------------

## 15. End-to-End Instruction Flow --- Implementation View

1.  `NetworkPropagate.c` packs data into 32-bit words and loads the
    additional packed values into `t3`/`t4` (x28/x29).
2.  The assembler recognizes `mac8im` through `rv_i`, `riscv-opc.h`, and
    `riscv-opc.c`.
3.  `decoder.sv` recognizes opcode `0001011`, funct3 `001`, and sends
    the instruction to `CVXIF` as `MAC8IM`.
4.  `issue_read_operands.sv` requests five GPR values: rs1, rs2, old rd,
    x28 and x29.
5.  `scoreboard.sv` checks dependencies and can forward all additional
    values or stall when they are not ready.
6.  `issue_stage.sv` carries the rs4/rs5 signals between the blocks.
7.  `cvxif_fu.sv` sends all five values in `x_issue_req.rs[0..4]`.
8.  `cvxif_instr_pkg.sv` marks the MAC8IM pattern as accepted with
    writeback.
9.  `cvxif_example_coprocessor.sv` performs eight byte-wise
    multiplications and adds the previous rd accumulator.
10. The 32-bit result returns through CV-X-IF and is written back to
    `rd`.

------------------------------------------------------------------------

## 16. Files to Read First

For a new developer, the recommended order is:

1.  `NetworkPropagate.c` --- understand how software feeds MAC8IM.
2.  `decoder.sv` --- see how the custom instruction enters CV-X-IF.
3.  `issue_read_operands.sv` --- understand rd/x28/x29 mapping.
4.  `scoreboard.sv` --- understand dependency handling and forwarding.
5.  `cvxif_fu.sv` --- see the five-source CV-X-IF request.
6.  `cvxif_example_coprocessor.sv` --- see the actual eight-lane MAC.

------------------------------------------------------------------------

## 17. Verification Note

The supplied archives contain the 16 relevant files rather than a
complete standalone CVA6 build tree, so this guide does not claim a new
full regression run.

Before replacing the original experimental version, validate the cleaned
files in the same complete project used for MAC8IM Method 2:

1.  rebuild from clean;
2.  run the same MNIST simulation;
3.  compare final network outputs with the known MAC8IM Method 2 output;
4.  compare cycle/instruction counters;
5.  if a mismatch occurs, inspect issue operands, x28/x29 forwarding,
    CV-X-IF request operands, coprocessor result and CPU writeback in
    Questa.

The line numbers in this guide refer exactly to the delivered
`cleaned_code/` files.

------------------------------------------------------------------------

## 18. Recommended Repository Freeze

After confirming that the cleaned source, simulation result and this
guide all correspond to the same version, freeze the repository with a
Git commit/tag.

For example:

``` bash
git status
git add .
git commit -m "Add cleaned MAC8IM Method 2 implementation and documentation"
git tag mac8im-method2-clean-v1.0
```

A fixed tag is useful because the implementation section contains exact
line references.
