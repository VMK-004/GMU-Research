# DCD Research Notes — circuit discovery for code generation

Project notes for Prof. Yao | last updated 9/16/2026

*(Same content as `Research_Overview_Mohan_DCD_notes.docx`. Edit this `.md` in Cursor; keep the Word file if you need to paste into Google Docs.)*

---

## Project goal

This project builds on Data-driven Circuit Discovery (DCD; [arXiv:2605.09129](https://arxiv.org/abs/2605.09129)) and asks whether language models use different internal circuits/mechanisms for code-generation tasks that look like a single task to humans. The DCD paper showed this kind of mismatch on tasks such as IOI; the goal here is to test whether the same finding holds for coding, starting from simple settings such as balanced parentheses after the released DCD pipeline is understood and replicated.

Released code: https://github.com/Ziyu-Yao-NLP-Lab/data-driven-circuit-discovery

This document is written about the project (not one person’s voice). Weekly progress and group updates can be filled by anyone on the DCD track (including Arjun).

---

## Concepts

### Circuit

A circuit is a small subset of the model’s computational graph (important edges / wires between components such as attention heads and MLPs) that is enough to explain a specific behavior. It is not the whole model — only the parts that matter for that behavior.

### Task vs what the model may actually do

Humans often label many examples as one task (for example IOI, or later “close the parentheses”). DCD’s point is that the model may still use different internal mechanisms for different subsets of those examples. So a circuit found on one dataset variant may not be a general “task circuit.”

### Clean vs corrupted examples

Each dataset row has a clean prompt and a corrupted (counterfactual) prompt with matching token length. Attribution uses that pair to score which edges matter for the correct behavior on that example. From the IOI+GPT-2 run, one real train row was:

- **clean:** Then, Heather and Justin had a lot of fun at the school. Justin gave a snack to
- **corrupted:** Then, Heather and Justin had a lot of fun at the school. Jesse gave a snack to
- **label:** Heather | **corrupted_labels:** Justin | **prompt_type:** ABBA

### Attribution / EAP-IG

Attribution scores every edge for one example: how much that edge mattered for getting the right answer on that example. EAP-IG is the scoring method used in the config (`EAP-IG-inputs`). The output for one example is a fingerprint (`edges_scores` in a `.pt` file), not yet a finished circuit drawing.

### DCD pipeline

DCD does not assume one circuit for the whole dataset. Conceptually:

- **01** — build examples (clean/corrupted CSVs)
- **02** — fingerprint each train example (per-example EAP-IG)
- **03** — cluster examples with similar fingerprints
- **04** — build one circuit per cluster (keep important edges)
- **05** — faithfulness: if only those edges remain, does behavior still match the full model?

### Train vs test (in this repo’s DCD setup)

Train is used to discover circuits (attribution + clustering + circuit creation). Test is used to grade faithfulness. They must not be the same rows. In the IOI+GPT-2 run: train had 594 examples; test had 204 examples.

### Faithfulness

Faithfulness measures how close a (possibly sparse) circuit is to the full model’s behavior on the evaluation set. Circuit size is the fraction of top edges kept (for example 0.05 ≈ top 5%). Score near 1 means the circuit matches the full model on the metric; near 0 means it does not. CPR / CMD summarize the faithfulness curve across sizes.

### Why multiple circuits matter

A single circuit discovered on all training examples can look weak when kept sparse, because it may be mixing different mechanisms. DCD’s multiple circuits are meant to specialize. Average faithfulness alone is not enough later — cluster specialization matters too (especially because `random_k_split` can also look strong under best-of-K).

---

## Weekly progress (DCD track)

Update this section every week. Past weeks keep completed work; the current week stays blank until filled.

### Week of 8/3 – 8/9 (filled)

- DCD repo set up on Hopper under `/scratch/mvallab/research/data-driven-circuit-discovery`.
- Python 3.11.7 venv; requirements + EAP-IG + MIB-circuit-track installed.
- Full DCD pipeline 01→05 run with `configs/dcd/ioi/gpt2.yaml` (GPT-2 only).
- 01: IOI data built (train=594 across 6 prompt types; test=204).
- 02: EAP-IG fingerprints for all 594 train examples.
- 03: clustering → `best_configs` / `all_configs` (e.g. k=4 and k=7 depending on method).
- 04: DCD cluster circuits created; E-Act baseline hit CUDA OOM and was skipped.
- 05: faithfulness on `test.csv`. At circuit size 0.05: DCD methods ~0.82–0.86; single circuit 0.51; EAP baseline 0.74; random ~0; `random_k_split` also high (~0.88).
- Results saved under `results/dcd/ioi/gpt2/evaluation/` (`faithfulness_results.json`, `cpr_cmd_results.json`).
- Short notes added to Manas and Lakshmi’s research doc on understanding the secure code generation paper and a possible later link to DCD (behavior vs mechanisms).

### Week of 8/10 – 8/16

- (to fill)
- (to fill)
- (to fill)

### Week of 8/17 – 8/23

- (to fill)

### Week of 8/24 – 8/30

- (to fill)

### Week of 9/1 – 9/15 (filled)

- Read end-to-end *Failure by Interference: Language Models Make Balanced Parentheses Errors When Faulty Mechanisms Overshadow Sound Ones* (NeurIPS 2025; arXiv:2507.00322) and *Interpretability in the Wild: A Circuit for Indirect Object Identification in GPT-2 Small* (arXiv:2211.00593).
- **Failure by Interference (balanced parentheses):** the task is predicting the correct number of closing parentheses; the paper decomposes it into **One-Paren**, **Two-Paren**, **Three-Paren**, and **Four-Paren** sub-tasks from how the tokenizer represents \(N \in \{1,2,3,4\}\) closing parentheses as single tokens.
- Understanding framed top-down: study **attention heads** and **FF neurons** that contribute to the **final logit** from the last-token position, using the **logit lens** (Algorithms 1–2 for task correctness and token promotion).
- Core finding from the paper: LMs use many components with varying **generalizability** and **reliability**; some implement **sound mechanisms**, others **faulty mechanisms**; errors occur when faulty mechanisms **overshadow** sound ones (**failure by interference**). Related paper terms: **noisy promotion**, **dual-sign mechanism** (FF neurons), low **precision** / **recall** / selectivity; mitigation via **RASTEER** (rank by soundness, then steer by scaling activations); paper also reports a **circuit baseline** using **activation patching**.
- **Interpretability in the Wild (IOI):** read as the standard example of **mechanistic interpretability** / **circuit** analysis “in the wild” — how **GPT-2 small** implements **indirect object identification (IOI)**.
- Methods and validation terms from the IOI paper: **path patching**, **knockouts**, **mean ablation**; criteria **faithfulness**, **completeness**, and **minimality**. Circuit head classes named in the paper include **Name Mover Heads**, **Duplicate Token Heads**, **S-Inhibition Heads**, and **Backup Name-Mover Heads**.

### Week of 9/16 – 9/22 (current — brainstorm + Sep 22)

- Brainstorm how the two papers connect to the main pathway already scoped for this project: run **DCD** on the balanced parentheses tasks (1–4 closing parentheses) and test whether the model establishes **distinct circuits** for different numbers of closing parentheses (and later different task contexts), per Prof. Yao’s framing.
- Role mapping (paper terms only): the IOI paper is bottom-up **circuit** discovery with causal interventions (**path patching** / **knockouts**); Failure by Interference is top-down at the final logit (**sound** vs **faulty** mechanisms across One–Four-Paren); **DCD** is the experimental vehicle — per-example **EAP-IG** fingerprints, **clustering**, multiple circuits, and **faithfulness**.
- Link to DCD’s question: Failure by Interference already shows specialized vs more **generalizable** components across One–Four-Paren; DCD asks whether different example subsets (e.g. different numbers of closings) use **distinct circuits** — natural next experiment after the IOI+GPT-2 replication.
- **By Sep 22:** prepare a short professor-facing progress summary from the 9/1–9/15 reading block and this brainstorm.
- **By Sep 22:** plan how to align the failure-by-interference **One–Four-Paren** templates with DCD’s **clean / corrupted** pairs (train vs test), using the [failure-by-interference](https://github.com/Ziyu-Yao-NLP-Lab/failure-by-interference) dataset/repo.
- **By Sep 22:** lock the first post–Sep-22 DCD experiment scope — one model already studied in Failure by Interference (e.g. a GPT-2 size) and ask whether **Three-Paren** vs **Four-Paren** induce **distinct circuits** under DCD clustering (matching Prof. Yao’s stated question).
- Optionally skim the failure-by-interference repo layout so the post–Sep-22 DCD data build (pipeline 01) is concrete.

---

## Group progress

Space for updates from everyone related to this research group.

### Arjun (DCD track)

- (to fill)
- (to fill)

### Manas

- (to fill)
- (to fill)

### Lakshmi

- (to fill)
- (to fill)

---

## Next for the DCD track

- IOI + GPT-2 replication of the DCD repo (01→05) is complete from previous weeks.
- Reading of Failure by Interference (balanced parentheses / **sound** vs **faulty** mechanisms, **RASTEER**) and Interpretability in the Wild (IOI **circuit**, **path patching**, faithfulness / completeness / minimality) is complete (week of 9/1–9/15).
- **Through Sep 22:** finish the brainstorm write-up for Prof. Yao; plan clean/corrupted adaptation of **One–Four-Paren** templates for DCD; lock first experiment as **Three-Paren** vs **Four-Paren** distinct-circuit check under DCD clustering on one GPT-2-scale model from the Failure by Interference study.
- **Main next goal (after Sep 22):** run DCD on the balanced parentheses tasks (1–4 closing parentheses) from the [failure-by-interference](https://github.com/Ziyu-Yao-NLP-Lab/failure-by-interference) repo — start with pipeline 01 (clean/corrupted CSVs), then EAP-IG fingerprints → clustering → circuits → faithfulness.
- Along the way: test whether different numbers of closings (and later different contexts) use distinct circuits.
- Optional leftover from IOI: specialization plots / E-Act baseline only if needed; not the main focus now.
