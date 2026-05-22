# LSTM in Excel

This workbook (`LSTM.xlsx`) implements a **single time-step LSTM cell** forward pass in Excel. Each gate is on its own worksheet, with matrix multiply (`MMULT`), sigmoid, and tanh built from formulas—no Python or macros required.

---

## What is in the workbook

| Sheet | Range | Role |
|-------|--------|------|
| **Forget Gate** | A1:Q21 | Computes forget gate $f_t$ |
| **Input Gate** | A1:T28 | Input gate $i_t$, candidate $\tilde{C}_t$, cell state $C_t$ |
| **Cell State & Output** | A1:Q25 | Output gate $o_t$ and hidden state $h_t$ |

There is **no Back Propagation sheet** in the current file (only forward pass). You can add training/backprop later if needed.

---

## Architecture (one timestep)

```text
Inputs:  h_(t-1)  (3 values)  +  x_t  (3 values)  →  z = [h_(t-1); x_t]  (6 values)

Forget Gate:     f_t = σ(W_f · z + b_f)
Input Gate:      i_t = σ(W_i · z + b_i)
                 C̃_t = tanh(W_C · z + b_C)
                 C_t = f_t ⊙ C_(t-1) + i_t ⊙ C̃_t
Output:          o_t = σ(W_o · z + b_o)
                 h_t = o_t ⊙ tanh(C_t)
```

⊙ = element-wise multiply. The model has **3 LSTM units** (labeled Cell 1, Cell 2, Cell 3).

### Data flow between sheets

```text
Forget Gate  ──f_t, h_(t-1), x_t──►  Input Gate  ──C_t──►  Cell State & Output  ──►  h_t
```

- **Input Gate** reads $f_t$ from `'Forget Gate'!O5:O7` and reuses $h_{t-1}$, $x_t$ from Forget Gate.
- **Cell State & Output** reads $C_t$ from `'Input Gate'!T5:T7` and again uses $h_{t-1}$, $x_t$ from Forget Gate.

---

## Layout note

Each sheet title (e.g. **Forget Gate**) is in cell **I1**. Calculation blocks use columns **B–G** for $h_{t-1}$ / $x_t$ / weights and **L–T** (Input Gate) or **M–Q** (other sheets) for per-cell results.

---

## Sheet 1: Forget Gate

**Formula:**

$$f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f)$$

### Inputs (editable numbers)

| Label | Cells | Default values |
|--------|--------|----------------|
| $h_{t-1}$ | B4:D4 | 0.1, 1, 0.5 |
| $x_t$ | B6:D6 | 1.256, 0.2, 0.8 |

### Weights and bias

| Label | Cells | Shape |
|--------|--------|--------|
| $W_f$ | B8:G10 | 3×6 (one row per cell) |
| $b_f$ | B12:D12 | 3 biases |

Example $W_f$ row 1 (B8:G8): `0, 0, 0, -0.025, 0, 0`  
$b_f$: `0.01, 0.05, 0.01`

### Per-cell calculation (rows 5–7)

For each cell (1–3):

1. **M** = `MMULT(W_f row, z)` — dot product with concatenated input  
2. **N** = M + bias  
3. **O** = `1/(1+EXP(-N))` — sigmoid → **$f_t$**

### Concatenated vector $z$

| Label | Cells |
|--------|--------|
| $h_{t-1} + x_t$ (horizontal) | B14:G14 |
| Transpose of $z$ (for MMULT) | B16:B21 |

---

## Sheet 2: Input Gate

**Formulas:**

- $i_t = \sigma(W_i \cdot z + b_i)$
- $\tilde{C}_t = \tanh(W_C \cdot z + b_C)$
- $C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$

Row **4** is the header row; rows **5–7** are Cell 1, 2, and 3.

### Per row (rows 5–7) — main columns

