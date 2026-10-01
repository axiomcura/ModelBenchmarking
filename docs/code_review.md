# Code review - September 2026

This review was done while reorganizing the repository for publication. It covers every notebook and the
helper code now in `src/model_benchmarking/`. Locations refer to the new layout (`notebooks/<step>/<notebook>`,
"Part" = the section headings inside each notebook).

**How the reorganization was checked**

* Every non-empty code cell of the original notebooks was carried over (49 of 50; the remaining cell was a list
  of file paths, now in the main README's *Data products* tables).
* An AST-level comparison of every cell before and after (ignoring comments and whitespace) shows **no code
  changes other than those listed in section 1**.
* Helper functions moved to `src/model_benchmarking/` produce **bit-identical outputs** to the original notebook
  definitions on test data (33 checks), and no cell gained an undefined name.
* The notebooks themselves could not be re-executed: they read private data from
  `gs://calico-reference-embeddings`. Saved outputs are therefore those of the original runs.

---

## 1. Changes made to the code

Only changes needed for correctness or to avoid breaking the project environment were made.

| # | Where | Change | Why | Effect on saved results |
|---|---|---|---|---|
| F1 | 4.2, Parts 2b–2d and 3 | DRUG-seq **factor** cache renamed `cached_drugseq_24_48h.parquet` → `cached_drugseq_10uM_24h_factors.parquet` | Part 1 writes **raw genes** (14,370) to the same file name, and the benchmark only downloads if the file is missing. Running the notebook in order would silently benchmark raw genes instead of the 43 factors. | None - saved outputs show the 43 factors were used. |
| F2 | 4.1 and 4.2, Part 1 | Morphology cache named after its input file instead of the fixed `cached_dino_full.parquet` | After `DINO_FULL_URI` was switched to OpenPhenom-384, the old DINO download was reused. | The saved Part 1 output is DINO (Normed), 272 features, not OpenPhenom-384 as the code states. Re-running analyzes OpenPhenom-384. |
| F3 | 0.3, Part C | Local staging file names made distinct from Part A | Part C only downloads if the file is missing, and used Part A's file names; a leftover **raw** CSV could have been saved as the **normalized** table. | Unknown - cannot be verified without the data; Part A deletes its files on success. |
| F4 | 4.2, Part 3 | Numeric dose/time columns (e.g. `concentration`) excluded from morphology features, as in the benchmark cells | Otherwise such a column would be used as an image feature when decoding a model whose table contains it. | None - the decoded model (OpenPhenom) stores metadata as text. |
| F5 | 0.1, 2.1, 4.1, 4.2 | `!pip install` lines commented out; Colab-only `%%bash` setup in 0.5 exits outside Colab; GPU check in 4.1/4.2 skipped without CUDA | `pip install --force-reinstall torch` would overwrite the uv environment; `sudo apt-get` should never run on a user's machine; the GPU check crashed on CPU-only machines. | None. |
| F6 | 3.1, 4.1, 4.2 | 19 helper functions (CKA, SVCCA, pooling, annotation, neighbourhood metrics, …) moved to `src/model_benchmarking/` and imported | One tested implementation instead of up to 11 copies. Where the original relied on the notebook's global `device`, the function takes an optional `device` argument that defaults to the same GPU-with-CPU-fallback logic. | None (bit-identical). |
| F7 | 1.3, Part 2 | Narrative comment blocks at the start and end of the code cell moved into the markdown above it | Documentation, not code. | None. |
| F8 | 0.x, 1.1, 1.2, 2.1 | `Make_Parquets.ipynb` split into one notebook per step; invalid widget metadata dropped | The original notebook failed to render on GitHub (`'state' key is missing from 'metadata.widgets'`). | None. |
| F9 | 4.1, 4.2, 0.5 | Jupyter-widget progress-bar outputs removed; installation logs cleared | They cannot be displayed outside the original session. | None. |
| F10 | `pyproject.toml` | `pandas<3` | The notebooks were run with pandas 2.2; pandas 3 changes default string dtypes (e.g. `select_dtypes(include=['object'])` in 0.1 would select nothing). | None. |
| F11 | 4.3 (replaces the draft `Morphology_vs_Drugseq_v4.ipynb`), Part 4 | InChIKeys of the 22 microtubule/spindle drugs replaced by keys looked up by name in PubChem | In the draft, 20 of 22 keys (all except colchicine and albendazole) did not match PubChem, so the drug scan could not find those drugs; its saved output shows the fallback text search being used. | The draft's Part 4 outputs are not reproducible and were not carried over. |
| F12 | 4.3, Part 3 | An axis can no longer be called "Pristine" if it is flagged as stressed (cell-count correlation) | In the draft the "pristine" test ran first and ignored the cell-count criterion. | None (cell counts were never available in the draft). |
| F13 | 4.3, Part 2 | Morphology fold enrichment reported as n/a when the background mean cosine is not positive; decoding axes computed with the exact eigendecomposition used by the benchmark | The draft divided by a non-positive mean (a meaningless negative fold) and used a randomized low-rank PCA for decoding. | Saved Part 2 output (FastICA run) may differ slightly from a re-run. |
| F14 | 4.3, Part 1-2 | Local caches named `cached_<folder>__<file>.parquet` after the source URI; plate/well and JCP identifier bridges and the whole-screen PubChem/ChEMBL annotation fallback of the draft dropped | Fixed cache names reuse another source's download; the bridges were never used (every input has an InChIKey column) and referenced lookup files the draft never built; the fallback made thousands of web requests. | None. |

---

## 2. Open issues

### Critical

**C1 - ChemPerturb-Bridge embeddings were generated with an untrained network** (0.5)
The saved run printed *"Checkpoint path not provided or not found. Initializing base LPM compound encoder."*: no
`--checkpoint` was given, so the encoder's weights are random. `jump_cp_latent_perturbation_embeddings.parquet`
is therefore a random projection of Morgan fingerprints - chemical structure, not a learned perturbation space.
Because no random seed is set in the script, re-running it would even give different values.
In addition, `load_state_dict(..., strict=False)` silently ignores weights whose names do not match, so a
supplied checkpoint could also leave layers random, and the encoder architecture is defined inline and must
match the trained model exactly.
*Action:* regenerate with trained weights; load with `strict=True` (or assert the returned missing/unexpected
keys are empty); do not use the current file as ChemPerturb-Bridge output. No saved analysis uses it, but notebook 4.3 can
select it as the anchor `cpb_latent_all` - do not benchmark against it until it is regenerated.

### High

**H1 - Nearest-neighbour analysis uses one arbitrary well per compound** (3.1, all parts)
`df[~df.index.duplicated(keep='first')]` keeps only the first well of each compound. The DINO and OpenPhenom
tables have several wells per compound (e.g. 76,725 wells for 9,946 compounds in OpenPhenom-1920), so image
neighbours come from a single replicate at an arbitrary dose. This adds noise, likely *under*-estimates
agreement for image models, and makes Step 3 inconsistent with Step 4 (which uses the median of all wells).
*Action:* aggregate wells per compound (median, as in Step 4) before building neighbour graphs.

**H2 - Numeric metadata may be used as features** (3.1 all parts; 1.2 Part A)
Features are selected with `select_dtypes(include=[np.number])`. Any numeric metadata column enters the
distance calculation. The comment in 4.1/4.2 Part 1 ("preventing 'concentration', etc. from ruining the math")
indicates the DINO tables contain a `concentration` column, and the saved Step 1.2 code keeps
`Phenotypic_Activity_Score` in the DINO active-only table.
*Action:* print the number of feature columns per dataset (DINO should have 384 raw / 272 normalized) and select
features by name (e.g. `emb_*`, `OpenPhenom_Sphered_Dim_*`, `Mini_Dim_*`).

**H3 - The analyses use the pre-correction normalization** (1.1 → 1.2, 3.1, 4.x)
The project history reports that normalization errors were found on 09/23/2026 and that all morphology datasets
were re-normalized (`*_0922_normed.parquet`). The re-normalization code is not in the repository, and every
analysis still reads the earlier files (`*_sphered_mad_robustized.parquet`,
`BAYER_DINO_embeddings_normed_final.parquet`, and the active-only files derived from them).
*Action:* add the corrected normalization code, point the analyses at its outputs and re-run; or document why the
earlier files are still appropriate.

**H4 - Files used by the analyses are produced outside the repository** (provenance)
For a reproducible publication the following need code in this repository or a citable source: the 10 µM / 24 h
DRUG-seq slice (input to 1.3); CP-CNN and ImageNet-CNN embeddings (4.x); `minimol_embeddings__full_final.parquet`
and `JUMP_CP_Global_Signatures_full.parquet` (4.x); the DINO active-only file actually used downstream (the
saved code in 1.2 writes `...ACTIVE_ONLY-test2.parquet`, but `...-actives_only.parquet` is read); the sign2 copy
(0.1 writes to a different folder than 3.1 reads); and all MOA-annotated files.

**H5 - Saved outputs do not always match the code** (4.1, 4.2)
* 4.1/4.2 Part 1: output is DINO (Normed), code says OpenPhenom-384 (see F2).
* 4.1 Part 4 and 4.2 Part 4: output names the model "OpenPhenom 384 (Normed + Actives)", which is in neither
  notebook's model list; 4.2 Parts 4–5 outputs are copies of the v2 session.
* 4.2 Part 2b: output header says "99%" variance; the code uses 90%.
* 4.2 Part 3: output decodes design A's leaderboard (score +0.1416, 36 axes), whereas the notebook text implies
  design C.

*Action:* re-run each notebook top to bottom before producing figures for publication.

**H6 - 4.2 Parts 4–5 were not adapted to the factor pipeline**
They are verbatim copies of 4.1 Parts 4–5. After 4.2 Part 2, `drugseq_pooled` holds the 43 factors, so Part 4's
gene correlations, symbol mapping and GSEA would run on factor names. Part 5 reads raw genes from the Part 1 cache
(so Part 1 must be run first) and aligns morphology with genes, not factors.
*Action:* port Part 3's approach (separate factor and gene tables) or remove these parts from v3.

### Medium

**M1 - Max-absolute pooling across doses** (4.1 all parts; raw-gene decoding in 4.2)
For each gene, the most extreme value across doses is kept (`model_benchmarking.pooling.max_abs_pool`). This mixes
doses gene by gene and systematically inflates noise (the expected maximum of several noisy values is larger than
any single one). The factor-based v3 benchmark avoids this by using one dose. Consider a single dose, the mean, or
a replicate-weighted consensus.

**M2 - Adjusted SVCCA scores are not comparable across models in designs A and B** (4.2 Parts 2b–2c)
Each model keeps a different number of principal components (5–78 axes), and the adjusted AUC averages over that
model's own axes. Models with few axes are scored only on their strongest ones. Design C (Part 2d) fixes this and
should be the basis for model comparisons.

**M3 - Circularity in the driver-neighbourhood analysis** (4.1 Part 5, 4.2 Part 5)
Driver compounds are selected from a morphology–expression axis, so their clustering in the morphology and
expression spaces is partly guaranteed. Only the MiniMol and Chemical Checker rows are independent tests. The
rule-based verdict uses the morphology/expression enrichment thresholds and inherits this bias.

**M4 - Optimistic p-values for driver similarity** (4.1 Parts 3–4, 4.2 Part 3)
The Mann–Whitney test treats all pairwise similarities among drivers as independent observations, which they are
not (k drivers give k(k−1)/2 pairs). A permutation test over random compound sets of the same size is appropriate.
The random background set can also include drivers.

**M5 - Only one end of each canonical axis is examined** (4.1 Parts 3–5, 4.2 Parts 3–5)
Drivers are compounds with scores above mean + 2 SD. The sign of a canonical axis is arbitrary (the code only
fixes a convention), so compounds at the negative extreme are ignored.

**M6 - Significance of canonical axes** (4.1, 4.2)
An axis is "significant" when its correlation exceeds the null mean + 2 SD at that rank. This is roughly a
one-sided p < 0.025 per axis, uncorrected for testing up to 50 axes, and counts non-contiguous ranks. With 100
permutations, the reported CKA p-value (0.0099) is the smallest value attainable, not an exact p-value.

**M7 - The decoding axes differ from the benchmark axes** (4.1 Part 3)
Decoding keeps the top 5,000 variable genes while the benchmark (Part 2) uses all genes, so the decoded axes are
recomputed and not identical to those that were ranked.

**M8 - CP-CNN in design C shows no signal** (4.2 Part 2d)
The CP-CNN table used in design C (11,664 features, `master_cp_cnn_normed_drugseq_subset.parquet`) differs from
designs A–B (512 features) and gives adjusted CKA < 0 and no significant axes. Check that its feature columns and
compound identifiers are correct before concluding the model carries no signal.

**M9 - Pilot "soft filter"** (4.1/4.2 Part 1)
Dropping the 25% of compounds with the smallest embedding *norm* measures distance from the origin, not from DMSO;
for non-normalized embeddings this is not an activity measure.

**M10 - Compound annotation heuristics** (`model_benchmarking.annotation`)
Matching on the first InChIKey block can return a stereoisomer or salt of a different registered compound; the
IC50 fallback does not restrict units (`standard_units`) and labels every target as "inhibitor"; only the first
mechanism is reported.

**M11 - Reporting of p-values** (3.1 Parts C–G)
`Likelihood_GT_Chance` = (1 − p) × 100 is not the probability that an association is real. p-values printed as
`0.00e+00` are below double precision and should be reported as < 1e-300. The hypergeometric test treats
neighbour edges as independent draws; kNN graphs have fixed out-degree, so p-values are optimistic (fold
enrichment remains a valid descriptive measure).

**M12 - Pairwise cohorts are not comparable across models** (4.3 Part 1)
With `COHORT_STRATEGY = "pairwise"` every model is scored on its own compounds (N = 1,551 for most models but 2,669
for CP-CNN in the saved run), so adjusted AUCs differ partly because the compounds differ. Use `"common"` for ranking.

**M13 - Heuristic thresholds in the purity sweep and morphology verdicts** (4.3 Parts 2-3)
Kurtosis > 6 / ≥ 5, Hoyer ≥ 0.38 / 0.40, pan-stress < 6% / ≥ 14%, and the morphology verdict cut-offs are not
calibrated against a null (e.g. the same statistics for random axes). The two kurtosis/Hoyer floors in the decoding
loop (6.0, 0.40) and in the sweep (5.0, 0.38) also differ. The stress panel is a 13-gene list, and "total signal" is
the sum of log fold-changes, not sequencing depth.

**M14 - Unmixed axes lose their canonical correlation** (4.3 Part 2)
Varimax/FastICA re-cut the subspace of the top axes, so an unmixed axis has no canonical correlation, and the
"significant axes" tag (axis index ≤ number of significant canonical axes) no longer refers to the same axis. FastICA
orientation and ordering (by skewness and variance) are conventions, and with `n_components` equal to the number of
axes the result can change with the seed.

**M15 - Axis-explainer limits** (4.3 Part 4)
The gene module can only use genes among the 5,000 most variable (4 of 57 in the draft run), the drug list covers one
drug class and needs at least two drugs in the screen, and scanning 10 axes x 2 tests is not corrected for multiple
testing.

### Low

* **L1** 4.1 Part 5 and 4.2 Part 5 label the morphology row "Morphology (DINO)" regardless of the model analyzed.
* **L2** mAP in `evaluate_driver_neighborhoods` counts each driver as its own last-ranked hit, slightly lowering
  every value (documented in the docstring).
* **L3** 4.1 Parts 3–5 and 4.2 Parts 4–5 build the winner's cache name without removing `-` and `,`, unlike the
  benchmark cells; they would not find the cache if a model such as "CP CNN drugseq subset (normalized, 512d)"
  ranked first.
* **L4** 0.3 Part A reads the CSV with `ignore_errors=true`; skipped lines are not reported.
* **L5** 1.2 Part B: with fewer than 10 DMSO wells the "activity filter" drops the 5% of compounds closest to DMSO,
  which is a trim rather than an activity test.
* **L6** 3.1 Part A: overlap % is normalized by the first dataset's edges, but the heatmap mirrors it as if
  symmetric.
* **L7** Several cells call `warnings.filterwarnings('ignore')`, hiding potentially useful warnings.
* **L8** `AllChem.GetMorganFingerprintAsBitVect` (0.5) is deprecated in recent RDKit releases (still works in the
  locked version; `rdFingerprintGenerator.GetMorganGenerator` is the replacement).
* **L9** `model_benchmarking.annotation` catches a specific set of exceptions; an `IndexError` (e.g. an empty
  PubChem `Information` list) is not among them and would stop the decoding loop.
* **L10** Procedural code is still duplicated across cells (kNN overlap loop ×5, ZCA normalization ×2, clustering
  ×2, OpenPhenom table builder ×2). Moving these into `src/model_benchmarking/` would make the steps shorter and
  testable; it was not done here to keep the analysis code unchanged.
* **L11** Steps 1–2 still contain Colab-oriented staging assumptions. Step 0 now uses repository-relative
  `data/raw/` and stage-local `results/` directories, although some cells still require `gsutil`.
* **L12** 4.3 reads the Part 2 gene-level DRUG-seq from `embeddings/Novartis_DRUGseq_expression_data/` but the anchor
  `drugseq_raw` from `indexed_by_inchikey/`; the two files are assumed to hold the same signatures.
