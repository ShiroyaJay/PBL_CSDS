# CSDS PBL — Vulnerability Detection in Java Methods

Course project for the MSc course **Cybersecurity Data Science** (PBL), Hamburg University
of Technology (TUHH). The task: given the source code of a single Java method, predict
whether it is vulnerable.

**Result: Rank #1 on the course challenge leaderboard** — F1 62.3%, precision 55.1%,
recall 71.7%, accuracy 74.0% on the 1,000-method challenge set.

## Contributors

<!-- Name, Surname | Matriculation No -->
- Jay Shiroya
- *(add teammate)*
- *(add teammate)*

## Repository structure

| Path | Description |
|------|-------------|
| `lab/CDS_project_part1.ipynb` | Part 1 — baseline model on the provided student dataset |
| `CDS_project_part2.ipynb` | Part 2 — dataset construction: mining vulnerable/fixed Java methods from [SAP ProjectKB](https://github.com/SAP/project-kb) fix commits with PyDriller |
| `CDS_project_part3.ipynb` | Part 3 — model training and challenge submission (UniXcoder + static-feature hybrid) |
| `CDS_part2_plan.md` | Part 2 planning notes |
| `PART3_TRAINING_APPROACHES.md` | Full experiment log — every approach tried in Part 3, including the failures |
| `lab/requirements.txt` | Python dependencies |

## Approach in one paragraph

Part 2 mines SAP ProjectKB: each CVE statement maps to the Git commit(s) that fix it. Rather
than cloning all 594 referenced repositories in full (infeasible — some are tens of GB), we
fetch only each fixing commit and its parent (`git fetch --depth 2 --filter=blob:none`) and
extract, with PyDriller, the Java methods it changed — the pre-commit version labelled
*vulnerable*, the post-commit version *fixed*. No synthetic "unchanged method" padding is
added, so the result is honest and roughly balanced: **6,530 methods, 51.0% vulnerable, 663
CVEs, 264 repos**. Part 3 rebuilds this into **dataset v2 — 14,887 methods (20.4% vulnerable,
691 CVEs, 270 repos)** by adding the post-fix twins as hard negatives and length-matched
unchanged same-repo methods as easy negatives. Part 3 shows that pointwise classification on pure before/after
pairs is at chance (the two classes are near-identical code), that a contrastive pair-ranking
loss is the first thing to beat chance, and that the decisive lift comes from **rebuilding the
dataset to match the real task** ("vulnerable vs ordinary code", not "pre-patch vs
post-patch"). The final model is a frozen-bottom **UniXcoder** encoder with 16 regex-based
static security features concatenated to the CLS embedding, trained with class-weighted BCE
under a leakage-free repository-grouped split. See `PART3_TRAINING_APPROACHES.md` for the
complete history.

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
