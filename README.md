# VLSI-Course-Project-EE-617-
We implemented the research paper on CICAM operation named "CICAM: A Reconfigurable SRAM-Based Computing-in-Content-Addressable Memory Architecture Using Charge Sharing" on Cadence Virtuoso at 28nm node CMOS Technology.
# 4×4 CICAM Array - Vector File Instructions


Note: Please refer to the PPT and video for any additional information. They provide a detailed overview of the key conditions, values, and simulations.

## Files at a glance (path for all is /home/hari)

| File | Operation | MODE | S2 S1 S0 | Enable to pulse | Expected result |
|---|---|---|---|---|---|
| `write_read_0101_4_4x4.vec` | Write 0101 to Row 0, then read Row 0 | 0 | 000 (READ), set in schematic | `eno` | r0 r1 r2 r3 = **0 1 0 1** |
| `nor_cim_4x4_6_final.vec` | CIM NOR of Row 0 and Row 1 | 0 | 010 (NOR), in vector file | `eno` | r0 r1 r2 r3 = **1 0 0 0** |
| `cam_4x4_10_final.vec` | CAM search for 0101 | 1 | not used | `camen` | cam0 = **1** (match) |
| `cim_cam_ananya_version_5.vec` | CIM NOR, then switch to CAM search | 0 → 1 | 010 (NOR), in vector file | `eno`, then `camen` | r = **1000**, then cam0 cam1 cam2 cam3 = **1 0 1 0** |
| `allselectlines.vec` | CIM on Row 0 (A = 0101) and Row 1 (B = 0011) for all 8 select codes, run one after another | 0 | 000 → 111, stepped in the vector file | `eno` (once per operation) | r0 r1 r2 r3 = **0001, 1110, 1000, 0111, 1110, 0001, 0110, 1001** |

---

## Output signals - what to plot

| Signal | Meaning |
|---|---|
| `r0 r1 r2 r3` | **CIM / read outputs.** Final digital result from the output circuit of columns 0-3. These are only valid while `eno (enable)` is high. |
| `cam0 cam1 cam2 cam3` | **CAM outputs.** Match result of rows 0-3, sensed from ML0-ML3 by the CAM sense amplifiers. 1 = match (ML stayed high), 0 = mismatch (ML discharged). These are only valid while `camen (cam enable) ` is high. |
| `ML0-ML3` | Match lines of rows 0-3. Precharged high, then left floating during a search. |
| `RBL0-3`, `RBLB0-3`, `CRL0-3` | Analog bitline and charge-redistribution levels (V<sub>DD</sub>, ⅔V<sub>DD</sub>, ½V<sub>DD</sub>) during CIM. Useful for debugging; the digital answer is on r0-r3. |
| `a0-a3`, `b0-b3` | Internal storage nodes (Q) of Row 0 and Row 1. Prefer to probe these to confirm a write actually happened. |

Bit order: index 0 is column 0, which is the **first** value of each 4-bit group in the vector file. For example, `BL0 BL1 BL2 BL3 = 0 1 0 1` stores the word 0101, and `r0` is the result for that leftmost bit.

---

## Signals that are NOT in the vector files (to be set in the schematic while simulating)

| Signal | Value / source | Notes |
|---|---|---|
| `eno` | Pulse source (0 V → 0.9 V) | Enable for the output-circuit sense amplifiers. **See the timing rule below.** |
| `camen` | Pulse source (0 V → 0.9 V) | Enable for the CAM sense amplifiers on ML0-ML3. **See the timing rule below.** |
| `S2 S1 S0` | DC sources, **only for `write_read_0101_4_4x4.vec`** | Set to 0 0 0 (READ). The other CIM files already drive S2 S1 S0 from the vector file. |

---

## Enable timing rule (eno / camen)

The sense amplifier **precharges its outputs when the enable is 0 and evaluates when it is 1**. It latches whatever is on its inputs at the moment the enable rises. Therefore:

1. Ensure that `eno` / `camen` is ON only AFTER the read / compute / search has developed: after RWL (and RWL_CIM) are asserted and the bitlines or match lines have settled. If the enable rises too early, the SA latches the precharged (all-high) lines and gives a wrong result.
2. **Keep the enable LOW during write and precharge phases.**
3. **Read r0-r3 / cam0-cam3 only while the enable is high.** When it is low, the outputs just show the SA precharge value.
4. Keep the enable pulse inside the evaluation window, and turn it off before the next precharge.
---

## Operating modes (from the CICAM cell)

