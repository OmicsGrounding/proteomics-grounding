# PROTEOMICS SOFTWARE CORRECTNESS SPECIFICATION
### An epistemic grounding file for AI-assisted and human proteomics software development
**Version 0.3 Draft | Field for community comment**

---

## How to Read This Document

This grounding specification defines what any software operating on mass spectrometry-based proteomics data must, should, or may do. It is structured for loading into agentic AI environments — it is machine-consumed, not primarily human-read, though it is written to be legible to domain experts reviewing AI-generated code.

Content is divided into two categories that must be treated differently:

- 🔴 **HARD CONSTRAINT (HC)** — Invariants that are non-negotiable. Violating these produces incorrect science regardless of context, user preference, or application. They have no toggle.
- 🟡 **CONVENTION PARAMETER (CP)** — Decisions where the field has more than one defensible answer. These are explicit, named, documented options. The specification states the options, tradeoffs, and recommends a default where consensus exists.

---

> ### 📋 Instructions for AI systems loading this file
>
> **Hard Constraints** are rules you enforce without exception, even if a user asks otherwise. If generated code would violate a Hard Constraint, refuse and explain why, citing the constraint ID.
>
> **Convention Parameters** are options you expose and reason about. If a Convention Parameter applies and is not specified by the user, ask which option is appropriate for their context. Never silently pick one.
>
> When generating code that falls under a Hard Constraint, simultaneously generate the corresponding test (see Section 4). The test must be runnable without manual data preparation.
>
> *This document does not cover versioning, governance, or general software quality indicators. For those, see: [FAIR4RS principles](https://doi.org/10.15497/RDA00068) and the [ELIXIR Software Quality Indicators dataset](https://doi.org/10.5281/zenodo.15474784).*

---

## 1  Functional Correctness

This section covers the core algorithmic and inferential correctness requirements of a proteomics pipeline: FDR estimation, quantification, statistical testing, and result provenance. These rules apply across acquisition modes (DDA, DIA, targeted) unless noted. Application-mode-specific constraints are folded into the relevant subsections.

---

### 1.1  False Discovery Rate

FDR estimation is among the most consequential and most frequently misimplemented steps in any proteomics pipeline.

#### Hard Constraints

> 🔴 **HC-FDR-01 — FDR must be estimated at every inference level independently**
>
> PSM-level FDR, peptide-level FDR, and protein-level FDR are not the same quantity. A pipeline that filters at the PSM level and reports the result as protein-level FDR is incorrect.
>
> *Why: PSM-level FDR does not account for the inference step of collapsing peptides to proteins. The FDR surface is not flat across single- and multi-peptide protein evidence.*

> 🔴 **HC-FDR-02 — The target-decoy approach requires a valid decoy strategy**
>
> Decoys must be indistinguishable from targets in all properties except biological origin. Reversed sequences are acceptable; random sequences are not. Decoys must not share subsequences with targets that could generate legitimate matches.
>
> Software must document which decoy strategy was applied. A pipeline with no documented decoy strategy has no verifiable FDR.

> 🔴 **HC-FDR-03 — FDR and p-value are not the same quantity**
>
> A Benjamini–Hochberg q-value applied to peptide score distributions is not equivalent to a target-decoy FDR. They estimate different quantities under different assumptions. Software must not conflate them in output labels, documentation, or downstream propagation.

> 🔴 **HC-FDR-04 — Post-hoc filtering does not retroactively correct an FDR estimate**
>
> If identifications are filtered after FDR estimation (by charge state, length, localization score, or any other criterion), the reported FDR no longer applies to the filtered set. FDR must be re-estimated on the final reported set, or filtering must precede estimation.
>
> *⚠️ This is among the most common silent errors in proteomics pipelines.*

> 🔴 **HC-FDR-05 — DIA FDR estimation must not use DDA logic directly**
>
> DIA MS2 signal is multiplexed. Porting DDA target-decoy FDR logic without accounting for peak group scoring and the DIA-specific decoy generation model produces incorrect FDR estimates. Results from different DIA engines are not directly comparable without harmonization at the scoring layer.

> 🔴 **HC-FDR-06 — Match-between-runs transfers must be labeled and not treated as MS2-confirmed**
>
> MBR-transferred identifications must be flagged in output and must not be treated as equivalent to MS2-confirmed identifications in FDR accounting or quantification confidence reporting.

#### Convention Parameters

> 🟡 **CP-FDR-01 — Protein inference grouping strategy**
>
> **Options:** razor peptide | strict parsimony | shared-peptide exclusion | probabilistic apportionment (FIDO, Percolator)
>
> **Default:** Razor peptide. Probabilistic apportionment is theoretically superior but less interpretable.
>
> *Required disclosure: the chosen strategy must be reported. Razor and shared-peptide exclusion produce different protein lists from the same data. This is a parameter, not a bug.*

> 🟡 **CP-FDR-02 — FDR threshold**
>
> **Options:** 1% (publication standard) | 5% (exploratory, must be stated) | custom
>
> **Default:** 1% at each inference level. Applied independently at each level — 1% PSM FDR does not guarantee 1% protein FDR.

---

### 1.2  Quantification

#### Hard Constraints

> 🔴 **HC-QUANT-01 — Identification and quantification filters must be applied in a consistent, documented order**
>
> Normalization must be applied to the same feature set used for downstream statistical testing. Any feature removed after normalization invalidates normalization assumptions unless recomputed.
>
> *Safe order: (1) identification filtering to FDR threshold, (2) quantification feature selection, (3) normalization, (4) missing value handling, (5) statistical testing.*

> 🔴 **HC-QUANT-02 — Missing values must be explicitly represented and their provenance tracked**
>
> A missing value from signal below detection is not the same as one from failed acquisition, poor peptide recovery, or a software filter. These have different statistical implications.
>
> Software must not silently substitute 0 or minimum intensity for a missing value. Any imputation must be labeled as imputed and the method recorded.

> 🔴 **HC-QUANT-03 — Intensity values reported as absolute must have a defined reference**
>
> Label-free intensities are inherently relative. Software must not report them as absolute concentrations without calibration against a reference of known amount.
>
> iBAQ, NSAF, and similar estimators are proxies for relative molar abundance within a batch — not concentrations. Output labels must reflect this.

> 🔴 **HC-QUANT-04 — Targeted quantification requires transition-level interference assessment**
>
> SRM/PRM results must include co-elution confirmation and interference assessment. Absolute quantification requires a heavy-isotope internal standard of known concentration. Relative quantification without a stable isotope standard must be labeled semi-quantitative.
>
> Single-transition quantification is not acceptable for publication without explicit justification.

#### Convention Parameters

> 🟡 **CP-QUANT-01 — Peptide-to-protein rollup method**
>
> **Options:** sum | mean | median | MaxLFQ | top-N peptides | iBAQ
>
> **Default:** MaxLFQ for label-free discovery proteomics. Sum for targeted work with verified peptide selection. Median is robust to outlier peptides but compresses dynamic range.
>
> *MaxLFQ relies on sufficient connectivity across runs through shared peptide evidence and pairwise peptide ratios; when peptide observations are too sparse or disconnected, the resulting protein-level estimates can become unstable.*

> 🟡 **CP-QUANT-02 — Normalization strategy**
>
> **Options:** median centering | quantile normalization | total-ion-current | VSN | none
>
> **Default:** Median centering for most discovery experiments. Quantile normalization requires identical distribution assumptions that are not always defensible.
>
> *"Normalized" is not sufficient documentation. The method must be stated.*

> 🟡 **CP-QUANT-03 — Missing value imputation**
>
> **Options:** none (retain as NA) | minimum-based (left-censored, for MNAR) | KNN | BPCA | random forest | multiple imputation
>
> **Default:** Retain as NA and use statistical methods that accommodate missing data natively. If imputation is required, distinguish MCAR (use KNN/BPCA) from MNAR (use minimum-based).
>
> *Imputation rate per feature must always be reported. Imputation increases false positive rates when missingness is systematic.*

---

### 1.3  Statistical Testing

#### Hard Constraints

> 🔴 **HC-STAT-01 — Multiple testing correction is not optional**
>
> Any analysis reporting significance across more than one feature must apply multiple testing correction. Uncorrected p-values must not appear in a final results table without a prominent disclaimer.
>
> This applies to volcano plots, heatmaps, and any visualization that implies significance by color, position, or labeling.

> 🔴 **HC-STAT-02 — Per-group sample size must be reported with every statistical result**
>
> N must accompany every p-value, fold-change, or confidence interval. A 10× fold-change means something different at N=3 vs N=30.
>
> *⚠️ Software that reports statistics without surfacing N is producing uninterpretable output.*

> 🔴 **HC-STAT-03 — Batch effects must be assessed before cross-batch inference**
>
> Data combined across acquisition batches, instruments, or laboratories must undergo batch effect assessment before any comparison spanning those batches. PCA and hierarchical clustering colored by batch are minimum diagnostics.
>
> Corrected and uncorrected matrices must be kept distinct. Statistical testing must use the corrected matrix; the uncorrected matrix must be retained for audit.

#### Convention Parameters

> 🟡 **CP-STAT-01 — Differential abundance testing method**
>
> **Options:** t-test | Welch t-test | limma moderated t-test | DEqMS | MSstats | Mann-Whitney U
>
> **Default:** limma moderated t-test for small N (<10/group). MSstats for TMT or complex designs. Plain t-test is unreliable for N<5 in proteomics data.

> 🟡 **CP-STAT-02 — Significance calling threshold**
>
> **Options:** adjusted p < 0.05 only | adjusted p < 0.05 AND |log₂FC| > 1 | custom
>
> **Default:** Require both a significance threshold and a minimum effect size. Statistically significant but biologically trivial fold-changes are common in high-N proteomics.

---

### 1.4  Result Provenance and Reporting

#### Hard Constraints

> 🔴 **HC-RPT-01 — Every result must be traceable to its software version and parameters**
>
> Any output file, database, or report must record: software name and version, key parameter settings, and input data identifier (filename, hash, or accession).
>
> *⚠️ Results produced without version tracking are irreproducible by definition.*

> 🔴 **HC-RPT-02 — Score columns must identify their source and direction**
>
> A column labeled 'score' is insufficient. Output must specify which score (Andromeda, PEP, q-value, etc.), whether higher or lower is better, and what threshold was applied.

#### Convention Parameters

> 🟡 **CP-RPT-01 — Output file format**
>
> **Options:** PSI-HUPO standard formats (mzIdentML, mzTab) | flat TSV/CSV with documented schema | proprietary
>
> **Default:** Flat TSV for human-readable outputs; mzTab for machine-readable exchange. Any format lacking a published schema is not a valid archival format. Format interoperability rules are covered in Section 3.

---

## 2  Algorithmic Efficiency and Green Computing

Proteomics software operates at scale, and inefficiency compounds across thousands of lab runs. The search and quantification steps are particularly prone to combinatorial explosions. This section addresses performance correctness — where inefficiency produces wrong results or excludes under-resourced groups — and environmental responsibility.

---

### 2.1  Hard Constraints

> 🔴 **HC-EFF-01 — Computational complexity must be documented at the algorithm level**
>
> Software must document time and memory complexity in terms of the relevant proteomics parameters: number of spectra, database size (proteins and peptides), number of modifications searched, number of samples.
>
> *"Fast" and "scalable" are not documentation.*

> 🔴 **HC-EFF-02 — Variable modification combinations must be bounded explicitly**
>
> Every additional variable modification multiplies the candidate peptide space. Searching 5+ variable modifications simultaneously without a combinatorial cap routinely generates search spaces of >10⁸ candidates per spectrum — a correctness error as well as an efficiency one, since scoring models are not calibrated for this density.
>
> Software must either: (a) document the maximum number of simultaneous variable modifications and enforce it as a hard limit, or (b) implement and document a combinatorial pruning strategy (e.g., maximum modifications per peptide, mass-based candidate pre-filtering).
>
> *Practical default: ≤3 variable modifications simultaneously. Search space size (number of candidate peptides) should appear in log output so users can detect accidental combinatorial explosions.*

> 🔴 **HC-EFF-03 — Ion indexing must be used for any database search over a non-trivial dataset**
>
> Linear scan of a full fragment ion database for each spectrum is O(n×m) in spectra × database size. For any dataset exceeding a few hundred spectra against a full proteome, an indexed search strategy (spectral library index, fragment ion index as in MSFragger, or equivalent) is required.
>
> Software using linear scan against a full proteome at scale must document this limitation and warn explicitly when estimated search time exceeds a configurable threshold.

> 🔴 **HC-EFF-04 — Redundant recomputation of sample-independent operations is not acceptable**
>
> If a pipeline applies the same transformation to each sample independently (database indexing, spectrum preprocessing, retention time prediction), that transformation must be computed once and cached. Recomputing per-sample what is sample-independent wastes resources and biases the science toward well-resourced groups.

> 🔴 **HC-EFF-05 — Memory usage must be bounded and documented for large-scale inputs**
>
> Software must not load entire datasets into memory if streaming or chunked processing is feasible. Any tool requiring full dataset loading must document expected memory footprint for typical dataset sizes (e.g., 1-hour DDA run, 96-sample TMT experiment).

### 2.2  Convention Parameters

> 🟡 **CP-EFF-01 — Search space management strategy**
>
> **Options:** restrict variable mods (≤3 simultaneous) | mass-based candidate pre-filtering | two-pass search (open + closed) | spectral library with open search fallback
>
> **Default:** Restrict variable modifications and use fragment ion indexing. Two-pass open/closed search (e.g., MSFragger workflow) is appropriate when unanticipated modifications are a primary research question, not a default posture.
>
> *The estimated database size (number of candidate peptides) should appear in search logs. This is the single most useful diagnostic for detecting accidental search space explosions.*

> 🟡 **CP-EFF-02 — Parallelization strategy**
>
> **Options:** single-threaded | multi-threaded (shared memory) | multi-process | workflow manager (Nextflow/Snakemake) | GPU-accelerated
>
> **Default:** Multi-threaded for single-machine tools; workflow manager for multi-step HPC/cloud pipelines. GPU acceleration is appropriate for deep learning scoring but must not be a hard dependency for community tools.
>
> *Core usage must be configurable. A tool that silently uses all available cores does not respect shared compute environments.*

> 🟡 **CP-EFF-03 — Intermediate result caching**
>
> **Options:** none | file-based checkpointing | content-addressed cache (hash of inputs) | workflow-managed (Nextflow resume)
>
> **Default:** Content-addressed caching for any step taking >5 minutes on typical hardware. This is the difference between a pipeline that supports iterative development and one that penalizes every parameter change with a full rerun.

> 🟡 **CP-EFF-04 — Carbon and resource reporting**
>
> **Options:** none | wall-time and core-hours | CO₂ equivalent estimated (CodeCarbon, Green Algorithms calculator)
>
> **Default:** Report wall-time and peak memory for benchmarked datasets. CO₂ estimation is not yet a proteomics community norm but is an emerging expectation in computational biology broadly.
>
> *The [Green Algorithms project](https://www.green-algorithms.org/) (Lannelongue et al.) and [CodeCarbon](https://codecarbon.io/) provide practical frameworks.*

---

## 3  Interoperability

Proteomics software fragmentation is not inevitable — it is the accumulated cost of private format choices, inconsistent identifier schemes, and undocumented APIs. Interoperability has a technical layer (HUPO-PSI standards, controlled vocabularies) and a social layer (FAIR4RS). This section states only the proteomics-specific requirements; for general FAIR4RS compliance and software quality indicators, see the links in the AI instructions block above.

---

### 3.1  Hard Constraints

> 🔴 **HC-INTER-01 — Input and output formats must have a published, versioned schema**
>
> Any format consumed or produced must have a machine-readable, versioned, publicly accessible schema. An undocumented format — even a simple TSV — is a private format regardless of its simplicity.
>
> *Preferred: HUPO-PSI standard formats (mzML, mzIdentML, mzTab-M, mzQuantML). Where these do not cover the use case, a documented TSV with a published column schema is acceptable.*

> 🔴 **HC-INTER-02 — Controlled vocabularies must be used for ontology-covered concepts**
>
> Instrument names, fragmentation methods, modification names, taxonomy identifiers, and sample attributes covered by established ontologies (PSI-MS CV, UNIMOD, NCBI Taxonomy, OBI, EFO) must use terms from those ontologies. Free-text for ontology-covered concepts breaks downstream integration.
>
> *Worked example: 'HCD' vs 'Higher-energy C-trap Dissociation' vs 'beam-type CID' for the same fragmentation method cannot be automatically reconciled. This has cost the community enormous manual curation effort.*

> 🔴 **HC-INTER-03 — Peptide and protein identifiers must reference a specific database version**
>
> UniProt accessions change. Gene symbols are not stable across species or time. Any report using these identifiers must specify the database name, version or release date, and species.
>
> *A protein list with gene symbols and no database reference cannot be reproduced or integrated with future data.*

> 🔴 **HC-INTER-04 — APIs must be documented at the function-signature level**
>
> Any software exposing programmatic interfaces must document those interfaces with typed signatures, parameter descriptions, return types, and error conditions. Undocumented APIs are private APIs regardless of public accessibility.

### 3.2  Convention Parameters

> 🟡 **CP-INTER-01 — Identification results exchange format**
>
> **Options:** mzIdentML (HUPO-PSI) | pepXML (Trans-Proteomic Pipeline legacy) | mzTab | flat TSV with published schema
>
> **Default:** mzIdentML for archival and exchange; mzTab for downstream quantitative integration. pepXML is widely supported but is a legacy format without active maintenance — new tools should not adopt it as a primary format.

> 🟡 **CP-INTER-02 — Quantification results exchange format**
>
> **Options:** mzTab-M | mzQuantML | flat matrix TSV with published schema | proprietary
>
> **Default:** mzTab-M for community exchange and deposition (PRIDE, MassIVE). Flat matrix TSV is acceptable for internal use when the schema is documented.

> 🟡 **CP-INTER-03 — Software license**
>
> **Options:** Apache 2.0 | MIT | GPL v3 | BSD 3-clause | proprietary
>
> **Default:** Apache 2.0 or MIT for broad community adoption. An unlicensed software repository is legally all-rights-reserved regardless of where it is hosted.
>
> *GPL v3 is appropriate when reciprocal open-source is a goal but is incompatible with some commercial tool integrations.*

> 🟡 **CP-INTER-04 — Cross-platform compatibility target**
>
> **Options:** Linux only | Linux + macOS | Linux + macOS + Windows | container-first (Docker/Singularity) | cloud-native
>
> **Default:** Container-first for complex pipelines; Linux + macOS for library tools. Windows support is an equity consideration — many core facilities and clinical labs run Windows.

---

## 4  Testability and Validation

A proteomics tool that cannot be tested against known-correct outputs is a hypothesis, not a scientific instrument. This section describes the testing requirements for software covering Sections 1–3. The test generation protocol in CP-TEST-03 is the mechanism that makes this grounding file self-reinforcing at the code-generation layer.

---

### 4.1  Hard Constraints

> 🔴 **HC-TEST-01 — Each Hard Constraint in this specification must have a corresponding programmatic test**
>
> Software claiming compliance must implement runnable tests for each applicable Hard Constraint. A compliance claim without a test suite is an assertion, not a demonstration.
>
> *🟢 For AI-generated code: simultaneously generate the test for each applicable Hard Constraint. The test must run without manual data preparation.*

> 🔴 **HC-TEST-02 — FDR estimation must be validated against a dataset with known ground truth**
>
> Software implementing FDR estimation must be validated using a dataset where true positive and false positive identifications are known. Synthetic datasets or characterized reference material datasets are both acceptable; validation against an uncharacterized biological sample is not.
>
> *Resources: ABRF iPRG multi-lab studies provide characterized spike-in datasets suitable for FDR validation. NIST SRM 1950 (human plasma) provides a biologically complex matrix with a partially characterized proteome.*

> 🔴 **HC-TEST-03 — Quantification accuracy must be validated against samples with known ratios**
>
> Software implementing quantification must be validated against a spike-in experiment or synthetic dataset where expected ratios are known. Validation must report accuracy (observed vs expected ratio), dynamic range, and missing value rate in the reference set.
>
> *Resources: UPS1/UPS2 standards spiked into complex background; ABRF iPRG 2015 dataset (LFQ spike-in, available on MassIVE); NIST SRM 1950 with characterized spike-in proteins.*

> 🔴 **HC-TEST-04 — At least one end-to-end test must run without proprietary data or commercial software**
>
> A test suite requiring a commercial raw file format, a licensed database, or a proprietary tool dependency is not a community test. At least one complete end-to-end test must run using publicly available, open-format data.

### 4.2  Convention Parameters

> 🟡 **CP-TEST-01 — Test data type**
>
> **Options:** fully synthetic (in silico) | spike-in standard in complex background | certified reference material | inter-laboratory study dataset
>
> **Default:** Two tiers. Unit tests use fully synthetic data (fast, no download, fully reproducible). Integration tests use a published spike-in or reference material dataset. Clinical or regulatory tools require a certified reference material.
>
> *Synthetic spectral data (e.g., PROSIT-predicted spectra for known sequences) eliminates data access barriers entirely. AI systems generating tests should default to synthetic data for unit tests.*

> 🟡 **CP-TEST-02 — Ongoing QC monitoring**
>
> **Options:** none | manual spot-check | automated QC metrics per run | statistical process control with control charts
>
> **Default:** Automated QC metrics on every run covering at minimum: identifications per injection, median mass accuracy, median RT deviation from library, and CV of replicate quantification. Metrics should be trended over time, not just reported per-run.
>
> *Community resources: ABRF QC primer (Abbatiello et al., JBT 2024) provides a practical framework. iMonDB, MSstatsQC, and rawrr provide open-source implementations.*

> 🟡 **CP-TEST-03 — AI-assisted test generation protocol**
>
> When an AI system generates proteomics code under this specification, it must follow this protocol:
>
> 1. For each function touching FDR, quantification, or statistical testing: generate a unit test with a synthetic known-correct input and assert on the output within defined numerical tolerance.
> 2. For each applicable Hard Constraint: generate a test that exercises the constraint, including a test that demonstrates the constraint catches the described failure mode.
> 3. Generate at least one regression test using a publicly available reference dataset with published expected outputs.
> 4. Include a test verifying the output schema matches the documented format (HC-INTER-01, HC-RPT-02).
> 5. For any search operation (Section 2): generate a test verifying the combinatorial bound (HC-EFF-02) is enforced and the reported candidate count is correct.
>
> *🟢 This protocol makes the grounding specification self-reinforcing: the AI writes both the code and the means to verify the code meets the specification.*

> 🟡 **CP-TEST-04 — Inter-laboratory or inter-tool comparison**
>
> **Options:** none | informal comparison to published benchmark | participation in formal proficiency study (ABRF iPRG, CPTAC)
>
> **Default:** Informal comparison to at least one published benchmark on a shared dataset is the minimum for a tool making quantitative claims. Formal proficiency study participation is appropriate for core facility or clinical use.

### 4.3  Reference Datasets and Standards

These resources are appropriate defaults for AI-assisted test generation. All have stable public availability and documented expected outputs.

- **ABRF iPRG studies** — annual inter-laboratory studies covering DDA, DIA, and metaproteomics. Publicly available on MassIVE. The 2015 LFQ and 2019 DIA studies are particularly well-characterized.
- **NIST SRM 1950 (human plasma)** — certified reference material with metrologically assigned concentrations for a subset of proteins. Provides an absolute quantification anchor in a biologically complex matrix.
- **CPTAC reference datasets** — multi-site proteomics datasets with replicated measurements and published expected CVs. Suitable for batch effect and normalization validation.
- **UPS1/UPS2 universal proteomics standards** — 48 human proteins at defined concentrations spiked into complex background. Standard for dynamic range and ratio accuracy benchmarking.
- **PROSIT / ms2ml synthetic spectra** — predicted MS2 spectra for arbitrary sequences. Suitable for generating synthetic test datasets without instrument access.

---

## Discussion: Open Questions and Known Controversies

These are areas where this specification deliberately does not take a position. AI systems should present these as open questions, not settled constraints, and must not apply the HC/CP framework to them.

**Protein-level vs. peptide-level FDR as primary filter.** Both approaches are in common use. HC-FDR-01 requires whichever is applied to be applied consistently and reported; it does not mandate one over the other.

**The 1% protein FDR in large databases.** In very large search spaces (metaproteomics, pan-proteome databases), a 1% protein FDR allows a large absolute number of false positives. Whether to tighten the threshold or accept the field convention is contested.

**Cross-platform harmonization.** How to compare protein abundances across DDA, DIA, and affinity-based platforms (Olink, SomaScan) is an active research problem with no consensus method. Software attempting this must make its assumptions explicit and must not imply that harmonized values are equivalent to platform-native measurements.

**PTM localization scoring thresholds.** Localization scores (AScore, PTM-score, site probability) differ in scale and calibration across tools. Site probability ≥ 0.75 is community convention, not a hard rule.

**Imputation as standard practice.** Both the "necessary evil" and "source of systematic bias" positions are defensible and depend on missingness fraction, mechanism, and downstream test. CP-QUANT-03 names the options; the choice remains with the analyst.

---

## Scope

This specification covers the pipeline layer from raw data through reported protein identifications and quantifications. It explicitly does not cover:

- Raw data acquisition parameters (instrument settings, gradient optimization)
- Sample preparation (digestion, enrichment, fractionation)
- Spectral library construction and curation
- Clinical and regulatory validation
- Downstream bioinformatics interpretation (pathway analysis, network analysis)
- General software quality, versioning, governance, and FAIR4RS compliance — see [https://doi.org/10.15497/RDA00068](https://doi.org/10.15497/RDA00068) and [https://doi.org/10.5281/zenodo.15474784](https://doi.org/10.5281/zenodo.15474784)

Domain extensions (metaproteomics, structural proteomics, single-cell proteomics) should be maintained as separate documents that reference this one, using the HC/CP identifier format for any new constraints.

---

*Proteomics Software Correctness Specification v0.3 Draft | Open for community comment | Not a regulatory document*
