# Transformer Scaled Dot-Product Attention Hardware – ECE564 Final Project

This repository contains a **SystemVerilog-based hardware implementation** of the **Scaled Dot-Product Attention** block — a core component in Transformer models such as GPT and BERT. Designed for **ECE564: ASIC Design**, the project maps high-level neural network operations into synthesizable RTL and simulates them using ModelSim.

---

## Project Overview

Transformers compute contextual relationships between words via **self-attention**. The central formula this project implements is:

Z = softmax((Q × Kᵀ) / sqrt(dₖ)) × V

This hardware design computes:
- Query (Q), Key (K), and Value (V) matrices via matrix multiplications
- Score matrix \( S = QK^T \)
- Final attention output \( Z = S \cdot V \)

---
## Flow chart
  ![image](https://github.com/user-attachments/assets/fcdbeb00-a74e-4013-919d-7a4058e82877)

---

## Implementation Details

### Step 1: Matrix Loading from SRAM

The DUT loads input and weights from SRAM. SRAM has latency, so timing must be managed carefully.

```verilog
always_ff @(posedge clk) begin
  if (!reset_n) begin
    state <= IDLE;
    // Reset logic
  end else begin
    case (state)
      IDLE: begin
        if (dut_valid) begin
          state <= READ_INPUT;
          dut_ready <= 0;
        end
      end
      READ_INPUT: begin
        // Load matrices I and WQ/WK/WV from SRAM into internal buffers
        read_counter <= read_counter + 1;
      end
```

### Step 2: Q, K, V Matrix Computation
The module performs dense matrix multiplication to generate Query, Key, and Value matrices:
```verilog
for (int i = 0; i < Q_ROWS; i++) begin
  for (int j = 0; j < Q_COLS; j++) begin
    Q[i][j] = 0;
    for (int k = 0; k < I_COLS; k++) begin
      Q[i][j] += I[i][k] * WQ[k][j];
    end
  end
end
```
The same logic is reused for K and V by switching out the weight matrix.

### Step 3: Compute Score Matrix 
After computing Q and K, the DUT performs a dot product between each row of Q and each column of Kᵀ:
```verilog
for (int i = 0; i < Q_ROWS; i++) begin
  for (int j = 0; j < K_ROWS; j++) begin
    S[i][j] = 0;
    for (int k = 0; k < Q_COLS; k++) begin
      S[i][j] += Q[i][k] * K[j][k];  // Note transpose: K[j][k]
    end
  end
end
```
### Step 4: Final Output 
Finally, matrix Z is computed using a standard matrix multiplication:
```verilog
for (int i = 0; i < S_ROWS; i++) begin
  for (int j = 0; j < V_COLS; j++) begin
    Z[i][j] = 0;
    for (int k = 0; k < S_COLS; k++) begin
      Z[i][j] += S[i][k] * V[k][j];
    end
  end
end
```
### Step 5: Writing Output to SRAM
The resulting matrix Z is written back to SRAM for the testbench to verify:
```verilog
SRAM_result[addr] <= Z[row][col];
```

## Timing Diagram
![image](https://github.com/user-attachments/assets/3d21a806-289a-4893-b886-e8086941b3af)

## Results Achieved
Throughput = 1/( # cycles x time period) = 26469.03

Area achieved is: 12045.0121
