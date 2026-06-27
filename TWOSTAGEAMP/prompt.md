# AI4EDA Automatic Optimization Prompt

## Part 1: Workflow Description

Use the files under `/ylab/data/TS65/xuzhen/ZHEN_AI4EDA` only.

Step 1:
Read the Spectre command from `INPUT/cmd.txt`.

Step 2:
Execute the Spectre simulation command exactly as written.

Step 3:
Store all simulation results in `OUTPUT/`.

Step 4:
Run `INPUT/metric_extraction.ocn` after Spectre finishes.

Step 5:
Generate `OUTPUT/performance.txt` and `OUTPUT/performance.csv`.

Step 6:
Before reading or accepting performance metrics, read the operating region of
all MOS devices M0-M7 from `OUTPUT/performance.txt` or `OUTPUT/performance.csv`.
Every MOS device must satisfy:

```text
region = 2
```

If any MOS device reports `region != 2`, `EVAL_ERR`, or `nan`, treat the
current simulation as an invalid operating point. Do not accept the performance
metrics for target satisfaction until all MOS devices have `region = 2`.

Step 7:
Read the extracted performance metrics from `OUTPUT/performance.txt` or
`OUTPUT/performance.csv`.

Step 8:
Compare the current performance against the target performance.

Step 9:
If any target is not satisfied, modify only tunable sizing and bias parameters in `INPUT/input.scs`.

Step 10:
Run the simulation and metric extraction again.

Step 11:
Repeat the iteration until all performance targets are satisfied or until 10 optimization iterations have been completed.

Iteration loop:

```text
simulate -> extract metric -> compare -> modify parameters -> simulate again
```

Stop condition:

```text
All MOS devices have region = 2 and all performance targets are satisfied,
or optimization iteration 10 completed
```

## Part 1.1: Optimization Rules

Rule 1:
The maximum number of optimization iterations is 10.

```text
Run no more than 10 optimization iterations.
If all targets are satisfied before iteration 10, stop early.
If the targets are still not satisfied after iteration 10, stop and summarize the remaining gaps.
```

Rule 2:
After every optimization iteration, append a bilingual Chinese-English record to `OUTPUT/optimization.txt`.

Each record must include:

```text
Iteration number / 优化次数
Current performance / 当前性能
Operating region of each MOS device / 每个 MOS 管的工作区
Distance to each target / 每个指标距离目标性能的差距
Whether all target performances are met / 是否达到所有目标性能
Whether all MOS devices are in saturation region=2 / 是否所有 MOS 管均处于 region=2 饱和区
Parameter modifications made in this iteration / 本次优化做出的参数修改
Reasoning basis for each modification / 每项修改的依据
```

The record must be written in both Chinese and English. Use append mode so previous iteration records are preserved.

Recommended format:

```text
Iteration N / 第 N 次优化

Current Performance / 当前性能:
DC Gain = ...
GBW = ...
Phase Margin = ...
Supply Current = ...
Power = ...

MOS Operating Regions / MOS 工作区:
M0_region = ...
M1_region = ...
M2_region = ...
M3_region = ...
M4_region = ...
M5_region = ...
M6_region = ...
M7_region = ...
All MOS region=2: Yes/No
是否所有 MOS 管 region=2：是/否

Target Distance / 目标差距:
DC Gain gap = ...
GBW gap = ...
Phase Margin gap = ...
Supply Current gap = ...
Power gap = ...

Target Status / 目标状态:
All targets met: Yes/No
是否达到所有目标性能：是/否

Modification Basis / 修改依据:
English: ...
中文：...

Parameter Changes / 参数修改:
English: ...
中文：...
```

Rule 3:
Do not generate plot images during optimization. Instead, generate one CSV file
that MATLAB can read directly after optimization stops.

CSV requirements:

```text
Filename: OUTPUT/optimization_history.csv

Each row = one optimization iteration.
Columns:
iteration
DC_gain
GBW
PhaseMargin
SupplyCurrent
Power
M0_region
M1_region
M2_region
M3_region
M4_region
M5_region
M6_region
M7_region
AllMOSRegion2
AllTargetsMet
```

