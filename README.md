# VLSI-Course-Project-EE-617-
We implemented the research paper on CICAM operation named "CICAM: A Reconfigurable SRAM-Based Computing-in-Content-Addressable Memory Architecture Using Charge Sharing" on Cadence Virtuoso at 28nm node CMOS Technology.
# 4×4 CICAM Array - Vector File Instructions

Vector (`.vec`) stimulus files for the 4×4 CICAM (Computing-in-Content-Addressable Memory) array built in Cadence Virtuoso (`vlsi_project` → `4x4_CICAM`).

**Authors:** Ananya Anilkumar Pawar (24110037), Hari Nisanth S (24110129)
**Instructor:** Prof. Joycee Mekie


## Files at a glance (path for all is /home/hari)

| File | Operation | MODE | S2 S1 S0 | Enable to pulse | Expected result |
|---|---|---|---|---|---|
| `write_read_0101_4_4x4.vec` | Write 0101 to Row 0, then read Row 0 | 0 | 000 (READ), set in schematic | `eno` | r0 r1 r2 r3 = **0 1 0 1** |
| `nor_cim_4x4_6_final.vec` | CIM NOR of Row 0 and Row 1 | 0 | 010 (NOR), in vector file | `eno` | r0 r1 r2 r3 = **1 0 0 0** |
| `cam_4x4_10_final.vec` | CAM search for 0101 | 1 | not used | `camen` | cam0 = **1** (match) |
| `cim_cam_ananya_version_5.vec` | CIM NOR, then switch to CAM search | 0 → 1 | 010 (NOR), in vector file | `eno`, then `camen` | r = **1000**, then cam0 cam1 cam2 cam3 = **1 0 1 0** |

---

## Output signals – what to plot

| Signal | Meaning |
|---|---|
| `r0 r1 r2 r3` | **CIM / read outputs.** Final digital result from the output circuit (sense amplifiers to logic → 8:1 MUX) of columns 0–3. These are only valid while `eno (enable)` is high. |
| `cam0 cam1 cam2 cam3` | **CAM outputs.** Match result of rows 0–3, sensed from ML0–ML3 by the CAM sense amplifiers. 1 = match (ML stayed high), 0 = mismatch (ML discharged). These are only valid while `camen (cam enable) ` is high. |
| `ML0–ML3` | Match lines of rows 0–3. Precharged high, then left floating during a search. |
| `RBL0–3`, `RBLB0–3`, `CRL0–3` | Analog bitline and charge-redistribution levels (V<sub>DD</sub>, ⅔V<sub>DD</sub>, ½V<sub>DD</sub>) during CIM. Useful for debugging; the digital answer is on r0–r3. |
| `a0–a3`, `b0–b3` | Internal storage nodes (Q) of Row 0 and Row 1. Prefer to probe these to confirm a write actually happened. |

Bit order: index 0 is column 0, which is the **first** value of each 4-bit group in the vector file. For example, `BL0 BL1 BL2 BL3 = 0 1 0 1` stores the word 0101, and `r0` is the result for that leftmost bit.

---

## Signals that are NOT in the vector files (set these in the schematic)

| Signal | Value / source | Notes |
|---|---|---|
| `eno` | Pulse source (0 V → 0.9 V) | Enable for the output-circuit sense amplifiers. **See the timing rule below.** |
| `camen` | Pulse source (0 V → 0.9 V) | Enable for the CAM sense amplifiers on ML0–ML3. **See the timing rule below.** |
| `Vref` | DC 0.75 V | SA reference; must sit between ⅔V<sub>DD</sub> and V<sub>DD</sub>. |
| `S2 S1 S0` | DC sources, **only for `write_read_0101_4_4x4.vec`** | Set to 0 0 0 (READ). The other CIM files already drive S2 S1 S0 from the vector file. |
| `vdd`, `gnd` | Driven by the vector file (vih = 0.9 V) | Do **not** also put a DC source on Vdd, or Spectre gives error as:  *SPECTRE-620: Node Vdd driven by multiple voltage sources*

---

## Enable timing rule (eno / camen)

The sense amplifier **precharges its outputs when the enable is 0 and evaluates when it is 1**. It latches whatever is on its inputs at the moment the enable rises. Therefore:

1. ** Ensure that `eno` / `camen` is ON only AFTER the read / compute / search has developed**: after RWL (and RWL_CIM) are asserted and the bitlines or match lines have settled. If the enable rises too early, the SA latches the precharged (all-high) lines and gives a wrong result.
2. **Keep the enable LOW during write and precharge phases.**
3. **Read r0–r3 / cam0–cam3 only while the enable is high.** When it is low, the outputs just show the SA precharge value.
4. Keep the enable pulse inside the evaluation window, and turn it off before the next precharge.

Suggested pulse windows for the files below (adjust the `delay` and `width` of the pulse source):

