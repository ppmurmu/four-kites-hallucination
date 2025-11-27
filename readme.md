# Factuality & Hallucination Detection Framework
### Architectural Approach & Methodology

## 1. Problem Statement
Large Language Models (LLMs) exhibit two primary failure modes regarding factuality:
1.  **Uncertainty:** The model knows it lacks information but is forced to complete a sequence, leading to low-confidence guessing.
2.  **Confabulation (Misbelief):** The model confidently generates false information due to corrupted training data or pattern mimicking.

Standard detection methods (RAG or Human Eval) are resource-intensive. This framework solves the problem using **intrinsic model signals** and **behavioral consistency** to detect hallucinations without external ground truth.

---

## 2. Core Hypothesis
A model that possesses factual knowledge about a query will exhibit **Stability**:
* **Intrinsic Stability:** High probability assigned to the selected tokens.
* **Extrinsic Stability:** Consistent answers across multiple regeneration attempts, even when randomness (temperature) is introduced.

Conversely, a hallucinating model will exhibit **Instability**: high entropy (confusion) or semantic drift (changing the story) when re-queried.

---

## 3. The Methodology: "Constrained Interrogation"

We employ a **Dual-Signal Pipeline** to quantify this stability.

### Step 1: Constrained Input (Signal Isolation)
LLMs often mask hallucinations with verbose language (e.g., *"It is generally believed that..."*). To isolate the factual signal:
* **Strategy:** We use specific Prompt Engineering to force a **Single-Token / Single-Word** output.
* **Goal:** Remove linguistic noise. If the model is asked for a date, it must provide *only* the date. This makes verification binary and precise.

### Step 2: Intrinsic Signal Detection (The "Gut Check")
We analyze the model's internal confidence before it outputs text.
* **Mechanism:** Accessing the raw **Logits** of the generated tokens.
* **Metric:** **Average Token Confidence (0.0 - 1.0)**.
* **Logic:** A low confidence score indicates the model is statistically "guessing" the next token, which is a strong proxy for hallucination type #1 (Uncertainty).

### Step 3: Extrinsic Signal Detection (The "Cross-Examination")
We test the robustness of the answer by trying to force the model to slip up.
* **Mechanism:** **Self-Consistency Sampling**.
* **Process:** We generate the answer $N$ times (e.g., 3-5 samples) with a high Temperature ($T=0.7$).
* **Analysis:** We use a lightweight Embedding Model (MiniLM) to calculate the **Semantic Cosine Similarity** between the $N$ samples.
* **Logic:**
    * *Factual:* The model converges on the same answer despite randomness (e.g., "Paris", "Paris", "Paris"). **Consistency $\approx$ 1.0**.
    * *Hallucination:* The model drifts into different plausible guesses (e.g., "1999", "2005", "Unknown"). **Consistency $\downarrow$**.

---

## 4. Scoring Algorithm
The final **Hallucination Index** is a weighted aggregate of the two signals.

$$\text{Hallucination Index} = 1.0 - (w_1 \times \text{Confidence} + w_2 \times \text{Consistency})$$

* **Weights:** We typically weight Consistency higher ($w_2=0.7$) because it catches "confident hallucinations" where the model is sure but wrong, whereas Logits ($w_1=0.3$) only catch "uncertainty."

### Interpretation
* **Low Score (0.0 - 0.3):** **Reliable.** The model is confident and tells the same story repeatedly.
* **High Score (0.6 - 1.0):** **Hallucination.** The model is either guessing (low confidence) or making up contradictory facts (low consistency).