Optional transfer:

```text
If the Windows local path is mounted and writable from the VM, also copy
OUTPUT/optimization_history.csv to:
C:\Users\xuzhe\Documents\MATLAB\AI4EDA\TWO_STAGE_AMP

If that Windows path is not mounted or not writable, keep the CSV in OUTPUT/
and report that direct transfer is unavailable from the VM.
```

## Part 2: Parameter Search Space

The following tunable parameters were automatically parsed from the `parameters` statement in `INPUT/input.scs`:

```text
parameters C=1p IBIAS=10u L=1u M5=1 M6=1 M7=1 MINPUT=1 MINPUT2=1 MLOAD=1 R=10k WBIAS=2u WINPUT=1u WINPUT2=2u WLOAD=1u
```

Device mapping:

```text
L        : common MOS channel length for M0-M7
WLOAD    : PMOS load width for M0 and M1
WINPUT2  : second-stage PMOS width for M2
WINPUT   : NMOS input-pair width for M3 and M4
WBIAS    : NMOS bias/output width for M5, M6, and M7
MLOAD    : multiplicity for M0 and M1
MINPUT2  : multiplicity for M2
MINPUT   : multiplicity for M3 and M4
M5/M6/M7 : multiplicity for M5, M6, and M7
C        : compensation capacitor C0
R        : compensation resistor R0
IBIAS    : bias current source I0
```

Circuit device roles:

```text
M3 and M4:
NMOS differential input pair. M3 receives VIN-, and M4 receives VIN+.
They convert differential input voltage into first-stage signal current.

M0 and M1:
PMOS active-load current mirror for the first stage. M1 is diode-connected and
sets the PMOS mirror gate voltage at net18. M0 mirrors current into net20,
forming a single-ended first-stage output node.

M5, M6, and M7:
NMOS bias current mirror group. M5 is diode-connected by the external IBIAS
source and generates the bias gate voltage net15. M6 mirrors this bias as the
tail current source of the input differential pair. M7 mirrors this bias as the
second-stage/output NMOS current source.

M2:
PMOS second-stage common-source gain transistor. Its gate is driven by the
first-stage output node net20, and it drives VOUT against the M7 current source.

C0 and R0:
Miller compensation network from the first-stage output node net20 to VOUT.
C0 sets the compensation capacitance, and R0 tunes the compensation zero to
improve phase margin.

I0:
External DC bias current source. It biases the M5/M6/M7 current mirror group.
```

Search Space:

```text
C       : [200f, 10p]
IBIAS   : [2u, 50u]
L       : [60n, 5u]
M5      : [1, 10]
M6      : [1, 10]
M7      : [1, 10]
MINPUT  : [1, 10]
MINPUT2 : [1, 10]
MLOAD   : [1, 10]
R       : [2k, 100k]
WBIAS   : [1u, 20u]
WINPUT  : [500n, 10u]
WINPUT2 : [1u, 20u]
WLOAD   : [500n, 10u]
```

Search guidance:

```text
Increase DC gain by increasing L, increasing load/input device width, or improving current mirror ratios.
Increase GBW by increasing transconductance with WINPUT or IBIAS, while watching current and phase margin.
Improve phase margin by tuning C and R first, then resizing the second stage and bias devices.
Reduce supply current and power by lowering IBIAS or multiplicities after gain, GBW, and phase margin are acceptable.
Keep multiplicity parameters as positive integers.
Keep L at or above 60n for the TSMC 65 nm process.
```

## Part 3: Target Performance

Default OTA targets:

```text
All MOS devices M0-M7 must have region = 2
DC Gain > 70 dB
GBW > 10 MHz
Phase Margin > 60 degree
Supply Current < 100 uA
Power < 150 uW
```

Use these targets as the acceptance criteria for automatic optimization. If a
metric or MOS operating region reports `EVAL_ERR` or `nan`, or if any MOS
device reports `region != 2`, treat the current simulation as failed and adjust
parameters conservatively before rerunning.
