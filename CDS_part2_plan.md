# CDS Project Part 2 — Building a Vulnerability Dataset from ProjectKB

## Context

Part 1 used a pre-made HDF5 dataset (`source` + `labels`) to evaluate a pre-trained
vulnerability-prediction model. Part 2 (`CDS_project_part2.ipynb`) asks us to **create such a
dataset ourselves**: mine SAP's ProjectKB knowledge base (CVEs → fixing commits in open-source
Java repos), extract *vulnerable* and *non-vulnerable* method versions, and save them as a
labeled dataset.

**Verified facts (measured on the real data, not assumed):**
- CVE statements live on the **`vulnerability-data` branch** of `https://github.com/SAP/project-kb`, in `statements/<CVE-ID>/statement.yaml` — **1297 CVE folders**.
- Parsing all statements yields **1819 usable fixing commits** across **1227 CVEs and 594 git repositories**; 66 entries are skipped (SVN repositories such as `svn.apache.org`, whose "commit ids" are numeric revision numbers — PyDriller is git-only).
- **~500 commit hashes in ProjectKB are truncated to 39 characters** (a genuine data quirk, verified in the raw YAML). Git cannot fetch a commit by a partial hash, so these need special handling (see mining strategy below) — and even with that fallback, a number of them still fail to resolve at mining time (dead/renamed repos, force-pushed history), so not all 1819 commits end up contributing samples.
- Repo hosts: github.com (1724 commits), gitbox/git-wip-us.apache.org (88), android.googlesource.com (6), salsa.debian.org (1).

**Environment (verified):** Python 3.9.6, PyDriller 2.5.1, PyYAML, h5py, pandas, GitPython, git 2.50 — all installed.

## Dataset design: real fix-commit pairs only, no padding

Following the construction used by real-world datasets (Big-Vul, Devign), each fixing commit
gives one labeled pair per method it changed:

1. **Vulnerable (label = True):** the *pre-commit* version of every method changed by a fixing commit.
2. **Fixed / non-vulnerable (label = False):** the *post-commit* version of that same method.

**We deliberately do not add "unchanged" methods as extra negatives.** An early iteration of
this pipeline explored padding the negative class with unchanged methods from the same files
(capped at 30/file) specifically to hit a large sample-count target. That was rejected: an
unchanged method is **not evidence of non-vulnerability** — it may simply be an
undiscovered/unfixed vulnerability — so padding with it injects label noise and fakes the
class balance. The implemented notebook only ever emits genuine before/after pairs from real
fixing commits, and reports the real size instead of forcing a target — **the task sets no
minimum sample count**, and Part 3 only needs enough data to fine-tune on.

(The large pool of same-repo unchanged methods was not thrown away — it's kept as an archived,
separate artifact and later reused deliberately in Part 3, with proper controls, as
length-matched "easy negatives" for dataset v2. See `PART3_TRAINING_APPROACHES.md`, Approach 8.)

## Mining strategy (how the data is actually fetched)

**Full `git clone` of every repository does not work at this scale.** An early pilot proved
it: `android.googlesource.com/platform/frameworks/base` alone stalled the run at 7.4 GB
downloaded (the repo is tens of GB), with 593 repos still to go. The implemented solution
avoids whole-repo clones entirely:

1. **Per fixing commit, fetch only what PyDriller needs** — the commit and its parent, without file contents:
   `git fetch --depth 2 --filter=blob:none origin <full-sha>`
   PyDriller then lazily pulls only the few file blobs the fix actually touched. This is a few MB per commit *regardless of repository size*.