| Column | Meaning | Example (Cell 1, row 5) |
|--------|---------|-------------------------|
| **B** | $f_t$ (from Forget Gate) | `='Forget Gate'!O5` |
| **D** | $C_{t-1}$ (editable) | `0.033` |
| **F** | $f_t \cdot C_{t-1}$ | `=B5*D5` |
| **J → K → L** | Input gate: `MMULT` → + $b_i$ → sigmoid → **$i_t$** | `L5` |
| **O → P → Q** | Candidate: `MMULT` → + $b_C$ → `TANH` → **$\tilde{C}_t$** | `Q5` |
| **S** | $i_t \cdot \tilde{C}_t$ | `=L5*Q5` |
| **T** | **$C_t$** | `=F5+S5` |

### Linked inputs (rows 11 & 13)

| Label | Cells |
|--------|--------|
| $h_{t-1}$ | B11:D11 → `'Forget Gate'!B4:D4` |
| $x_t$ | B13:D13 → `'Forget Gate'!B6:D6` |

### Weights and biases

| Label | Cells |
|--------|--------|
| $W_i$ | B15:G17 |
| $W_C$ | I15:N17 |
| $b_i$ | B19:D19 |
| $b_C$ | J19:L19 |

### Vector $z$ on this sheet

| Label | Cells |
|--------|--------|
| $h_{t-1}+x_t$ (horizontal) | B21:G21 |
| Transpose of $z$ (for `MMULT`) | B23:B28 |

Formula labels on this sheet: **P13** ($i_t$), **P15** ($\tilde{C}_t$), **P17** ($C_t$ equation).

---

## Sheet 3: Cell State & Output

**Formulas:**

- $o_t = \sigma(W_o \cdot z + b_o)$
- $h_t = o_t \odot \tanh(C_t)$

### From other sheets (linked)

| Item | Reference |
|------|-----------|
| $C_t$ | `'Input Gate'!T5`, `T6`, `T7` (shown in B4:B6) |
| $h_{t-1}$, $x_t$ | `'Forget Gate'!B4:D4`, `B6:D6` |

### Weights and bias

| Label | Cells |
|--------|--------|
| $W_o$ | B12:G14 |
| $b_o$ | B16:D16 |

Example $b_o$: `0.005, 0.001, 0.031`

### Per-cell (rows 5–7)

1. **L** = `MMULT(W_o row, z)`  
2. **M** = L + `b_o`  
3. **N** = sigmoid → **$o_t$**  
4. **P** = `TANH(C_t)`  
5. **Q** = **N × P** → **$h_t$**

---

## Excel functions used

| Function | Use |
|----------|-----|
| `MMULT` | Weight matrix × input vector |
| `EXP` | Sigmoid: `1/(1+EXP(-x))` |
| `TANH` | Candidate and $\tanh(C_t)$ |
| Cross-sheet refs | e.g. `='Forget Gate'!O5` |

Open the file in **Microsoft Excel** (or compatible) and press **F9** if values do not update after edits.

---

## How to use / experiment

1. Open `LSTM.xlsx` in Excel.  
2. Change **inputs** on Forget Gate (`h_(t-1)`, `x_t`) or **weights/biases** on any sheet.  
3. Confirm **Input Gate** and **Cell State & Output** recalculate.  
4. Final hidden state is **`h_t`** in column **Q**, rows **5–7** on **Cell State & Output**.

Suggested checks:

- Set all forget gate outputs near 1 → old cell state is mostly kept.  
- Change $C_{t-1}$ on Input Gate (**D5:D7**) → **T5:T7** ($C_t$) and **Q5:Q7** ($h_t$) should change.  
- Tweak one weight in $W_o$ → only output/hidden path should shift strongly.

---

## Project files (repo)

| File | Purpose |
|------|---------|
| `LSTM.xlsx` | Main workbook (forward LSTM) |
| `README.md` | This documentation |

---

## Summary

| Question | Answer |
|----------|--------|
| What is this? | LSTM **forward pass** in Excel for 3 units, one timestep |
| How many sheets? | 3 (gates + cell/output) |
| Is backprop included? | **No** (forward only in the current file) |
| Where is the final output? | **Cell State & Output**, cells **Q5:Q7** ($h_t$) |

If you add backprop later, keep it on a separate sheet and link to the same $z$, weights, and gate outputs documented above.
