# Part 3 — Training Approaches Log

A living record of every modelling approach tried for **single-method vulnerability
prediction** on the Part-2 dataset (6,530 Java methods, ~51 % vulnerable, balanced
before/after CVE-fix pairs). **Append new approaches here as they are tried — do not delete past ones.**

## Common setup
- **Data:** `data/vulnerability_dataset.hdf5` (`source`, `labels`) + `data/vulnerability_dataset_metadata.csv` (`cve_id`, …). HDF5↔CSV rows verified aligned.
- **Split:** 70/15/15 **grouped by `cve_id`** via `StratifiedGroupKFold` (no CVE crosses splits → leakage-free). Random split shown for comparison only.
- **Metric focus:** ROC-AUC (threshold-independent) + F1/recall (recall = security-critical). Chance ROC-AUC = 0.50.
- **Hardware:** MacBook M4 Pro, PyTorch **MPS** backend.
- **Encoder:** `microsoft/codebert-base` unless noted.

## Summary table

| # | Approach | Key hyper-params | Split | Test/CV ROC-AUC | Verdict |
|---|----------|------------------|-------|-----------------|---------|
| 1 | Frozen CodeBERT (mean-pool) + MLP head | 768→256→1, dropout 0.3, Adam 1e-3 | grouped | ~0.50 (val) | ✗ chance |
| 2 | Frozen CodeBERT + LogReg linear probe | C=0.1, mean-centered | grouped 5-fold | **0.535 ± 0.014** | ✗ chance |
| 3 | Frozen CodeBERT + LogReg linear probe | C=0.1 | random 5-fold | **0.500 ± 0.010** | ✗ chance (leakage doesn't help) |
| 4 | Full fine-tune CodeBERT (pointwise BCE) | lr 2e-5, 3–4 ep, bs 16, max_len 256, warmup 0.1 | grouped | val 0.516–0.536 | ✗ chance, train loss stuck ~0.69 |
| 5 | Partial fine-tune (bottom 8 layers frozen) | lr 3e-5, 3 ep, bs 16, max_len 256 | grouped | not run (superseded) | — designed as notebook Config B; pivoted before running |
| 6 | **Contrastive pairwise ranking (CodeBERT)** | margin 1.0 + 0.3·BCE, lr 2e-5, pair-bs 8, max_len 256, 4 ep | grouped | **test 0.555 / val 0.587** | ✓ **WORKS** — first above-chance, monotonic lift, loss ↓ |
| 7 | **UniXcoder + static-feature hybrid, contrastive ranking** | UniXcoder-base, bottom-8 frozen, +16 static feats on CLS, head 256+dropout 0.3, margin 1.0 + 0.3·BCE, lr 2e-5, early-stop on grouped val AUC, grouped 5-fold CV | grouped 5-fold | **~0.55–0.60** (cloud run); **challenge leaderboard F1 ≈ 0.40** | ✗ ran; no real lift over #6 — model-side levers exhausted |
| 8 | **Dataset v2 + pointwise BCE (± auxiliary pair loss)** | rebuilt data: pos + post-fix hard negs + ~9k length-matched same-repo easy negs (~15k @ 20% vuln); UniXcoder hybrid unchanged; class-weighted BCE (Config A) vs BCE + 0.3·pair margin-rank (Config B); repo-grouped split; early-stop on val PR-AUC; threshold = val max-F1 | repo-grouped | **challenge leaderboard: F1 62.3%, P 55.1%, R 71.7%, acc 74.0% — Rank #1** (vs 40% for #7) | ✓ **WORKS** — the data-side fix pays off |
| 9 | **Challenge-aligned inference stack** (proxy-val selection + duplicate override + 5-fold ensemble + max_len 512) | model = #8 Config A unchanged; early-stop & threshold on an *easy-neg-only, ~30%-positive* proxy val; near-dup kNN label override on challenge; average 5 fold-models' probabilities; retrain once at max_len 512 | repo-grouped | — (proposed, not yet run) | ⏳ **PROPOSED** — see detailed log |

> **Headline finding (updated):** *Pointwise* single-method classification (frozen **or** fine-tuned
> CodeBERT) sits at **≈ chance (ROC-AUC ~0.5)** under a leakage-free split — because the label is
> defined *relative to a fix commit* and the vulnerable/fixed methods are near-identical (embedding
> cosine **0.995**), so an isolated method carries almost no *pointwise* signal (cf. Chakraborty et
> al., IEEE TSE 2021).
>
> **The fix that worked (Approach 6):** reframe the problem to **exploit the pairing** with a
> **contrastive margin-ranking loss** — train so `score(vulnerable) > score(fixed) + margin` for each
> matched before/after pair. This removes all per-method confounders, forces the encoder onto the
> actual patch region (added guards / sanitizers / bounds checks), and **lifts held-out ROC-AUC above
> chance (test ≈ 0.55, val ≈ 0.59) with a monotonically decreasing loss**. Inference still scores a
> single method. The realistic ceiling for this hard task is modest (~0.55–0.60 AUC), but it is a
> genuine, reproducible signal — unlike every pointwise approach.
>
> **Headline finding v2 (2026-07-02, after the Approach-7 run + 40% challenge F1):** the ~0.58 AUC
> ceiling is a property of the *dataset*, not the models — pure before/after pairs train
> "pre-patch vs post-patch", while the challenge (and reality) is "vulnerable vs ordinary code".
> **Approach 8** rebuilds the data (same-repo length-matched easy negatives restored from the
> archived Part-2 run, ~15k methods @ 20% vulnerable) and returns to pointwise training, keeping
> the pair-ranking loss only as an auxiliary term. Data construction dominates architecture
> (Chakraborty et al., IEEE TSE 2021).

---

## Detailed log

### Approach 1 — Frozen CodeBERT embeddings + MLP classifier head
- **Idea:** run CodeBERT once (frozen), mean-pool token states → 768-d vector per method, cache to `data/codebert_embeddings.npy`, train a small MLP head. Cheap to iterate on MPS.
- **Config A head:** 768→256→1, ReLU, dropout 0.3, BCEWithLogits, Adam lr 1e-3, wd 1e-4.
- **Config B head:** 768→512→128→1, dropout 0.5, lr 5e-4, wd 1e-3.
- **Result:** train loss → 0 (memorises 1000-sample smoke set) but **val/test ROC-AUC ≈ 0.50**; full-train head stuck at loss 0.693 (random). Predicts almost everything "vulnerable".
- **Why it fails:** see Approach 2/diagnostic — frozen embeddings of vuln vs fixed pairs collide.

### Approach 2 & 3 — Frozen-embedding linear probe (grouped vs random)
- **Idea:** sanity-check whether *any* separable signal exists in frozen features, independent of the head.
- **Result:** LogReg AUC **0.535 ± 0.014** (grouped) and **0.500 ± 0.010** (random).
- **Pair diagnostic:** mean cosine similarity of the vulnerable vs fixed version of the *same* method = **0.995** (median 0.999) over ~2,934 pairs; random-pair baseline 0.954. → The two classes are essentially the same point in frozen space.
- **Conclusion:** frozen encoder is provably inadequate; motivates fine-tuning. Random split is *not* easier — leaked pairs carry opposite labels.

### Approach 4 — Full fine-tune of CodeBERT (end-to-end)
- **Idea:** unfreeze the encoder so it can learn to amplify the small vuln-vs-fix differences. Decided after Approach 1–3.
- **Config:** `RobertaForSequenceClassification` (num_labels=2), AdamW lr 2e-5, wd 0.01, batch 16, max_len 256, linear warmup 10 %, grad-clip 1.0, 3–4 epochs. ~340 s/epoch on M4 Pro.
- **Result (4 261 train / 1 004 val):**
  | epoch | train_loss | val_loss | val_auc | val_f1 |
  |---|---|---|---|---|
  | 0 | 0.7003 | 0.6924 | 0.536 | 0.672 |
  | 1 | 0.6961 | 0.6935 | 0.516 | 0.331 |
  | 2 | 0.6945 | 0.6956 | 0.532 | 0.661 |
- **Conclusion:** train loss barely moves (under-fitting, not over-fitting); val AUC stays ~chance. Even fine-tuning cannot extract signal from a single isolated method.

### Approach 5 — Partial fine-tune (freeze bottom 8 encoder layers)
- **Idea:** Config B for the notebook comparison — fewer trainable params (top 4 layers + head), lr 3e-5. Tests capacity/regularisation effect.
- **Status:** *designed but not run to completion* — superseded by Approach 6 once it was clear the bottleneck is the **pointwise framing**, not model capacity (Approach 4 already showed capacity isn't the limiter). Kept here for honesty / future reference.

### Approach 6 — Contrastive pairwise ranking ✓ (the approach that worked)
- **Diagnosis driving it:** the failure of Approaches 1–4 is *structural*. Each "vulnerable" method and its "fixed" sibling are the same code minus a tiny security patch, with **opposite labels**. A *pointwise* loss ("is this one method vulnerable?") makes the model fight near-identical, contradictory examples → it parks at chance. **But the patch that flips the label is exactly the signal**, and security fixes share recurring shapes (added bounds/null checks, sanitisers) that generalise across CVEs.
- **Idea:** stop hiding the pairs. Train CodeBERT with a **`MarginRankingLoss`** over matched `(vulnerable, fixed)` pairs so `score(vuln) > score(fixed) + margin` (score = `logit_vuln − logit_fixed`), plus a small BCE term (weight 0.3) for threshold calibration. This cancels every per-method confounder and points the gradient straight at the patch region. **Inference is unchanged** — score a single method; the grouped split keeps both halves of a training pair in the same split, so no leakage.
- **Pairs:** group by `cve_id|file_path|method` → 2,934 matched pairs total, **1,911 in the training split**.
- **Config:** `RobertaForSequenceClassification`, AdamW lr 2e-5, wd 0.01, pair-batch 8 (=16 methods/step), max_len 256, linear warmup 10 %, grad-clip 1.0, margin 1.0, BCE weight 0.3.
- **Result (PoC, 4 261 train / 1 004 val / 1 265 test):**
  | epoch | loss | val_auc | test_auc | test_f1 |
  |---|---|---|---|---|
  | 0 (warmup) | 1.420 | 0.533 | 0.516 | 0.644 |
  | 1 | 1.305 | 0.576 | 0.536 | 0.572 |
  | 2 | 1.144 | 0.584 | 0.553 | 0.607 |
  | 3 | 1.017 | **0.587** | **0.555** | 0.586 |
- **Verdict:** ✓ **First approach to beat chance.** Loss decreases monotonically (the model actually learns, vs. the stuck ~0.69 of pointwise) and held-out AUC climbs steadily. Modest absolute number (~0.55–0.59 AUC) but a *real, reproducible* signal — and it is the methodologically correct framing for paired before/after data. **This is the approach the final notebook is built around** (pointwise baseline kept as the comparison that shows the lift).

### Approach 7 — UniXcoder + static-feature hybrid, contrastive ranking ⏳ (built for cloud, not yet run)
- **Idea:** keep the proven contrastive margin-ranking core (Approach 6) but raise the realistic ceiling with the two levers that genuinely add signal — (a) swap the backbone CodeBERT → **`microsoft/unixcoder-base`** (data-flow/AST-aware pretraining, where injection/taint/bounds vulns live), and (b) build a **hybrid head**: concatenate **16 interpretable static security features** to the encoder CLS vector before the classifier. Static features (regex over Java source, `log1p` + train-only `StandardScaler`): dangerous-sink counts (`exec`/`Runtime`/`ProcessBuilder`, SQL string-concat, deserialize, reflection), file ops, weak crypto, input sources, null guards, bounds checks, sanitizer calls, loops, conditionals, try/catch, string-concat, n_lines. Verified to fire on the real data (null-guards 46 %, conditionals 64 %, sanitizers 12 % nonzero) and class-mean diffs point the right way (fixes add sanitizers/conditionals).
- **Anti-overfitting (baked in):** freeze embeddings + bottom 8 encoder layers (few trainable params), dropout 0.3 on head, weight decay 0.01, lr 2e-5 + 10 % warmup, **early-stop on grouped val AUC (keep best weights)**.
- **Task-3 "change parameters & compare" = static-feature ablation:** Config A pointwise baseline (≈chance) vs **B-enc** (contrastive, encoder only) vs **B-hybrid** (contrastive + static). Same architecture, isolates the contribution of the static features.
- **Final reporting:** **grouped 5-fold CV** (per fold: held-out test = fold; small grouped val carved from train for early-stop; static scaler refit on fold-train → no leakage) → **mean ± std** for stability, plus train↔test F1 gap.
- **Where:** primary solution in `CDS_project_part3.ipynb` (Cloud/Colab T4; single split ~few min, 5-fold CV ~20–25 min). **No local Mac training** (per project preference).
- **Status:** notebook built & statically validated (all cells parse, rendered regex extractor reproduces the 16-feature matrix). Awaiting a GPU run to fill in numbers; expected ~0.62–0.68 grouped AUC (vs ~0.55 for Approach 6).
- **Result (cloud run, recorded 2026-07-02):** grouped ROC-AUC **~0.55–0.60** — no material lift over Approach 6. A `submission.csv` at threshold 0.45 scored **F1 ≈ 0.40 on the challenge leaderboard**.
- **Post-mortem:** seven model-side variations (frozen/fine-tuned, CodeBERT/UniXcoder, pointwise/contrastive, ±static features) moved AUC only 0.50 → ~0.58. The bottleneck is not the model — it is the **data construction** (see Approach 8).

### Approach 8 — Dataset v2 (same-repo easy negatives) + pointwise BCE, pair loss as ablation ⏳ (built, awaiting cloud run)
- **Diagnosis driving it (evidence, not hypothesis):**
  - The Part-2 dataset defines "not vulnerable" as *the fixed version of the same method* (cosine ≈ 0.995). That trains "pre-patch vs post-patch" — the hardest possible discrimination and **not the deployment task**.
  - The challenge set is a different distribution: 36/1000 challenge functions exactly match Part-2 rows (whitespace-normalized) and **35 of 36 are the vulnerable version** while their fixed twins are absent → challenge negatives are *ordinary methods*, not post-fix twins.
  - The instructors' Part-1 lab dataset confirms the construction style: 28.3% vulnerable, negatives unrelated to positives (median token-Jaccard ≈ 0.11). (It is C code, so unusable directly for training, but the sampling recipe is the tell.)
  - The 40% leaderboard F1 is reproduced by the arithmetic of a near-chance ranker at threshold 0.45 on a ~30%-positive set — i.e. mostly a task/threshold mismatch, not merely a weak model.
- **The fix — rebuild the data (dataset v2):** keep the pairs, restore the archived same-repo negatives (`data/archive_20260625_204235/` = the original Part-2 run's "padding": 26,707 real unchanged Java methods from the same repos/files, 254 repos / 625 CVEs, 77% from files touched by fix commits).
  - Composition: **3,037 pos + 2,902 hard negs (post-fix twins) + 8,948 easy negs = 14,887 methods @ 20.4% vulnerable**.
  - Easy negatives **length-matched** to the positives' line-count distribution (medians: pos 18 / hard 19 / easy 15) so "long method ⇒ vulnerable" cannot be a shortcut; a length-only baseline is reported in the notebook as an honesty check.
  - Deduped by whitespace-normalized code; **568 contradictory pair rows dropped** (whitespace-only "fixes" where vuln == fixed after normalisation — pure label noise that polluted Approaches 1–7).
  - Files: `data/vulnerability_dataset_v2.hdf5` + `_v2_metadata.csv` (adds `neg_type` ∈ pos/hard/easy); build cell inside the notebook (pandas-only, seconds, runs locally).
- **Split:** 70/15/15 via `StratifiedGroupKFold` grouped by **`repo_url`** — strictly stronger than CVE grouping (also blocks project-idiom recognition). Verified: train 9,576 / val 2,469 / test 2,842, zero repo overlap, ~20% vulnerable each.
- **Model & training:** UniXcoder hybrid unchanged (bottom-8 frozen, 16 static features, dropout 0.3). **Config A** = class-weighted BCE (pos_weight ≈ 4). **Config B** = A + 0.3·MarginRankingLoss over the 1,664 matched train pairs (the Approach-6 insight kept as a *component*: pairs sharpen the boundary on hard negatives). Early stop on grouped **val PR-AUC**; operating threshold = **val max-F1** (replaces the hardcoded 0.45); final model = better val PR-AUC of A/B.
- **Evaluation:** PR-AUC/F1-centred (imbalance-aware; PR-AUC chance = 0.20), confusion matrix, train↔test gap, length-only baseline, **error analysis** (top FP/FN, FP-rate hard vs easy negatives, metrics by length bin), optional repo-grouped 5-fold CV.
- **Status:** notebook reworked in place (`CDS_project_part3.ipynb`, 23 cells), dataset built with all assertions passing, full pipeline smoke-tested locally through one forward pass (no local training, per project preference). Trained on Colab T4; submitted at the val-chosen threshold.
- **Result (challenge leaderboard, 2026-07-02): F1 62.3% | precision 55.1% | recall 71.7% | accuracy 74.0% — Rank #1** (previous submission, Approach 7: F1 ≈ 0.40).
- **Post-hoc confirmation of the diagnosis:** the four leaderboard numbers pin down the challenge composition — solving accuracy = 1 − 0.867·π gives **π ≈ 30% vulnerable (~300/1000)**, matching the predicted lab-set-style construction (28.3%). The all-1s baseline on that base rate would score F1 ≈ 0.46; the model beats it by ~16 points, i.e. the signal is genuine, not a threshold artifact.
- **Internal Colab numbers (recorded 2026-07-07):**
  - Data as loaded: 14,887 methods, 20.4% vuln, 270 repos, 691 CVEs. Split: train 9,576 / val 2,469 / test 2,842, zero repo overlap.
  - **Config A (class-weighted BCE) won:** val PR-AUC 0.290 (epochs: 0.270 → 0.286 → 0.290 → 0.289) vs **Config B (BCE + 0.3·pair loss)** 0.271 (early-stopped @ ep 3). The auxiliary pair loss *hurt* on dataset v2.
  - **Test (each at its own val-max-F1 threshold):** A: acc 0.514 / P 0.253 / R 0.621 / **F1 0.359 / PR-AUC 0.307 / ROC-AUC 0.592**; B: acc 0.587 / P 0.267 / R 0.506 / F1 0.349 / PR-AUC 0.304 / ROC-AUC 0.567. (PR-AUC chance ≈ 0.22 on this test split.)
  - **Train↔test gap (final A):** PR-AUC 0.390 → 0.307 (gap 0.083), ROC-AUC 0.736 → 0.592 (gap 0.144), F1 0.436 → 0.359 — moderate overfit, not memorisation.
  - **Length-only baseline (test): ROC-AUC 0.506 / PR-AUC 0.253** — near chance, so the length-matching worked; the model's signal is not "long ⇒ vuln".
  - **FP rate by negative type (test): hard 62.2% vs easy 47.7%** — post-fix twins remain largely inseparable, and even the same-repo easy negatives (77% from fix-touched files) are much harder than the challenge's ordinary-code negatives. This is why internal test F1 (0.359) wildly *understates* challenge F1 (0.623): the internal test set is intentionally harder than the deployment distribution.
  - **By length bin (test):** recall rises 0.40 → 0.87 and precision stays ~0.19–0.31 from short to 40+-line methods — the model over-predicts on long methods (and 40+ lines exceeds max_len 256 truncation).
  - **Error analysis:** top FPs are all *hard* negatives from XXE-style fixes (post-fix `XMLInputFactory`/`DocumentBuilderFactory`/`SAXParserFactory` creation, CVE-2019-3772/3773, CVE-2018-11758…) — the fixed code still contains the scary factory pattern. Top FNs are trivial 3–6-line positives (getters/delegates from CVE-2018-1131, CVE-2012-0881…) — file-level CVE labelling marks every touched method "vulnerable", so many positives are label noise with no security content.
  - **Submission:** threshold 0.343 (val max-F1, val F1 0.356) → 390/1000 challenge methods predicted vulnerable (39.0% share vs estimated π ≈ 30%).

### Approach 9 — Challenge-aligned inference stack 📝 (proposed 2026-07-07, not yet implemented)

**Goal:** push challenge F1 from **62.3%** toward **~68–72%** *without inventing a new model*. Approach 8's
error analysis shows the remaining loss is concentrated in three places that are all *around* the model,
not inside it: (a) model selection and thresholding happen on a val distribution the challenge doesn't have,
(b) the classifier is asked to re-derive labels we already possess exactly for ~3.5% of the challenge set,
and (c) single-split single-model inference throws away variance reduction and truncates long methods.
Precision (55.1%) is the weak side of the leaderboard F1 — every component below is chosen to buy precision
at minimal recall cost.

**Diagnosis driving it (all evidence from the recorded Approach-8 run):**
- **Selection/deployment mismatch.** Early stopping and the operating threshold both use the full internal
  val set, which is ~19% *hard* post-fix twins (62.2% FP rate) and whose easy negatives are themselves
  adversarial (same repo, 77% from fix-touched files, 47.7% FP rate). The challenge negatives are ordinary
  methods. Internal test F1 0.359 vs challenge F1 0.623 proves the internal sets *understate* the model on
  the deployment distribution — so the val-max-F1 threshold (0.343) is calibrated for the wrong world.
- **Over-firing.** Submitted predicted-positive share = 39.0% against an estimated challenge base rate
  π ≈ 30%. Back-solving the leaderboard numbers: TP ≈ 215, FP ≈ 175, FN ≈ 85. Trimming the lowest-confidence
  positives converts FPs to TNs roughly 2:1 vs recall lost while the ranking holds.
- **Free labels ignored.** 36/1000 challenge methods are whitespace-exact matches to Part-2 rows; **35 are
  the vulnerable version**. Several recorded top-FNs are precisely such trivial-looking vulnerable methods
  (3-line getters, CVE-2018-1131) the classifier scores p ≈ 0.03 — unrecoverable by any threshold.
- **Truncation.** The 40+-line test bin runs at recall 0.87 / precision 0.31; those methods exceed
  max_len 256, so the model sees a fragment and over-predicts.
- **Single model.** The notebook already contains repo-grouped 5-fold CV machinery; only one split's model
  was used for the submission.

**The proposal — four stacked components (ordered by expected gain ÷ cost), then optional data ablations:**

1. **Challenge-proxy validation slice for *both* early stopping and threshold.**
   Construct `val_proxy` = all val positives + val *easy* negatives only, with easy negatives randomly
   subsampled so the slice is ~30% vulnerable (match π; keep the RNG seeded, sample once and freeze).
   Hard negatives are excluded entirely — the challenge essentially has none.
   - Early-stop each training run on **val_proxy PR-AUC** (not full-val).
   - Operating threshold = **val_proxy max-F1**, cross-checked against a *predicted-share sweep*: report
     challenge F1-proxy at thresholds giving 28/30/32/35/39% predicted-positive share so the submission
     choice is an informed pick, not a single point.
   - Cost: pandas only, no GPU. Can even be applied retroactively to saved Approach-8 probabilities if the
     per-row val/challenge scores were kept.

2. **Exact + near-duplicate label override on the challenge set.**
   Retrieval, not learning: for each challenge method, find the most similar method in the *full* labelled
   dataset v2 (train+val+test — no leakage concern, the challenge is a separate distribution and this is
   deployment-time lookup, the same trick a real security team would use).
   - Tier 1 (exact): whitespace/comment-normalized string equality → copy the label verbatim
     (≥ 36 rows, 35 known vulnerable).
   - Tier 2 (near-dup): normalized token-set Jaccard (or char-5-gram MinHash for speed); above a high
     threshold (proposed **≥ 0.90**, tuned only by inspecting matches, never on the leaderboard) → copy the
     neighbour's label; below it, keep the classifier's score untouched.
   - Guardrail: log every override with its matched source row + similarity for manual eyeballing before
     submission; expected volume ~40–80 rows.
   - Cost: CPU-only, minutes. Directly rescues the p≈0.03 false negatives no model change can reach.

3. **5-fold probability ensemble.**
   Run the existing repo-grouped 5-fold CV loop to completion (Config A recipe only — Config B's pair loss
   is *dropped*, it lost on dataset v2: val PR-AUC 0.271 vs 0.290). Score the challenge set with each of the
   5 fold-models and **average the probabilities** before thresholding. Threshold re-derived per component 1
   on out-of-fold predictions (each method's score comes from the fold that didn't train on its repo — an
   honest full-dataset estimate). Typical gain for pure compute: +1–3 F1, mostly precision (variance
   reduction kills the idiosyncratic high-confidence FPs of any single run).
   - Cost: ~5× one training run on a T4 (~2–3 h total); no new ideas, the loop exists.

4. **One retrain at max_len 512** (batch size halved, grad-accum ×2 to keep the effective batch; everything
   else identical). Compare against 256 *on val_proxy* before adopting — expected to help precision on the
   40+-line bin specifically; if val_proxy F1 doesn't improve, keep 256 and skip the 5-fold rerun at 512.

**Optional ablation pair (only if run budget allows — this is the "change parameters & compare" axis):**
- **9-abl-1, positive denoising:** drop (or weight 0.5) positives that are ≤ 4 lines *and* whose paired fix
  diff contains none of the 16 static-feature tokens — the CVE-2018-1131-style file-level label noise seen
  in the top FNs. Risk: throws away real short vulns; that's why it's an ablation, not a default.
- **9-abl-2, hard-negative downweighting:** weight hard negatives 0.5 (or train easy-only) — the 62% FP rate
  on post-fix twins suggests they distort the boundary, and the challenge contains almost none. Evidence in
  favour: Config B, which *emphasised* the pairs, lost to plain BCE. Evidence against: twins are the only
  thing teaching "the guard matters". Decide on val_proxy, not ideology.

**What deliberately stays the same:** dataset v2, repo-grouped split, UniXcoder-base hybrid with 16 static
features, bottom-8 frozen, class-weighted BCE, lr 2e-5, dropout 0.3 — Approach 8 established these; the
levers above are orthogonal to all of them.

**Evaluation & success criteria (in order, each verifiable before the next):**
1. val_proxy threshold alone (saved Approach-8 probabilities, no retrain) → predicted challenge share drops
   toward ~30–33%; submit only if val_proxy F1 ≥ current.
2. + duplicate override → every override manually inspected; expected +1–2 F1 from ~35 guaranteed TPs.
3. + 5-fold ensemble (fresh Colab run) → out-of-fold val_proxy F1 must beat the single model's.
4. + max_len 512 → adopt only on a val_proxy win.
- **Honesty checks carried over:** length-only baseline, FP-rate by negative type, train↔test gap, top-FP/FN
  dump — all re-reported on the ensemble.
- **Success:** challenge F1 ≥ 66% (stretch: 70%+). **Failure mode to watch:** if val_proxy-chosen thresholds
  disagree wildly across folds, π ≈ 30% is wrong — re-estimate from the next leaderboard feedback.

**Expected-gain arithmetic (rough, from the recorded numbers):** threshold to ~32% share: P 0.551 → ~0.60,
R 0.717 → ~0.68 ⇒ F1 ~0.64. Duplicate override: +~35 TPs ⇒ recall +~0.04 at no precision cost ⇒ ~0.66.
Ensemble + 512: +1–3 ⇒ **~0.67–0.70 plausible**; anything beyond that needs the ablations or new data ideas.

**Status:** 📝 proposed only — nothing implemented. First implementation step when picked up: check whether
the Approach-8 Colab run saved per-row val/challenge probabilities (if yes, components 1–2 need no GPU at all).

---

## Ideas not yet tried (candidates for future entries)
- **Run Approach 9** (challenge-aligned inference stack, proposed above) — components 1–2 may need no GPU at all if the Approach-8 per-row probabilities were saved.
- **Full graph / data-flow models** — GraphCodeBERT, Devign, ReVeal (explicit control/data-flow graphs; relevant for taint-style vulns).
- **Diff-based input** — feed the actual change explicitly (needs the pair at inference, so a different task framing).
- **Vulnerability-*type* (CWE) classification** among vulnerable methods (a target with stronger signal).
- **Larger / different code LMs** — CodeT5+, StarCoder embeddings (UniXcoder is already the Approach-7/8 backbone).

_When you try one of these, move it up into the Detailed log + Summary table with its result._
