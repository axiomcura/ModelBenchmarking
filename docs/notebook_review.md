# Notebook review: open problems and things to know

*Review of all 13 notebooks in `notebooks/`, October 2026. Every cell and every saved output was read. Nothing was
re-run, because the data are in a private Cloud Storage bucket.*

This document complements [`code_review.md`](code_review.md), which records the changes made while reorganizing the
repository and an earlier list of open issues (IDs such as H1 or M4). Issues already described there appear here
with their ID and a one-line summary; everything else is new.

## How to read this document

* **Cell N** is the N-th cell from the top of the notebook, counting markdown and code cells and starting at 1.
  Jupyter and VS Code do not show these numbers, so each reference also gives the section, e.g. *4.2 cell 84
  (C.8 Leaderboard)*.
* **Severity**
  * **Critical** - can invalidate a reported result or the data it rests on.
  * **High** - likely biases a conclusion or makes a comparison unfair.
  * **Medium** - weakens robustness, reproducibility or interpretation.
  * **Low** - misleading text, hygiene, or a latent bug that does not change the saved results.
* **Verify** marks a suspicion that cannot be confirmed without the data.

## What the project is trying to show

The project asks which Cell Painting image model (Bayer DINO, OpenPhenom 384/1,920, CP-CNN, ImageNet CNN) best
reflects what compounds do to cells, using references that need no human labels: chemical structure (MiniMol),
bioactivity (Chemical Checker) and the transcriptional response (Novartis DRUG-seq). Step 3 measures whether two
representations agree on each compound's nearest neighbours. Step 4 measures how much variation image embeddings
share with transcription (CKA and SVCCA) and interprets the shared axes (genes, pathways, driver compounds).
The expected deliverables are a **ranking of image models** and a **biological reading of the shared signal**.
Both depend on the issues below.

## The ten things to know first

1. **The benchmark has no clear winner yet.** In the fairest design (4.2 cell 84, C.8) the top three models are
   within 0.004 of each other (OpenPhenom 1920 Normed 0.1088, OpenPhenom 384 Normed 0.1072, DINO Normed 0.1050),
   and there are no confidence intervals. The winner changes with the design: OpenPhenom 1920 Normed in designs A
   and C, OpenPhenom 384 Normed in design B (4.2 cell 63), and DINO Normed against the CPB anchor (4.3 cell 18,
   where all models lie between 0.025 and 0.035). CKA and SVCCA also disagree. Raw embeddings have *higher* CKA than
   normalized ones in every table (4.1 cell 43: OpenPhenom 1920 Raw 0.122 vs Normed 0.074; 4.2 cell 84: DINO Raw
   0.086 vs Normed 0.079), whereas SVCCA prefers normalized ones. Sphering gives every direction the same variance,
   which helps SVCCA (blind to variance) and hurts CKA (weighted by variance). So "normalization helps" is partly a
   property of the metric.
