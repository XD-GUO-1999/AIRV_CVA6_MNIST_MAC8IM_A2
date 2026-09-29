# AIRV CVA6 MAC8IM Accelerator

Hardware/software co-design for accelerating a quantized MNIST CNN on
the CVA6 RISC-V processor.

This implementation includes:

-   Custom RISC-V instruction: `MAC8IM`
-   8-way INT8 MAC acceleration
-   Five-operand register read path
-   `rd`-based accumulation
-   Fixed `x28` / `x29` implicit operands for the second packed
    input/weight pair
-   Extended scoreboard dependency checking and forwarding
-   CV-X-IF coprocessor integration
-   Modified GNU assembler/toolchain
-   Questa RTL simulation
-   Zybo Z7-20 FPGA execution

## Documentation

For environment setup, MNIST compilation, Questa simulation, FPGA
execution, MAC8IM architecture, toolchain modifications, debugging,
validation, and a line-by-line description of every modified source file
relative to the Baseline:

👉 [User and Implementation Guide](USER_GUIDE_MAC8IM_METHOD2.md)

The implementation guide compares the **Baseline directly with the final
cleaned MAC8IM Method 2 source code**. Intermediate development versions
are intentionally excluded.

## Source Code

The cleaned MAC8IM Method 2 source files are located in:

``` text
cleaned_code/
```

## Accelerator Overview

``` text
NetworkPropagate.c
        ↓
MAC8IM custom instruction
        ↓
CVA6 decoder
        ↓
Issue / Read Operands
        ↓
Scoreboard / Forwarding
        ↓
CV-X-IF
        ↓
8-way INT8 MAC coprocessor
        ↓
Result / Writeback
```

The MAC8IM operand mapping is:

``` text
rs1      -> packed input values 0–3
rs2      -> packed weight values 0–3
old rd   -> accumulator
x28 / t3 -> packed input values 4–7
x29 / t4 -> packed weight values 4–7
```

The result is written back to `rd`.

For exact file names, Baseline line numbers, cleaned line numbers, and
the purpose of every modification, see `USER_GUIDE_MAC8IM_METHOD2.md`.
