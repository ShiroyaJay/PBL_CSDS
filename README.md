# CSDS PBL — Vulnerability Detection in Java Methods

Course project for the MSc course **Cybersecurity Data Science** (PBL), Hamburg University
of Technology (TUHH). The task: given the source code of a single Java method, predict
whether it is vulnerable.

**Result: Rank #1 on the course challenge leaderboard** (1,000-method challenge set) — see
[Results](#results).

## Contributors

<!-- Name, Surname | Matriculation No -->
- Jay Sureshbhai Shiroya | 674160
- Daniel Schaumann | 51006

## Repository structure

| Path | Description |
|------|-------------|
| `lab/CDS_project_part1.ipynb` | Part 1 — baseline model on the provided student dataset |
| `CDS_project_part2.ipynb` | Part 2 — dataset construction: mining vulnerable/fixed Java methods from [SAP ProjectKB](https://github.com/SAP/project-kb) fix commits with PyDriller |
| `CDS_project_part3.ipynb` | Part 3 — model training and challenge submission (UniXcoder + static-feature hybrid) |
| `lab/requirements.txt` | Python dependencies |

## Approach

**Part 2 — dataset.** Each SAP ProjectKB CVE maps to its fixing commit(s). Rather than cloning
all 594 referenced repositories in full (infeasible — some are tens of GB), we fetch only each
fixing commit and its parent (`git fetch --depth 2 --filter=blob:none`) and use PyDriller to
extract the Java methods the commit changed — the pre-commit version labelled *vulnerable*, the
post-commit version *fixed* (**6,530 methods, 51% vulnerable, 663 CVEs, 264 repos**).

**Part 3 — the key finding.** On pure before/after pairs the two classes are near-identical code
(embedding cosine 0.995), so *pointwise* classification sits at chance and even a contrastive
pair-ranking loss only reaches ~0.58 AUC. The decisive lift came from **rebuilding the dataset to
match the real task** — "vulnerable vs ordinary code", not "pre-patch vs post-patch". Dataset v2
adds the post-fix twins as hard negatives and length-matched same-repo methods as easy negatives
(**14,887 methods, 20.4% vulnerable, 691 CVEs, 270 repos**).

**Final model.** A **UniXcoder-base** encoder (embeddings + bottom 8 layers frozen) with 16
regex-based static security features concatenated to the CLS embedding, trained pointwise with
**class-weighted BCE** under a leakage-free **repository-grouped** split; the operating threshold
is picked on validation max-F1. An auxiliary pair-ranking loss was tried on v2 and *dropped* — it
hurt (val PR-AUC 0.271 vs 0.290).

## Results

Rank #1 on the course challenge leaderboard (1,000 held-out Java methods):

| Metric | Score |
|--------|-------|
| F1 | 62.3% |
| Precision | 55.1% |
| Recall | 71.7% |
| Accuracy | 74.0% |

## Not included in this repo

Large, generated, or course-provided artifacts are excluded (see `.gitignore`) and are
produced or downloaded locally when running the notebooks:

- `repos/` — source repositories cloned for commit mining (~7.4 GB)
- `data/` — mined datasets (HDF5 + metadata CSV), embeddings, mining checkpoints
- `project-kb/` — clone of the external SAP ProjectKB repository
- `Task/`, `referece_part2.ipynb` — course-provided task sheets and challenge data (not ours to publish)
- model checkpoints (`*.pth`) and other binary artifacts

## Setup & running

```bash
pip install -r lab/requirements.txt
```

1. **Part 1** (`lab/CDS_project_part1.ipynb`) — runs locally on the provided student dataset.
2. **Part 2** (`CDS_project_part2.ipynb`) — clones ProjectKB and the referenced repositories,
   then mines the dataset. Long-running; mining is checkpointed (`data/processed_commits.txt`)
   and can be interrupted and resumed.
3. **Part 3** (`CDS_project_part3.ipynb`) — builds dataset v2 from the Part 2 outputs and
   trains the model. Designed for a GPU runtime (Google Colab T4, ~30–40 min); only the
   dataset-build and smoke-test cells run on CPU.