2. **Truncated 39-char hashes** (cannot be fetched by SHA): fall back to a **one-time blobless history fetch** of the repo (`git fetch --filter=blob:none origin '+refs/heads/*:refs/remotes/origin/*'` — history without file contents), then resolve the short hash locally with `git rev-parse`. This resolves most, but not all, truncated hashes.
3. **Merge commits** (2+ parents) return no modified files from PyDriller by default, yet ProjectKB often records the *merge* itself as the fix. For those, diff the merge against its first parent directly with git and parse changed methods with `lizard` (the same method-boundary parser PyDriller uses internally), guarded by `MERGE_MAX_FILES = 10` to skip giant branch-integration merges that aren't genuinely targeted fixes.
4. **Robustness:** `GIT_TERMINAL_PROMPT=0` (dead/private repos fail fast instead of hanging), timeouts on every git call, per-commit try/except — failures are logged and skipped, never fatal.
5. **Checkpointing:** every extracted sample is appended to `data/mining_checkpoint.jsonl` and every processed commit to `data/processed_commits.txt`; re-running the cell skips processed commits, so the hours-long mining loop is interruptible and resumable.

**Scope:** Java-only, test files excluded — the model is evaluated on Java, so training data
must match that language.

## Implementation (in `CDS_project_part2.ipynb`)

- **Setup cell:** config constants (`KB_DIR`, `REPOS_DIR`, `DATA_DIR`, `MAX_REPOS` — set to a small number for a pilot run, `None` to mine everything, `MERGE_MAX_FILES = 10`).
- **Task 1 cell:** clone ProjectKB (`--branch vulnerability-data --single-branch --depth 1`, idempotent), count and list the CVE folders.
- **Task 2 cell:** parse every `statement.yaml` with `yaml.safe_load`; collect `(cve_id, repo_url, commit_hash)` from `fixes[].commits[]`; normalize URLs (strip `/`, `.git`); keep only `https://` non-SVN repos with 7–40-char hex commit ids; dedupe into the `fix_commits` DataFrame.
- **Task 3 cell:** the mining loop described above. Per modified `.java` file (`change_type == MODIFY`, path not containing "test") and per merge-commit diff: vulnerable + fixed versions of each changed method (matched by `long_name` between `methods_before` and `changed_methods`, sliced by `start_line..end_line`). No unchanged/padding methods are added.
- **Task 4 cells:** load the checkpoint into a DataFrame; clean (same code never appears with both labels — vulnerable wins; drop exact duplicate code; drop methods with fewer than 2 newlines — one-line getters/setters carry no signal); report the real dataset size (no target enforced); save `data/vulnerability_dataset.hdf5` with datasets `source` (UTF-8 strings) + `labels` (bool) — the same structure as Part 1's `student_dataset.hdf5` — plus `data/vulnerability_dataset_metadata.csv` with full provenance (CVE, repo, commit, file, method). Reload the HDF5 and print one sample per class as a round-trip check.

## Files
- Modified: `CDS_project_part2.ipynb`
- Runtime artifacts: `project-kb/` (KB clone), `repos/` (minimal per-commit fetches), `data/mining_checkpoint.jsonl`, `data/processed_commits.txt`, `data/vulnerability_dataset.hdf5`, `data/vulnerability_dataset_metadata.csv`
- `repos/` and `project-kb/` can be deleted once the dataset is built.

## Status & result (final)
- [x] Task 1: 1297 CVE folders found.
- [x] Task 2: 1819 fixing commits / 1227 CVEs / 594 repos extracted, 66 skipped.
- [x] Task 3: full mining run completed (checkpointed, resumable); some commits still fail to resolve even with the blobless-history fallback (dead/renamed repos, force-pushed history) and are skipped, not fatal.
- [x] Task 4: **8,540 raw samples mined → 6,530 after cleaning** (3,331 vulnerable / 3,199 fixed, **51.0% vulnerable**, drawn from **663 CVEs across 264 repositories**). HDF5 round-trips correctly; both classes present.

This is the dataset Part 2 delivers as-is — no padding, no forced size target. Part 3 builds a
separate, larger **dataset v2** on top of these pairs (see `PART3_TRAINING_APPROACHES.md`,
Approach 8) once pointwise classification on pure before/after pairs turned out to need
realistic "ordinary code" negatives to match the actual challenge task.
