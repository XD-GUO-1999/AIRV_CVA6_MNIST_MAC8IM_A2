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
-   Questa RTL simulation support

## Documentation

For the MAC8IM architecture, instruction flow, register mapping, CV-X-IF
integration, and a line-by-line description of every modified source
file relative to the baseline:

👉 [User and Implementation Guide](USER_GUIDE_MAC8IM_METHOD2.md)

The guide compares the **baseline implementation directly with the final
cleaned MAC8IM Method 2 implementation**. Intermediate development
versions are intentionally not documented.

## Source Code

The cleaned MAC8IM Method 2 source files are located in:

``` text
cleaned_code/
```

The main implementation path is:

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
MAC8IM Coprocessor
        ↓
Result / Writeback
```

## MAC8IM Operand Mapping

The MAC8IM instruction performs eight packed INT8 multiply-accumulate
operations.

The five source values are organized as:

``` text
rs1      : packed input values 0–3
rs2      : packed weight values 0–3
old rd   : accumulator
x28 / t3 : packed input values 4–7
x29 / t4 : packed weight values 4–7
```

The result is written back to `rd`.

## Main Modified Components

The implementation modifies the following parts of the baseline system:

-   CNN software implementation and MAC8IM inline assembly
-   GNU assembler instruction definition
-   CVA6 custom instruction decoder
-   GPR read-port configuration
-   Issue and operand-read path
-   Scoreboard dependency checking
-   Forwarding logic
-   CV-X-IF functional unit interface
-   CV-X-IF instruction recognition
-   Example coprocessor MAC datapath

For the exact baseline and cleaned line numbers associated with each
modification, see `USER_GUIDE_MAC8IM_METHOD2.md`.

## Notes

This package contains the cleaned source files relevant to MAC8IM Method
2. It is intended to be integrated into the same complete AIRV/CVA6
project revision used by the original implementation.

After replacing the corresponding files, rebuild the project and run the
same MNIST simulation to verify identical functional results.