| File | Evaluation window in vectors | `eno` high | `camen` high |
|---|---|---|---|
| `write_read_0101_4_4x4.vec` | RWL0 on from 2.0 ns | ~2.4 ns → end | — |
| `nor_cim_4x4_6_final.vec` | 3.2 – 6.2 ns | ~3.6 – 6.0 ns | — |
| `cam_4x4_10_final.vec` | 2.7 – 5.7 ns | — | ~3.1 – 5.5 ns |
| `cim_cam_ananya_version_5.vec` | CIM 3.2 – 6.2 ns, CAM 12.5 – 13.5 ns | ~3.6 – 6.0 ns | ~12.9 – 13.5 ns |

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

| S2 S1 S0 | Operation | Result (A = 0101, B = 0011) |
|---|---|---|
| 000 | READ | A |
| 001 | NOT | Ā |
| 010 | NOR | 1000 |
| 011 | OR | 0111 |
| 100 | NAND | 1110 |
| 101 | AND | 0001 |
| 110 | XOR | 0110 |
| 111 | XNOR | 1001 |

## Per-file instructions

### 1. `write_read_0101_4_4x4.vec` – write and read

| Time (ns) | Phase |
|---|---|
| 0 | Everything off, MODE = 0 |
| 0.2 | Write Row 0 = 0101 (WL0 = 1, BL = 0101, BLB = 1010) |
| 0.7 | Release bitlines |
| 1.0 | Precharge RBL = RBLB = 1, CRL = 0 |
| 2.0 – 2.2 | Read Row 0 (RWL0 = RWL_CIM0 = 1) |

- Set **S2 S1 S0 = 000** with DC sources in the schematic, because they are not in this vector file.
- **Check:** a0–a3 = 0101 after the write; **r0 r1 r2 r3 = 0 1 0 1** while `eno` is high.

### 2. `nor_cim_4x4_6_final.vec` – CIM NOR

| Time (ns) | Phase |
|---|---|
| 0.2 – 0.8 | Write A = 0101 into Row 0 |
| 1.0 – 1.6 | Write B = 0011 into Row 1 |
| 2.0 – 3.0 | Precharge RBL/RBLB = 1, CRL = 0, then release |
| 3.2 – 6.2 | CIM: RWL0 = RWL1 = RWL_CIM0 = RWL_CIM1 = 1 |
| 7.0 | Hold |

- MODE = 0, and S2 S1 S0 = 010 (NOR) is already in the vectors.
- **Check:** **r0 r1 r2 r3 = 1 0 0 0**. RBL/RBLB/CRL should show the V<sub>DD</sub> / ⅔ / ½ levels per column.
### 3. `cam_4x4_10_final.vec` – CAM search

| Time (ns) | Phase |
|---|---|
| 0.2 – 0.7 | Write Row 0 = 0101 |
| 1.5 | Apply search data: RBL = 0101, RBLB = 1010 |
| 2.5 | Precharge ML0–ML3 = 1 |
| 2.7 – 5.7 | Search: RWL0–3 = 1, RWL_CIM = 0, ML released (Z) |
| 6.5 | Hold |

- MODE = 1 throughout.
- **Check:** ML0 stays high and **cam0 = 1** (Row 0 matches 0101).
- cam1–cam3 depend on whatever Rows 1–3 power up holding. They are usually 0000, which is a mismatch, so ML discharges and the output is 0.
- in the next case of cim+cam we have also taken up the case of 2 matches and 2 mismatches where as expected, ML line changes/retains behaviour. (refer to ppt for the same)

### 4. `cim_cam_ananya_version_5.vec` – mode switching (CIM → CAM)

| Time (ns) | Phase |
|---|---|
| 0.2 – 1.6 | Write Row 0 = 0101, Row 1 = 0011 |
| 2.0 – 3.0 | Precharge |
| 3.2 – 6.2 | CIM NOR of Row 0 and Row 1 (MODE = 0) |
| 7.5 – 9.2 | Write Row 2 = 0101, Row 3 = 1011 |
| 10.5 | **MODE = 1**; search data RBL = 0101, RBLB = 1010 |
| 11.5 | Precharge ML0–ML3 |
| 12.5 – 13.5 | Search all rows (RWL0–3 = 1, RWL_CIM = 0) |
| 14.5 | Hold |

- **Check CIM:** **r0 r1 r2 r3 = 1 0 0 0**.
- **Check CAM** (search = 0101 against Rows 0101 / 0011 / 0101 / 1011): **cam0 = 1, cam1 = 0, cam2 = 1, cam3 = 0**. ML1 and ML3 discharge.
---

## Common mistakes / checklist

- [ ] There is no second DC source on Vdd.
- [ ] `eno` / `camen` rise **after** evaluation starts, not before.
- [ ] S2 S1 S0 are set in the schematic for the write/read file if not in vec file.
- [ ] The bitlines are precharged (RBL = RBLB = 1, CRL = 0) before every read or CIM operation.
- [ ] The match lines are precharged, then released to Z, before every CAM search.
- [ ] Turn WL off before releasing BL/BLB after a write.
