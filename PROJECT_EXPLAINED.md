# The Whole Project, Explained From Zero — Vulnerability Detection in Java Code

> **Who this is for.** Someone new to *both* security and machine learning who has to
> understand — and be examined on — this three-part project: what each part does, why, and
> how. Every technical term is defined the first time it appears. Read it top to bottom.
>
> **The one-sentence story of the whole project:**
> **Part 1** teaches you to *run and grade* a vulnerability-detection model (and discover why
> the usual "accuracy" grade lies). **Part 2** *builds an honest dataset* of vulnerable and
> fixed Java methods from real security fixes. **Part 3** *trains our own model* on that data
> and wins the course challenge — after discovering that the way you *build the data* matters
> more than the model you choose.
>
> This document covers **only the approach that actually worked** in each part, in the detail
> a beginner needs to prepare for the exam.

---

## Table of contents

- [0. The big picture](#0-the-big-picture)
- [1. Universal vocabulary](#1-universal-vocabulary-learn-once-reuse-everywhere)
- [2. Part 1 — Run and evaluate a given model](#2-part-1--run-and-evaluate-a-given-model)
- [3. Part 2 — Build an honest dataset](#3-part-2--build-an-honest-dataset)
- [4. Part 3 — Train our own model (the challenge win)](#4-part-3--train-our-own-model-the-challenge-win)
- [5. Exam cheat-sheet](#5-exam-cheat-sheet-one-line-answers)
- [6. Glossary index](#6-glossary-index)

---

## 0. The big picture

The project is a **cybersecurity data-science pipeline**: use machine learning to decide
whether a piece of source code contains a **security vulnerability** (a flaw an attacker
could exploit). It is split into three parts that form one continuous story:

| Part | Question it answers | What "worked" |
|------|---------------------|---------------|
| **1** | *How do we run a model and measure if it's any good?* | Learned to load a pre-trained model, compute TP/FP/FN/TN by hand, and conclude that **accuracy is misleading — F1 and recall tell the truth** on imbalanced security data. |
| **2** | *Where does trustworthy training data come from?* | Mined real CVE fix commits into **before(vulnerable)/after(fixed) Java method pairs** — no fake padding — giving an honest, balanced dataset. |
| **3** | *Can we train a model that actually detects vulnerabilities?* | Yes — but only after **rebuilding the dataset to match the real task**; final model reached **F1 62.3%, Rank #1** on the leaderboard. |

Everything below expands each part. Part 3 is the heaviest because it's where the real
insight lives.

---

## 1. Universal vocabulary (learn once, reuse everywhere)

### 1.1 Security terms

- **Vulnerability:** a flaw in code an attacker can exploit — to run commands, read files
  they shouldn't, crash the program, or inject malicious data.
- **CVE (Common Vulnerabilities and Exposures):** a public, globally unique ID for one known
  vulnerability, e.g. `CVE-2019-3772`. The "ticket number" of a security bug.
- **Fix commit / patch:** the exact code change (a Git *commit*) that removes a vulnerability.
  *Before* it the code is **vulnerable**; *after* it, **fixed**.
- **Sink:** a dangerous operation — `Runtime.exec()` (runs a shell command), `ProcessBuilder`,
  raw SQL string concatenation, Java deserialization, reflection. Untrusted input reaching a
  sink is where many vulnerabilities live.
- **Sanitizer / guard:** code that makes input safe before it reaches a sink — a null-check,
  bounds-check, escaping/validation. **Security fixes very often *add* a guard.**
- **XXE (XML External Entity):** a vulnerability class where an XML parser is tricked into
  reading local files or making network requests; fixed by configuring the parser
  (`XMLInputFactory`, `DocumentBuilderFactory`, `SAXParserFactory`). *(Reappears in Part 3.)*

### 1.2 Machine-learning terms

- **Model:** a mathematical function with many tunable numbers (**weights / parameters**)
  mapping an input to an output.
- **Training:** adjusting the weights so outputs match the correct answers on examples.
- **Inference / prediction:** using a trained model to score new inputs (no learning).
- **Label:** the correct answer for one example. Here `1` = vulnerable, `0` = not.
- **Dataset / data loader:** the collection of examples, and the code that feeds them to the
  model in **batches** (small groups) so they fit in memory.
- **HDF5:** a binary file format for storing large numeric datasets efficiently. Both the
  provided data and ours use it (keys `source` = code text, `labels` = true/false).
- **Encoder / (code) language model:** a large neural network **pre-trained** on huge amounts
  of code, so it already "understands" syntax. We reuse it instead of learning code from
  scratch. Parts use **CodeBERT** / **UniXcoder**.
- **Embedding (feature vector):** the encoder's output — ~768 numbers summarizing a method's
  meaning. Similar code → similar vectors.
- **Logit → sigmoid → probability:** the model's raw score is a **logit**; a **sigmoid**
  function squashes it into a probability between 0 and 1.
- **Loss function:** a number measuring *how wrong* the model is right now. Training = making
  loss go down. **BCE (Binary Cross-Entropy)** is the standard loss for yes/no problems.
- **Epoch:** one full pass over the training data.
- **Overfitting:** the model memorizes training examples instead of learning general rules —
  great on training data, bad on new data. Measured by the train-vs-test score gap.
- **Chance / baseline:** the score a know-nothing model gets. If a metric sits at chance, the
  model learned nothing useful.

### 1.3 Evaluation terms (how we grade — the heart of Part 1)

The model labels many methods; compare its guesses to the truth:

- **TP (true positive):** predicted vulnerable, actually vulnerable ✅
- **FP (false positive):** predicted vulnerable, actually safe ❌ (false alarm)
- **FN (false negative):** predicted safe, actually vulnerable ❌ (**missed vuln — the
  dangerous mistake in security**)
- **TN (true negative):** predicted safe, actually safe ✅

The four combine into metrics:

- **Accuracy** = (TP + TN) / everything. *Fraction correct.* **Misleading on imbalanced
  data** — if 72% of methods are safe, always guessing "safe" scores 72% while catching zero
  vulnerabilities.
- **Precision** = TP / (TP + FP). *Of what I flagged, how much was real?* High = few false
  alarms.
- **Recall (sensitivity)** = TP / (TP + FN). *Of all real vulnerabilities, how many did I
  catch?* High = few misses. **The priority metric in security.**
- **F1 score** = harmonic mean of precision and recall = `2·P·R/(P+R)`. One number that is
  only high when **both** are decent. The leaderboard ranks on F1.
- **Threshold:** the probability cutoff (e.g. 0.5, or a tuned value) above which we call a
  method "vulnerable." Moving it trades precision against recall.
- **ROC-AUC:** threshold-independent ranking quality (0.5 = chance, 1.0 = perfect).
- **PR-AUC:** like ROC-AUC but for **imbalanced** data; its chance level equals the positive
  fraction (~0.20–0.30 here), not 0.5.
- **MCC (Matthews Correlation Coefficient):** a single balanced score (−1 to +1) that stays
  honest even under class imbalance — a good alternative headline metric for this task.

---

## 2. Part 1 — Run and evaluate a given model

### 2.1 The task

You are **given** a model that was already trained on a vulnerability dataset. Your job is
**not** to train anything — it's to **run it on a validation set of 1,000 code samples and
grade it properly**, then interpret what the grades mean. The point of Part 1 is to *learn the
evaluation vocabulary* you'll rely on for the rest of the project.

The seven tasks, and the reasoning behind each:

1. **Build a data loader** for `student_dataset.hdf5`. *Why:* models read data in batches
   from a standard interface; writing the loader teaches you how examples flow into a model.
2. **Show 10 random samples with labels.** *Why:* always eyeball your data before trusting
   any metric — sanity check that code text pairs with a true/false label.
3. **Inspect the class balance.** Result: **1,000 samples, 283 vulnerable (~28.3%), 717 safe.**
   *Why this matters:* the data is **imbalanced** — this single fact is what makes accuracy
   untrustworthy later.
4. **Load and run the pre-trained model** (`VulnPredictModel`). It's a tiny neural network:
   it takes a **768-number embedding** of a code sample and passes it through
   `Linear(768→64) → ReLU → Linear(64→64) → ReLU → Linear(64→1) → Sigmoid`, outputting one
   probability. *(ReLU is a simple non-linear function that lets the network learn
   non-straight-line patterns; Sigmoid turns the final score into a 0–1 probability.)*
5. **Predict and count TP/TN/FP/FN** at threshold 0.5. This is the **confusion matrix** — the
   raw material for every metric.
6. **Compute Accuracy, Precision, Recall, F1 by hand** (not with a library) — so you
   understand the formulas, not just call them.
7. **Interpret the results** — the real learning objective.

### 2.2 The result that "worked" — i.e. the lesson

The given model scored **Accuracy 73.6%** but **F1 only 0.13** (Recall **0.07** — it caught
just **20 of 283** real vulnerabilities, missing 263). The take-aways, which are prime exam
material:

- **Accuracy lies on imbalanced data.** 73.6% looks fine, but a model that blindly says "safe"
  for everything already scores ~71.7%. The model's accuracy came almost entirely from the
  large safe pool (TN = 716), while it **missed nearly every actual vulnerability**.
- **F1 exposes the failure.** Because recall is 0.07, F1 collapses to 0.13 — an honest picture.
- **In security, recall is king.** A **false negative** (missed vulnerability) can be exploited;
  a **false positive** (false alarm) just wastes a reviewer's time. So we optimize for catching
  vulnerabilities, using F1 (and MCC) as balanced summaries.

> **Part 1 in one line:** *Never trust accuracy on imbalanced security data — measure recall
> and F1, because missing a vulnerability is the costly error.* This lesson directly shapes how
> Parts 2 and 3 are built and judged.

---

## 3. Part 2 — Build an honest dataset

### 3.1 The task

Part 1 used someone else's data. Part 2 is: **create your own vulnerability dataset** from
**SAP ProjectKB** — an open knowledge base that links each CVE to the Git commit(s) that fixed
it, for open-source **Java** software.

The core idea: a fix commit is a labelled example generator. For each method a fix changed:

- the version **before** the commit is **vulnerable** (`1`),
- the version **after** the commit is **fixed / not vulnerable** (`0`).

### 3.2 The pipeline that worked (four tasks)

1. **Clone ProjectKB and find the CVE statements.** They live on its `vulnerability-data`
   branch as `statements/<CVE-ID>/statement.yaml`. *(YAML is a simple human-readable data
   format.)*
2. **Parse every statement** to collect *(CVE id, repository URL, fixing commit hash)*.
   Entries hosted on **SVN** (a different version-control system) are skipped, because the
   mining tool only reads Git.
3. **Extract only the methods each fix actually changed**, using **PyDriller** (a library that
   walks a Git repo's commit history) and **lizard** (a tool that locates function/method
   boundaries in source code). Two important, defensible decisions here:
   - **Java-only, tests excluded.** *Why:* the model is evaluated on Java, so training data
     must match that language (matching training and evaluation distributions is a core ML
     principle).
   - **Merge-aware mining.** A **merge commit** (a commit with two parents that joins branches)
     returns no changed files by default, yet ProjectKB often records the *merge* as the fix.
     For those we diff the merge against its **first parent** to recover the real changes,
     with a guard (`MERGE_MAX_FILES=10`) to skip giant branch-integration merges that aren't
     genuinely targeted fixes.
4. **Clean, deduplicate, and save** as `source` + `labels` in **HDF5** (same schema as Part 1)
   plus a metadata CSV (CVE id, file path, etc.). The mining loop is **checkpointed** so the
   long run can be interrupted and resumed, and genuine fetch failures are retried instead of
   silently dropped.

### 3.3 The decision that made it honest: **no padding**

The tempting shortcut is to pad the dataset with lots of random *unchanged* methods labelled
"not vulnerable" to get more data. **We deliberately did not.** *Why:* an unchanged method is
**not evidence of non-vulnerability** — it might be vulnerable and simply never fixed. Padding
with it injects **label noise** and fakes the class balance. (An earlier version had 33,035
samples that were **82% synthetic padding**; we dropped it.)

The honest result: **6,530 real methods, ~51% vulnerable** (3,331 vulnerable / 3,199 fixed) —
every sample a method genuinely touched by a CVE-fixing commit.

> **Part 2 in one line:** *Build the dataset from real security fixes as before/after method
> pairs; refuse fake padding, because data honesty beats data quantity.* (This same principle —
> data construction matters most — becomes the whole punchline of Part 3.)

---

## 4. Part 3 — Train our own model (the challenge win)

Now we train a model that reads **one Java method** and outputs a vulnerability probability,
then submit predictions on a hidden **1,000-method challenge set** (ranked on F1). This part
contains the project's key insight, so it gets the most detail.

### 4.1 Why the "obvious" model fails — the most important idea in the project

The obvious approach is **pointwise classification**: show one method, ask "vulnerable?
yes/no," repeat. **On the Part-2 data it scores at chance (ROC-AUC ≈ 0.50)** — a coin flip.
You must be able to explain *why*; it's the exam's favourite question.

**The reason:** in the Part-2 data, every "not vulnerable" example is the **fixed twin** of a
vulnerable example — the same method minus a tiny patch. We measured their similarity: the
vulnerable and fixed versions of the same method have an embedding **cosine similarity of
0.995** (1.0 = identical; random unrelated methods ≈ 0.954).

> **Cosine similarity** measures how aligned two vectors point: 1 = same meaning, 0 =
> unrelated. **0.995 means the two classes are almost the same point in the model's mind — but
> carry opposite labels.**

So a pointwise model is asked: "here are two nearly identical snippets; call one dangerous and
the other safe." With nothing else to go on, it can't, and it parks at chance. **This is
structural** — a property of how the labels were defined, not a bug you can tune away.
(Confirmed in the literature: *Chakraborty et al., IEEE TSE 2021* — **data construction
dominates model architecture** for vulnerability detection.)

### 4.2 The three-idea arc (memorize the story)

**Idea A — Contrastive pair-ranking (first thing to beat chance).** Instead of hiding the
pairs, *exploit* them: train so that `score(vulnerable) > score(fixed) + margin` for each
matched pair (a **margin-ranking loss**). This cancels everything the two versions share and
forces the model onto the one difference — **the patch** (the added guard). It lifted ROC-AUC
0.50 → ~0.55–0.59: a real signal, the first to beat chance. **But** on the leaderboard it only
reached **F1 ≈ 0.40**, because it learned **"pre-patch vs post-patch"** — still not the real
task.

**Idea B — Rebuild the dataset (this won).** We inspected the challenge set and found:
- 36 of 1,000 challenge methods exactly match Part-2 methods, and **35 of those 36 are the
  *vulnerable* version** — their fixed twins are absent.
- The leaderboard math implies the challenge is **~30% vulnerable**, and its negatives are
  **ordinary, unrelated methods**, not post-fix twins.

Conclusion: **the real task is "vulnerable vs. ordinary code," not "pre-patch vs.
post-patch."** Our whole dataset was training for the wrong comparison. So we **rebuilt the
data** and returned to plain pointwise classification (which now works, because the classes are
no longer near-identical twins). That is the winning approach — detailed next.

### 4.3 The winning approach, decision by decision

#### (a) Dataset v2 — three kinds of examples

We assembled **14,887 methods, 20.4% vulnerable** (270 repos, 691 CVEs):

| Type | What it is | Count | Why it's included |
|------|-----------|------:|-------------------|
| **Positives** | Vulnerable (pre-fix) methods | 3,037 | What we want to catch. |
| **Hard negatives** | The post-fix twins | 2,902 | Teach that *the guard matters* — the fixed version is safe. "Hard" = looks almost like a positive. |
| **Easy negatives** | Ordinary unchanged methods from the same repos | 8,948 | **The fix.** They represent "normal code" — which is what the challenge's negatives actually are. |

Three deliberate decisions:
1. **Restore easy negatives** — 8,948 real unchanged Java methods from the same repositories
   (77% from files a fix commit touched, so realistic neighbours of vulnerable code). They make
   "not vulnerable" mean *ordinary code*, matching the challenge.
2. **Length-match** the easy negatives to the positives' line-count distribution. *Why:* if
   vulnerable methods were longer on average, the model could cheat with "long ⇒ vulnerable"
   instead of learning security. Matching lengths removes that shortcut (and we prove it with a
   length-only baseline in §4.4).
3. **Deduplicate and drop label noise** — removed 568 "pairs" whose fix changed only
   whitespace (after normalization the two versions are identical, so keeping both with opposite
   labels is pure contradiction).

**Imbalance:** 20.4% vulnerable → we use a class-weighted loss and PR-AUC as headline metric.

#### (b) The split — no leakage allowed

Divide into **train 70% / validation 15% / test 15%**:
- **Train:** the model learns from these.
- **Validation:** held out; used to *make decisions* (when to stop, what threshold) — never
  trained on.
- **Test:** touched once at the end, to estimate real performance.

Critical decision: split **grouped by repository** (`StratifiedGroupKFold` on `repo_url`).
*Why:* if methods from one project appeared in both train and test, the model could win by
memorizing that project's coding style — **data leakage** (cheating with info unavailable in
the real task). Grouping by repo guarantees **zero project overlap**, so the test score reflects
genuine generalization to *new codebases*.

> **Data leakage** = test information sneaking into training, inflating scores dishonestly.
> Grouped splitting is the standard defense when data has natural clusters (here, repos).

#### (c) The model — UniXcoder + 16 static features

Two inputs feed one small classifier:

- **UniXcoder encoder** (`microsoft/unixcoder-base`): a code language model pre-trained with
  awareness of **AST** (the grammatical tree structure of code) and **data-flow** (how values
  move through variables) — exactly where injection/taint/bounds vulnerabilities live. We take
  its **CLS vector** (the 768-number summary of the method).
- **16 static security features**: interpretable counts computed with **regex** (text pattern
  matching) — dangerous sinks (`exec`/`Runtime`/`ProcessBuilder`, SQL concat, deserialization,
  reflection), file ops, weak crypto, input sources, null-guards, bounds-checks, sanitizer
  calls, loops, conditionals, try/catch, string-concat, line count. *Why add hand-made features
  on top of a neural net?* They inject explicit security knowledge and are **interpretable**
  (you can point at "calls `Runtime.exec` twice"). Counts are `log1p`-transformed and
  standardized using **train-only** statistics (no leakage).

**Combine:** concatenate the 16 features onto the 768-dim CLS vector → small **classifier head**
(linear → dropout 0.3 → one logit).

**Anti-overfitting choices (Task 3 asks about these):**
- **Freeze the bottom 8 encoder layers** + embeddings → far fewer trainable parameters to
  memorize with.
- **Dropout 0.3** → randomly zero 30% of head activations each step, forcing redundancy.
- **Weight decay 0.01** → gently pulls weights toward zero, discouraging over-complex fits.
- **Early stopping** → keep the weights from the best-*validation* epoch, not the last.

#### (d) Training — the loss and stopping rule

- **Loss: class-weighted BCE.** BCE is the standard yes/no loss; **class-weighted** multiplies
  the loss on the rare positives by ~4 (`pos_weight ≈ 4`) so the model can't ignore the 20%
  minority. *(We also tried adding the Idea-A pair-ranking term — "Config B" — but on dataset v2
  it **hurt** (val PR-AUC 0.271 vs 0.290), so plain class-weighted BCE won. A clean "test it,
  don't assume" moment.)*
- **Optimizer: AdamW, learning rate 2e-5, 10% warmup.** The optimizer nudges weights to reduce
  loss; `2e-5` is a small, safe step for fine-tuning a pre-trained model (big steps would wreck
  its knowledge); warmup ramps the rate up gradually for stability.
- **Early stop on validation PR-AUC**, and **operating threshold = validation max-F1** (sweep
  cutoffs on val, pick the one maximizing F1, then fix it for the challenge — choosing the
  threshold from data, not by hand).

### 4.4 Results and honesty checks

**Leaderboard (the real scoreboard): F1 62.3%, precision 55.1%, recall 71.7%, accuracy 74.0%
→ Rank #1.** The previous best (Idea A, old data) was F1 ≈ 0.40 — so the **dataset rebuild
added ~22 F1 points with the same model family**: concrete proof that the *data*, not the
architecture, was the bottleneck. (An "always vulnerable" baseline on a ~30%-positive set scores
F1 ≈ 0.46; beating it by ~16 points shows the signal is genuine, not a threshold trick.)

Why we trust the number:
- **Length-only baseline:** ROC-AUC 0.506 — *chance*. Proves length-matching worked; the signal
  is **not** "long ⇒ vulnerable."
- **Train→test gap:** ROC-AUC 0.736 → 0.592 — moderate overfitting, but clear generalization
  (nowhere near memorization).
- **FP rate by negative type:** the model wrongly flags **62%** of hard negatives (post-fix
  twins) but only **48%** of easy negatives — twins stay nearly inseparable, and the challenge
  has almost none of them, which is *why* the internal test F1 (0.36) badly *understates* the
  true challenge F1 (0.62): the internal test set is deliberately harder than reality.

Where it still fails (good exam material):
- **Top false positives** are hard negatives from **XXE-style fixes** — the *fixed* code still
  contains the scary factory patterns (`XMLInputFactory`, `DocumentBuilderFactory`,
  `SAXParserFactory`), so the model reasonably stays suspicious.
- **Top false negatives** are trivial 3–6-line positives (getters/delegates) — **label noise**:
  file-level CVE labelling marks *every* touched method "vulnerable," even ones with no security
  content, so the model is "wrong" only against a noisy label.

> **Part 3 in one line:** *Pointwise fails because "vulnerable vs its own fixed twin" is
> unlearnable and not the real task; rebuilding the data to "vulnerable vs ordinary code" — then
> training a UniXcoder+static-features classifier under a leakage-free repo-grouped split — wins
> (F1 62.3%). Data construction dominates model architecture.*

---

## 5. Exam cheat-sheet (one-line answers)

- **Why is accuracy a bad metric here?** The data is imbalanced (~28% vulnerable in Part 1, 20%
  in Part 3); always-guess-"safe" already scores high accuracy while catching zero
  vulnerabilities. Use **F1 / recall / MCC**.
- **Which error is worse in security, FP or FN?** **FN** (a missed vulnerability can be
  exploited); a FP just costs review time. So prioritize **recall**.
- **Why no padding in Part 2?** An unchanged method isn't proof of non-vulnerability; padding
  injects label noise and fakes the balance. Keep only real fix-commit pairs.
- **Why does pointwise classification fail in Part 3?** Vulnerable and fixed twins are
  near-identical (cosine 0.995) with opposite labels — structurally unlearnable, and it's not
  the deployment task anyway.
- **What actually fixed Part 3?** Rebuilding the dataset so negatives are *ordinary code*
  (matching the challenge), not post-fix twins — then plain class-weighted BCE on a
  UniXcoder+static hybrid.
- **How do you prevent data leakage?** Split **grouped by repository** so no project appears in
  both train and test.
- **How do you reduce overfitting?** Freeze lower encoder layers, dropout, weight decay, early
  stopping on validation.
- **Overarching lesson of the whole project:** *In vulnerability detection, how you construct
  and grade the data matters more than which model you pick.*

---

## 6. Glossary index

| Term | One-line meaning | Section |
|------|------------------|---------|
| CVE | Public ID of one known vulnerability | §1.1 |
| Sink / sanitizer | Dangerous op / the guard that makes input safe | §1.1 |
| Fix commit / pair | Code change that removes a vuln → before/after method pair | §1.1, §3.1 |
| HDF5 | Binary format storing the code + labels dataset | §1.2 |
| Embedding / CLS vector | Numeric summary of a method from the encoder | §1.2, §4.3c |
| Logit / sigmoid | Raw score / squash-to-probability function | §1.2 |
| TP/FP/FN/TN | Confusion-matrix outcomes | §1.3 |
| Accuracy / precision / recall / F1 | Fraction correct / few false alarms / few misses / balance | §1.3, §2.2 |
| PR-AUC vs ROC-AUC | Imbalance-aware vs threshold-free ranking score | §1.3 |
| MCC | Imbalance-robust single-number score | §1.3 |
| Padding (and why we refuse it) | Fake unchanged negatives that add label noise | §3.3 |
| Merge-aware mining | Recovering changes from two-parent merge commits | §3.2 |
| Pointwise classification | "Is this one method vulnerable?" — fails on twins | §4.1 |
| Cosine similarity | Vector alignment; twins = 0.995 | §4.1 |
| Contrastive / margin-ranking | Train score(vuln) > score(fixed)+margin | §4.2 |
| Hard vs easy negative | Post-fix twin vs ordinary unrelated method | §4.3a |
| Data leakage / repo-grouped split | Test info leaking in / the defense against it | §4.3b |
| Static features | 16 regex counts of security-relevant patterns | §4.3c |
| Freeze / dropout / weight decay / early stop | Anti-overfitting levers | §4.3c–d |
| Class-weighted BCE | Yes/no loss reweighted for imbalance | §4.3d |
| Threshold | Probability cutoff for "vulnerable"; tuned on val | §4.3d |

---

*The complete Part-3 experiment history (every approach tried, including the failures, with
exact numbers) lives in `PART3_TRAINING_APPROACHES.md`. This document is the beginner-facing
companion covering all three parts and only the approaches that worked.*