2. **Several Step 4 inputs are not produced in this repository**, including every file 4.3 reads and the anchor
   its saved results use (see [Data provenance](#data-provenance)).
3. **OpenPhenom "normalization" is global, not per plate** (1.1 cells 19 and 21). One affine transform applied to
   every well cannot remove plate or batch effects, so they remain in the normalized files and in Step 1.2's activity
   calls.
4. **The Part 1 pilot in 4.1 and 4.2 pairs wells, not compounds**: 4,415 well rows share the DRUG-seq profiles of
   523 compounds, and the null shuffles wells. The pilot's CKA (0.141) and "78 significant axes" are inflated
   (4.1/4.2 cells 20 and 26).
5. **Rows of the Step 3 tables are computed on different compounds**, and the "Actives" tables keep a different,
   model-chosen set of compounds for every model, so fold enrichments cannot be compared across rows (3.1).
6. **Chemical Checker is not an independent reference.** Its 3,200 dimensions include five chemistry spaces (A1–A5),
   a transcription space (D1, L1000) and a cell-morphology space (D4, from Cell Painting). "Bioactivity vs
   morphology" (Step 3) and "drivers share a mechanism" (Step 4) are therefore partly chemistry-vs-chemistry or
   morphology-vs-morphology.
7. **Canonical axes are fitted and interpreted on the same compounds**, and a handful of potent compounds (HDAC
   inhibitors) drive most of the decoded axes in 4.2 (cell 107). The axes are neither validated out of sample nor
   independent biological programs.
8. **Run-order bugs in the Part 4 sections.** 4.1 Part 4 silently uses 5,000 genes instead of the 14,370 it
   describes (4.1 cell 65 → cell 82). 4.2 Part 4 stops with a `KeyError` (4.2 cell 110), and if that were fixed it
   would treat the 43 factors as genes.
9. **Dose and time handling is inconsistent.** DRUG-seq has only one time point (24 h; 1.3 cell 10), so every
   "24 h/48 h" filter does nothing. Gene-level analyses merge four doses by max-absolute pooling, while the factors
   use 10 µM only. As a result, 4.2 and 4.3 interpret their axes with differently prepared genes.
10. **Steps 0–2 have no saved outputs** (all except 1.3). Numbers quoted in their text (compound counts, DMSO
    counts, dimensions) cannot be checked from the repository.

## Data provenance

Files read by the analyses, and where they come from.

| File | Read by | Produced by |
|---|---|---|
| `JUMPCP_Sign3_CC_final.parquet` (CC sign3) | 2.1 cell 17; 3.1 cell 5 | 0.1 cells 11–19 |
| `JUMPCP_CC_Sign2_Global_Signatures.parquet` | 3.1 cell 5 (`embeddings/…/parquets/`) | 0.1 cell 22, but written to `Annotations_from_Roman/…` - copied by hand (H4) |
| `JUMP_CP_Global_Signatures_full.parquet` (CC, 115,689 compounds) | 4.1 cells 59, 95; 4.2 cells 91, 127; 4.3 cell 23 | **not in repo** (H4) |
| `minimol_embeddings__Kim_et_al_subset_final.parquet` (9,605 compounds) | 2.1 cell 6; 3.1 cell 5 | 0.2 |
| `minimol_embeddings__full_final.parquet` (110,444 compounds) | 4.1 cells 78, 95; 4.2 cells 110, 127 | **not in repo** (H4) |
| `BAYER_DINO_embeddings_{nonorm,normed}_final.parquet` | 3.1, 4.1, 4.2 | 0.3 (normalized values from the Bayer authors' pipeline) |
| `Bayer_DINO_embeddings_normed_final-actives_only.parquet` | 3.1 cell 5 | **not as written**: 1.2 cell 6 writes `…ACTIVE_ONLY-test2.parquet` (H4) |
| `openphenom_{384,1920}.parquet`, `…_sphered_mad_robustized(-ACTIVE).parquet` | 3.1, 4.1, 4.2 | 0.4, 1.1, 1.2 |
| CP-CNN and ImageNet tables | 4.1 cell 31; 4.2 cells 32, 49, 70; 4.3 cell 10 | outside the repo (H4) |
| `novartis_drugseq_u2os_10uM_24h.parquet` | 1.3 cell 15 | **not in repo** (H4) |
| `novartis_drugseq_u2os_10uM_24h_normalized_factors.parquet` | 4.2 Part 2–3 | 1.3 cell 27 |
| `indexed_by_inchikey/*` - all morphology tables (`*_0922_normed`), the anchors, CP-CNN `9.22` | 4.3 cell 10 | **not in repo**; the re-normalization code is missing (H3; 1.1 cell 1) |
| `cpb_observed_molecules_lpm_latents-extra.parquet` (anchor of the saved 4.3 run) | 4.3 cells 10–12 | **unknown** |
| `jump_cp_latent_perturbation_embeddings.parquet` (`cpb_latent_all` in 4.3) | 4.3 cell 10 (option) | 0.5 - **untrained network** (C1) |

## Findings by notebook

### Step 0 - download and assemble

**[0.1 Chemical Checker](../notebooks/0.download_and_assemble/0.1.chemical_checker_signatures.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 13 (2.2) | Medium (verify) | Only text metadata columns are converted to strings. Numeric metadata from the compound CSV stays numeric and is carried through every join (cell 17), so code that treats "all numeric columns" as features (2.1 cell 17, 3.1 `load_embedding_tables`) would add it to the 3,200 signature dimensions (H2). `astype(str)` also turns missing values into the text `"nan"`. | Check that exactly 3,200 numeric columns remain; prefix metadata with `Metadata_`. |
| 8, 15, 19, 30 | Low | `os.system("gsutil cp …")` return codes are ignored: a failed download fails later with an h5py error, and a failed upload is silent. | `subprocess.run(…, check=True)`. |
| 22, 30 | Low | sign2 is written to a different folder than 3.1 reads (H4; noted in cell 1). | Write to the folder that is read. |

**[0.2 MiniMol](../notebooks/0.download_and_assemble/0.2.minimol_embeddings.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 12 | Low | Both tables contain `Metadata_JCP2022`, so the SMILES join produces `Metadata_JCP2022_x` and `_y` (visible in the saved output of 3.1 cell 63). Conflicting identifiers for the same SMILES are kept side by side without a check. | Drop the duplicate column before joining, or assert that both agree. |
| - | Info | Builds only the Kim *et al.* subset (9,605 compounds); the 110,444-compound table used in Step 4 is built elsewhere (H4). | - |

**[0.3 Bayer DINO](../notebooks/0.download_and_assemble/0.3.bayer_dino_embeddings.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 12, 36 | Low | `ROW_NUMBER() OVER (PARTITION BY …)` has no `ORDER BY`, so when an identifier appears twice in the metadata, which row (and InChIKey) is kept is arbitrary and can change between runs. | Add an `ORDER BY` or assert uniqueness. |
| 8, 32 | Low | A cached CSV is reused whenever it exists, even after an interrupted download; download errors are ignored. `ignore_errors=true` drops malformed lines silently (L4). | Check file size or checksum; compare row counts. |
| 15–27 (Part B) | Low | Kept "for the record", but still uploads `BAYER_DINO_embeddings_norm_final.parquet` (one well per compound) to the bucket when the notebook runs top to bottom. Its name differs from the file that is used by two letters. | Remove the part or disable its upload. |

**[0.4 OpenPhenom](../notebooks/0.download_and_assemble/0.4.openphenom_embeddings.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 18, 33 | Medium (verify) | DMSO wells are recognized by `control_type` / `Metadata_control_type` / `pert_type` ∈ {`negcon`, `control`}. If only `pert_type` exists and uses `control` for every control, positive-control wells are relabelled DMSO and become part of the "untreated" reference in 1.1 and 1.2. | Print which column was used and its values; accept `negcon` only. |
| 18 | Low | DMSO is defined four different ways in Steps 0–1 (control type here, JCP ID in 1.1 cell 17, InChIKey or JCP ID in 1.2 cell 25, `control_type == 'negcon'` in 1.2 cell 10). | One shared helper. |

**[0.5 ChemPerturb-Bridge](../notebooks/0.download_and_assemble/0.5.chemperturb_bridge_embeddings.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 12 | Critical | The embeddings come from an untrained (random) encoder (C1). | Regenerate with trained weights. |
| 1 | Low | Says "the outputs below are those of the original run", but the notebook has no saved outputs. Also says no analysis notebook uses the file, whereas 4.3 cell 10 offers it as the `cpb_latent_all` anchor (flagged there). | Update the text. |

### Step 1 - preprocessing

**[1.1 Normalize OpenPhenom](../notebooks/1.preprocessing/1.1.normalize_openphenom.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 19, 21 (and 32, 34) | **High** | The DMSO median, the MAD and the sphering matrix are estimated once over all DMSO wells of the whole dataset and applied identically to every well. A single global transform rescales the space but cannot remove plate or batch offsets, which is what cell 1 says the step is for. The DINO "normalized" file comes from the Bayer authors' own pipeline (0.3), so Raw-vs-Normed and DINO-vs-OpenPhenom comparisons also mix two different normalization procedures. | Robust z-score per plate against that plate's DMSO wells, then sphere on all DMSO wells; re-run 1.2, 3 and 4 (see also H3). |
| 21, 34 | Medium (verify) | Eigenvalues get a fixed `+1e-4`. If there are fewer DMSO wells than features (1,920), or the features are strongly collinear, directions with almost no DMSO variance are scaled up about 100× (1/√1e-4), so noise can dominate distances. | Print the DMSO count and the eigenvalue spectrum; use a relative regularizer or drop the smallest components. |
| 19 | Info | Features with MAD ≤ 1e-5 are dropped, so "OpenPhenom 384/1920 Normed" have 382 and 1,909 features (4.3 cell 18). | - |
| 8, 10 | Low | The full file is loaded twice just to count columns and DMSO wells. | - |

**[1.2 Filter inactive compounds](../notebooks/1.preprocessing/1.2.filter_inactive_compounds.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 12, 27 | **High** (with 1.1) | Activity is the distance from one global DMSO median. With plate effects left in the data, wells on plates far from that centre are called active more often. | Use a per-plate DMSO reference. |
| 14, 29 | Medium | A well is active if it lies beyond the 95th percentile of DMSO distances, so about 5% of truly inactive wells pass by construction. Filtering wells and then taking the median of the survivors (Steps 3–4) keeps the noisiest replicates of inactive compounds. | Call activity per compound, e.g. from replicate consistency, with a false-discovery-rate threshold. |
| 10, 25 | Low | DMSO is defined differently in Part A and Part B (see 0.4). | One shared helper. |
| 6, 10, 16, 21 | - | Known: numeric columns used as features (H2), output name differs from the file read downstream (H4), Part B is run twice by editing paths. | - |

**[1.3 Normalize DRUG-seq](../notebooks/1.preprocessing/1.3.normalize_drugseq.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 10 (saved output) | Medium | The dataset has **one time point (24 h)** and four doses (0.01–10 µM). Every "24 h and 48 h" filter downstream (4.1 cells 16, 35; 4.2 cells 16, 36, 53, 74; 4.3 via `load_overlap_pair`) does nothing, and texts that mention 48 h (4.1 cell 1, 4.2 Parts 1–2) are wrong. | Remove the 48 h wording; state the dose policy explicitly. |
| 15–27 | Medium | Factors use the 10 µM slice only, whereas every gene-level analysis merges all four doses by max-absolute pooling (M1). 4.2 Part 3 and 4.3 Part 2 therefore find axes on one data preparation (10 µM factors, CPB latents) and interpret them with genes from another. | Interpret with the 10 µM gene slice that produced the factors. |
| 25 | Low | Convergence of `FactorAnalysis(max_iter=150, tol=1e-3)` is not checked, and later notebooks silence warnings. | Print `fa.n_iter_`. |
| 21, 23 | Info | The leading factors at 10 µM are likely dominated by cytotoxicity, stress and proliferation programs, which matches the Hallmarks found when decoding (TNF-α/NF-κB, p53, E2F; 4.1 cell 75, 4.2 cell 107). | Keep in mind when interpreting "shared biology". |
| 27 | Low | The output is written to the working directory rather than `results/`. | - |

### Step 2 - [chemical clustering](../notebooks/2.chemical_clustering/2.1.cluster_chemical_embeddings.ipynb)

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 6, 17 | Medium (verify) | Features are "numeric columns minus a hard-coded list". Numeric metadata (0.1 cell 13) would be clustered as a dimension, and `dropna()` drops compounds with missing metadata. There is no saved output to confirm 512 and 3,200 features. | Select features by name (`Mini_Dim_*`, `*_Dim_*`). |
| 10, 21 | Medium | HDBSCAN runs on the full 512- and 3,200-dimensional vectors, where density estimates are unreliable; 92% of MiniMol and 93% of sign3 compounds end up as noise (3.1 cells 63, 76). `min_samples=30` above `min_cluster_size=15` makes it very conservative. | Reduce dimensions first (e.g. UMAP to 10–20 dimensions); check cluster stability. |
| 17 | Medium | sign3 includes the five chemistry spaces (A1–A5), so sign3 clusters are partly structural clusters and 3.1 Part G is not purely "bioactivity". | Cluster on levels B–E only. |
| 17 | Low | The sign3 cluster file goes to `gs://…/Chemical_Checker_Embeddings/parquets/` at the bucket root (no `embeddings/`), unlike every other CC file. 3.1 cell 76 reads the same path, so it works. | Move it next to the other CC files. |

### Step 3 - [nearest-neighbour agreement](../notebooks/3.knn_neighborhood_analysis/3.1.knn_neighborhood_overlap.ipynb)

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 12, 23, 32, 43, 69, 82 | **High** | Each pair is aligned on the compounds both tables share, so every row of a results table uses a different compound set (cell 36: 5,890 / 8,257 / 5,756 compounds; cell 73: 749 / 440 / 641). Overlap % and the random expectation both depend on N, so fold enrichments cannot be compared across rows. In Parts F–G the "Actives" rows look better partly because they have fewer compounds. | Restrict every model to one common compound set per part. |
| 5, 28, 39, 65, 78 | **High** | "Actives" tables come from 1.2, where each model decides for itself which wells are active. Comparing DINO_Actives with OP1920_Actives compares each model on the compounds it found easiest. | Use one activity definition shared by all models, or none. |
| 12 and all loops | Medium | `fillna(0.0)`: sign2 has empty blocks for compounds without data, and filling them with 0 makes compounds with the same missing blocks look similar under cosine similarity. sign2 neighbourhoods therefore partly encode data availability. | Use sign3, or compare on observed blocks only. |
| 36, 47 | Medium | The CC anchor contains chemistry (A) and Cell Painting morphology (D4) spaces (point 6 above). | Repeat Part D with levels B, C, D2, D3 and D5 only. |
| 10 and loops | - | Known: only the first well per compound is used (H1), numeric metadata may become features (H2), p-values are optimistic and `Likelihood_GT_Chance` is not a probability (M11). | - |
| 16, 25, 36, 47, 73, 86 | Low | Results are kept only in memory (CSV export lines are commented out). | Save the tables to `results/`. |

### Step 4 - morphology vs. transcriptomics

**[4.1 Gene level (v2)](../notebooks/4.morphology_vs_transcriptomics/4.1.morphology_vs_drugseq_genes_v2.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 20, 26 (1.6, 1.9) | **High** | The pilot pairs every **well** with its compound's DRUG-seq profile: 4,415 rows for 523 compounds (cell 41 prints "4415 ➔ 523"). Replicate wells share an identical DRUG-seq row and the null shuffles wells one by one, so replicate consistency is counted as cross-modal agreement. CKA 0.141 (cell 24) and 78 significant axes (cell 26) are inflated and not comparable with Part 2. Also M9 (soft filter). | Aggregate to compounds first, or permute whole compounds. |
| 14 (1.3) | Medium (verify) | Pilot features are the columns starting with `emb_`. The configured file (OpenPhenom 384, cell 10) names its embedding columns differently (`0…383` per 0.4 cell 12; `OpenPhenom_Sphered_Dim_*` after 1.1), so re-running the pilot as configured probably selects no features. | Use `morphology_feature_columns`. |
| 41 (2g) | Medium (verify) | Wells are grouped by compound **and** dose, time and cell line (`detect_group_columns`). CP-CNN and ImageNet give 801 rows against 523 for the other models. If those tables have a dose column, a compound appears once per dose and its DRUG-seq row is repeated (the same pseudo-replication as the pilot). | Check `aligned_df[MERGE_KEY].is_unique` for every model. |
| 41, 43 | Medium | Ranking uses the raw mean canonical correlation with no per-model null, and CP-CNN and ImageNet are scored on more compounds. 4.2 supersedes this leaderboard. | Do not quote 4.1's ranking. |
| 41, 45, 61, 80, 95 | Medium | Caches are named after the model's display name (`cache_filename(name)`), not the file: editing a URI in cell 31 silently reuses the old download (the same kind of bug as F2). Also in 4.2. 4.3 fixed this. | Name caches after the URI. |
| 65 → 82, 86 | **High** (run order) | Cell 65 replaces `drugseq_features` with the top 5,000 genes. Cell 82 only reloads DRUG-seq if `drugseq_pooled` is missing, so Part 4 runs on 5,000 genes, not the 14,370 that cell 76 states. The saved Part 4 output (14,370 genes) comes from another session (H5). | Separate variable names per part, or reload explicitly. |
| 69, 75 | Medium | Canonical correlations of 0.94 to 0.74 are in-sample (50 principal components per side, 523 compounds) and shown without a null; decoding interprets these in-sample axes (M7). | Cross-validated CCA (fit on some compounds, score on held-out ones). |
| 97 (5.2) | Medium | Part 5 does not restrict to treated wells, and after grouping by dose `drop_duplicates(subset=[MERGE_KEY])` keeps an arbitrary dose. | Same preparation as Part 2. |
| 99 (5.3) | Low | Genes are chosen by variance *before* pooling, while Part 3 (cell 65) chooses them *after* pooling, so the gene sets differ. | One gene-selection helper. |
| 103 (5.5) | Medium (verify) | CC features are all numeric columns except the key and `pubchem_cid`; Part 3 (cell 67) also excludes `Metadata_*`. Numeric metadata may enter Part 5's CC similarities. | Use Part 3's selection. |
| 101 (5.4) | Low | The MiniMol key is renamed only if it is called `Metadata_InChIKey_x` or `_y`; there is no fallback as in cell 84, so another name raises a `KeyError`. | Reuse cell 84's logic. |
| 109 (5.8) | Medium | Rule-based verdicts use arbitrary thresholds (fold enrichment 5 and 2.5) on drivers selected from morphology and expression (M3); the row is always labelled "Morphology (DINO)" (L1). | - |
| 52–56 | Info | The JUMP metadata database is downloaded but not used by any later cell. | Remove or use it. |

**[4.2 DRUG-seq factors (v3)](../notebooks/4.morphology_vs_transcriptomics/4.2.morphology_vs_drugseq_factors_v3.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 20, 26 | **High** | Same well-level pilot as 4.1 (identical code and saved output). | As in 4.1. |
| 42, 44 (A) | - | Known: adjusted AUC averaged over a model-dependent number of axes (M2); the saved output reports 99% variance (H5). | - |
| 61, 63 (B) | **High** | The 90% variance threshold keeps 5 to 46 principal components per model (cell 63: OpenPhenom 384 Normed 7, OpenPhenom 1920 Raw 5, CP-CNN 46). Models with 5–7 components are scored only on their strongest axes and top the table. M2 applies to design B as much as to A. | Use design C, or a fixed k. |
| 82 (C.7) | Info | SVCCA keeps min(50, 43 factors) = 43 components for every model, so compressing to 384 dimensions only affects CKA. A fixed number of axes, not the compression, is what makes design C fair. | Say so in the text. |
| 84 (C.8) | **High** | The top three are within 0.004 with no confidence intervals (point 1); CP-CNN uses a different file and shows no signal (M8). | Bootstrap compounds; paired comparisons. |
| 91 (3.1) | Medium | Part 3 decodes whatever design ran last; the saved output decodes design A, and the "statistically significant" tags in cell 107 use design A's 36 axes (H5). | Pin Part 3 to design C. |
| 97 (3.4) | Medium | Genes are pooled across four doses, while the axes are fitted to the 10 µM factors (see 1.3). | Use the 10 µM gene slice. |
| 107 (3.9, saved output) | Medium | A few potent compounds drive many axes. HDAC inhibitors (mocetinostat, vorinostat, pyroxamide) are among the top drivers on axes 2, 3, 4, 6, 8, 9 and 10, and LDN-193189 alone scores 0.81 on axis 4 (next 0.31). The ten axes are therefore not ten independent programs, and a single compound can define one. Only the positive end is examined (M5); p-values are optimistic (M4). | Leave-one-compound-out stability; look at both ends; control for overall response strength. |
| 107 | Low | Enrichr refused a request (axis 6: "status code: 429"), so over-representation results for that axis are missing without a warning beyond a note. | Retry with back-off; cache results. |
| 110 (4.1) | **High** (run order) | Part 4 reads `leaderboard_df["SVCCA Mean (AUC)"]`, a column that exists only in 4.1. After Part 2 of this notebook it raises `KeyError`. If fixed, cells 114 and 118 would treat the 43 factors as genes (H6). | Adapt Part 4 to v3 or remove it. |
| 127–141 (Part 5) | Medium | A copy of 4.1 Part 5 with the same issues; its axes are recomputed against genes, so they are not the v3 benchmark axes (H6). | Adapt or remove. |
| 42, 59, 80 | Medium | Caches named by model name, not URI (see 4.1). | Name caches after the URI. |

**[4.3 Interchangeable anchors (v4)](../notebooks/4.morphology_vs_transcriptomics/4.3.morphology_vs_drugseq_anchors_v4.ipynb)**

| Cell | Severity | Problem | Suggested action |
|---|---|---|---|
| 10, 12 | **Critical (verify)** | The saved run's anchor, `cpb_observed_molecules_lpm_latents-extra.parquet`, has no documented origin. If it was made the way 0.5 makes embeddings (an encoder applied to chemical structure), it describes chemistry rather than measured transcription, and Part 1 ranks models by agreement with chemistry. The `cpb_latent_all` option is the 0.5 file from an untrained network (C1). | Document how the file was produced before interpreting 4.3. |
| 10 | High | Every input comes from `indexed_by_inchikey/` (`*_0922_normed`, CP-CNN 9.22), produced outside the repository (H3, H4). | Add the code or a source. |
| 16, 18 | **High** | With `PCA_MODE = "variance"`, OpenPhenom 384 Raw keeps 36 components, so SVCCA has 36 axes against up to 50 for the others. Its adjusted AUC averages over fewer, stronger axes, and it ranks second (M2 again). | Fix the number of SVCCA axes regardless of the PCA mode. |
| 18 (saved output) | Medium (verify) | DINO (Raw) has **385** features here but 384 in 4.2 (cell 84). The `indexed_by_inchikey` file probably contains one numeric column that is not an embedding; `load_overlap_pair` only excludes `Metadata_*`, `__*` and `pubchem_cid`. | List that file's numeric columns; select features by name. |
| 18 | High | All seven models lie between 0.025 and 0.035, with no confidence intervals; CP-CNN is scored on 2,669 compounds against 1,551 for the others (M12). | `COHORT_STRATEGY = "common"` plus a bootstrap. |
| 33 (saved output) | Medium | The morphology check labels 8 of 10 axes "phenotypic divergence": drivers are *less* similar to each other than random compounds (axis 2: 0.26 vs 0.31). After mean-centering, an average background cosine of about 0.3 means most compounds point in a common direction, which is expected when most compounds barely respond and a minority respond strongly. Strong responders (the drivers) are then compared with a mostly inactive background, so the verdict reflects response strength rather than coherence. | Centre on DMSO (or the median); draw the background from compounds with similar response strength. |
| 33 | Medium | The chemistry verdict "true phenotypic convergence" requires a mean Tanimoto below 0.25, but random ECFP4 pairs average about 0.1 (all ten axes: 0.09–0.12). The verdict reduces to "CC p < 0.01". | Compare drivers' Tanimoto with a random background, as the CC test does. |
| 33, 35 | Medium | Purity: kurtosis 0.2–1.2 and Hoyer 0.21–0.24 on every axis, far below the "pristine" thresholds (κ ≥ 5–6), so the metrics separate nothing here (M13). | Calibrate the thresholds on random axes. |
| 25 | Low | Unmixing is applied to the top 10 axes only; the FastICA result depends on that arbitrary number (M14). | Test 5, 10 and 20. |
| 27 | Medium | Gene decoding again pools four doses (see 1.3). | Use the 10 µM slice. |
| 38–44 | Info | Part 4 has no saved output; the gene module is limited to the 5,000 most variable genes (M15). | - |

## Things to know when re-running

* **Running Steps 0–1 overwrites shared files in the bucket.** 0.1–0.4 upload their outputs, 1.1 and 1.2 write
  straight to `gs://` paths that Steps 3–4 read, and 0.3 Part B uploads an unused table. Re-running them changes the
  inputs of every later analysis for everyone using the bucket.
* **Run Step 4 notebooks from top to bottom, once.** Later parts read variables from earlier ones (`leaderboard_df`,
  `drugseq_pooled`, `drugseq_features`, `cached_filepaths`), and re-running a single part can mix states (see 4.1
  cell 65 and 4.2 cell 110).
* **Local caches go to the working directory** (`cached_*.parquet`, ~1.5 GB for the raw DRUG-seq file, plus
  `jump_metadata.duckdb`). In 4.1 and 4.2 they are named after the model, so delete them after changing a URI.
* **Web services are needed** for decoding: MyGene.info, Enrichr (rate-limited; see 4.2 cell 107), PubChem and
  ChEMBL. Failures are printed as notes or give "Unknown"/"Unannotated" rather than stopping the run.
* **Warnings are silenced** in Step 4 (`warnings.filterwarnings("ignore")`), which also hides convergence warnings
  (L7).
* **Saved outputs do not always match the code** (H5). Additional cases: 4.1 Part 4 output uses 14,370 genes while
  the code now uses 5,000 when run in order; 4.3 Part 3 output shows a "Read Depth |r|" column that the code now calls
  "Total-Signal |r|".
* **Step 2 needs an NVIDIA GPU with RAPIDS cuML**; Step 4 runs on CPU but is much slower.

## Suggested order of work

1. **Pin down provenance**: the CPB anchor of 4.3, the `*_0922_normed` files, `JUMP_CP_Global_Signatures_full`,
   `minimol_embeddings__full_final` and the DINO active-only file. Either add the code or document the source.
2. **Normalize every image model the same way, per plate**, then regenerate the active-only files with an activity
   call made per compound.
3. **Freeze one benchmark protocol** before looking at results: one common compound set, one profile per compound
   (and one dose policy for DRUG-seq), a fixed number of SVCCA axes, cross-validated canonical correlations,
   bootstrap confidence intervals and paired model comparisons, with both CKA and SVCCA reported.
4. **Rebuild Step 3 on that common compound set**, with replicate-median profiles and without model-specific
   activity filters.
5. **Use Chemical Checker without its chemistry (A) and morphology (D4) spaces** wherever it serves as independent
   biology, and drop D1 when the question is about transcription.
6. **Fix or retire 4.1/4.2 Parts 4–5**, which run on the wrong genes, crash or duplicate 4.3.
7. **Make the decoding tests fair**: both ends of each axis, permutation tests over random compound sets,
   backgrounds matched for response strength, and the same dose preparation for the genes as for the anchor.