| Operation | MODE | Key signals |
|---|---|---|
| **Write** (6T style) | any | WL = 1. Write 1: BL = 1, BLB = 0. Write 0: BL = 0, BLB = 1. Afterwards turn off WL, then release BL/BLB (Z). |
| **Precharge** (before every read / CIM) | 0 | RBL = RBLB = 1, CRL = 0, then release to Z. |
| **Read** (one row) | 0 | RWL = RWL_CIM = 1 on that row. S2 S1 S0 = 000. |
| **CIM** (two rows) | 0 | RWL = RWL_CIM = 1 on **both** operand rows at once. S2 S1 S0 picks the logic function. |
| **CAM search** | 1 | RBL = search data (SL), RBLB = its complement (SLB). Precharge ML = 1, then release (Z). RWL = 1 and RWL_CIM = 0 on the searched rows. A mismatch discharges ML. |

`MODE` drives the RWL_CIM multiplexer: 0 = SRAM/CIM, 1 = CAM.

### Output-circuit select (S2 S1 S0)

| S2 S1 S0 | Operation 
|---|---|---|
| 000 | READ
| 001 | NOT
| 010 | NOR 
| 011 | OR 
| 100 | NAND
| 101 | AND 
| 110 | XOR 
| 111 | XNOR 
## Per-file instructions

### 1. `write_read_0101_4_4x4.vec` - write and read

| Step | Phase |
|---|---|
| 1 | Everything off, MODE = 0 |
| 2 | Write Row 0 = 0101 (WL0 = 1, BL = 0101, BLB = 1010) |
| 3 | Release bitlines |
| 4 | Precharge RBL = RBLB = 1, CRL = 0 |
| 5 | Read Row 0 (RWL0 = RWL_CIM0 = 1) |

- Set **S2 S1 S0 = 000** with DC sources in the schematic, because they are not in this vector file.
- **Check:** a0-a3 = 0101 after the write; **r0 r1 r2 r3 = 0 1 0 1** while `eno` is high.

### 2. `nor_cim_4x4_6_final.vec` - CIM NOR

| Step | Phase |
|---|---|
| 1 | Write A = 0101 into Row 0 |
| 2 | Write B = 0011 into Row 1 |
| 3 | Precharge RBL/RBLB = 1, CRL = 0, then release |
| 4 | CIM: RWL0 = RWL1 = RWL_CIM0 = RWL_CIM1 = 1 |
| 5 | Hold |

- MODE = 0, and S2 S1 S0 = 010 (NOR) is already in the vectors.
- **Check:** **r0 r1 r2 r3 = 1 0 0 0**. RBL/RBLB/CRL should show the V<sub>DD</sub> / ⅔ / ½ levels per column.

### 3. `cam_4x4_10_final.vec` - CAM search

| Step | Phase |
|---|---|
| 1 | Write Row 0 = 0101 |
| 2 | Apply search data: RBL = 0101, RBLB = 1010 |
| 3 | Precharge ML0-ML3 = 1 |
| 4 | Search: RWL0-3 = 1, RWL_CIM = 0, ML released (Z) |
| 5 | Hold |

- MODE = 1 throughout.
- **Check:** ML0 stays high and **cam0 = 1** (Row 0 matches 0101).
- cam1-cam3 depend on whatever Rows 1-3 power up holding. They are usually 0000, which is a mismatch, so ML discharges and the output is 0.
- The next case (CIM + CAM) also covers 2 matches and 2 mismatches; the ML lines discharge or hold as expected (refer to the ppt for the same).

### 4. `cim_cam_ananya_version_5.vec` - mode switching (CIM → CAM)

| Step | Phase |
|---|---|
| 1 | Write Row 0 = 0101, Row 1 = 0011 |
| 2 | Precharge |
| 3 | CIM NOR of Row 0 and Row 1 (MODE = 0) |
| 4 | Write Row 2 = 0101, Row 3 = 1011 |
| 5 | **MODE = 1**; search data RBL = 0101, RBLB = 1010 |
| 6 | Precharge ML0-ML3 |
| 7 | Search all rows (RWL0-3 = 1, RWL_CIM = 0) |
| 8 | Hold |

### 5. `allselectlines.vec` - all 8 select-line (S2 S1 S0) operations

| Step | Phase |
|---|---|
| 1 | Write A = 0101 into Row 0 |
| 2 | Write B = 0011 into Row 1 |
| 3 | Set S2 S1 S0 to the next code (000 → 111) |
| 4 | Precharge RBL = RBLB = 1, CRL = 0, then release |
| 5 | CIM: RWL0 = RWL1 = RWL_CIM0 = RWL_CIM1 = 1 |
| 6 | End compute; repeat steps 3-5 for all 8 codes |
| 7 | Hold |

\* READ and NOT are meant for a single active row. Here two rows are on, so the READ path (tapped from the AND output) gives A·B, and NOT gives its complement.

---

