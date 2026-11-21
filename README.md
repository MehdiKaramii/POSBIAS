[README_PosBias.md](https://github.com/user-attachments/files/35207936/README_PosBias.md)
# Position-Aware Document Relevance Scoring through Passage-Level Supervision of Large Language Models

This repository accompanies the study **“Position-Aware Document Relevance Scoring through Passage-Level Supervision of Large Language Models”** by Mehdi Karami, Maryam Khodabakhsh, and Seyedehfatemeh Karimi.

The project investigates whether relevance information derived from annotated evidence spans can be used to fine-tune large language models (LLMs) for **document-level relevance scoring**, while reducing the influence of evidence position on predicted relevance scores.

> **Core idea:** The LLM receives the query and the complete document, but its training target is derived from the query and the annotated relevance evidence span. This separates the source of supervision from the model input.

---

## Table of Contents

- [Research Overview](#research-overview)
- [Method at a Glance](#method-at-a-glance)
- [Dataset and Experimental Split](#dataset-and-experimental-split)
- [Training: Localized Relevance Supervision](#training-localized-relevance-supervision)
- [Evaluated Models](#evaluated-models)
- [Evaluation Protocol](#evaluation-protocol)
- [Reported Results](#reported-results)
- [Reproducing the Experiments](#reproducing-the-experiments)
- [Repository and Implementation Details](#repository-and-implementation-details)
- [Citation](#citation)
- [Acknowledgements](#acknowledgements)

## Research Overview

Document-level relevance estimation determines how well a document satisfies the information need expressed by a query. In long documents, relevant information may occur in different locations and may be surrounded by substantial amounts of unrelated content. Consequently, an LLM's predicted relevance score may be affected by where the relevant evidence appears, rather than only by the relevance of the document's content.

This study explores **localized relevance supervision** as a way to guide document-level scoring. BM25 is applied to the query and the annotated relevance evidence span to produce a span-level supervision signal. During fine-tuning, however, the LLM receives the original query and the complete document. The resulting model is evaluated against BM25 scores calculated over complete documents.

The study addresses three research questions:

1. **RQ1 — Score alignment:** Does fine-tuning with localized relevance supervision improve the alignment between predicted document-level relevance scores and reference relevance scores?
2. **RQ2 — Model families and sizes:** How does the effect of localized relevance supervision vary across the evaluated LLM families and model sizes?
3. **RQ3 — Position sensitivity:** Does localized relevance supervision reduce the positional sensitivity of LLM-based document relevance scoring?

## Method at a Glance

The approach has two main stages: **training** and **testing**.

### Training

1. Start with a PosIR query–document pair and its annotated relevance evidence span.
2. Calculate a BM25 score using the query and the annotated relevance evidence span.
3. Normalize the span-level BM25 score to the range `[0, 1]`. This value is the **span-level supervision target**.
4. Supply the query and the **complete document** to the LLM.
5. Fine-tune the LLM so that its document-level prediction approaches the span-level supervision target.

The distinction between the two paths is essential:

| Component | Information used | Role |
|---|---|---|
| Supervision target | Query + annotated relevance evidence span | Produces the training target |
| LLM input | Query + complete document | Produces a document-level relevance prediction |

The model is not trained by replacing the complete document with the evidence span. The evidence span is used to derive the supervision target; the complete document remains the model input.

### Testing

The original and fine-tuned models are evaluated on held-out query–document pairs. The LLM receives the query and complete document, and its predicted document-level relevance score is compared with a **full-document BM25 reference score**. Correlation measures are used to assess score alignment.

Position sensitivity is evaluated separately by creating controlled document variants in which the annotated relevance evidence span is placed at the beginning, middle, and end of the document. The model scores these variants, and CV and PSI quantify the variation across positions.

## Dataset and Experimental Split

The experiments use the **PosIR benchmark**, which provides query–document pairs with annotated spans identifying relevance evidence within documents.

The dataset is divided as follows:

| Partition | Proportion | Purpose |
|---|---:|---|
| Training | 80% | Fine-tuning the LLMs |
| Testing | 20% | Evaluating score alignment and position sensitivity |

Each query–document instance is associated with a query, a document, and an annotated relevance evidence span. The span annotation is used to identify the text from which the training supervision target is derived.

**Dataset access:** Add the official PosIR dataset URL or access instructions here before publishing this repository: `TODO — PosIR dataset link/access instructions`.

## Training: Localized Relevance Supervision

Let:

- `q` denote a query;
- `d` denote its complete associated document;
- `s_r` denote the annotated span containing relevance evidence for `q`;
- `f_θ(q, d)` denote the LLM's predicted document-level relevance score.

The supervision target is derived from the query and the evidence span.

The model is trained using the complete query–document pair:


![Training objective](https://github.com/user-attachments/assets/bcfa04aa-4ee1-4f3e-a3f6-91ac6231bc36)



where `L` is the training loss. The normalized span-level BM25 score is the target; it is not the model input.

## Evaluated Models

Six models from two model families are evaluated, with three model sizes in each family.

| Model family | Model sizes |
|---|---|
| Qwen3 | 0.6B, 4B, 8B |
| Ministral 3 | 3B, 8B, 14B |

Each model is evaluated in its original (unfine-tuned) and fine-tuned versions. The same general evaluation procedure is applied across the six model configurations.

## Evaluation Protocol

The evaluation covers two complementary aspects of document-level relevance scoring.

### 1. Alignment with full-document BM25

For each held-out query–document pair, the original and fine-tuned models produce document-level relevance scores:

<img width="431" height="94" alt="image" src="https://github.com/user-attachments/assets/2800d6e2-e961-408d-875b-fb40288261a7" />


The reference score is obtained by applying BM25 to the query and the **complete document**. The following correlation coefficients are used to compare model predictions with this reference:

- **Pearson's correlation coefficient** — measures linear association.
- **Spearman's rank correlation coefficient** — measures rank correlation.
- **Kendall's rank correlation coefficient** — measures rank association.

The full-document BM25 score used for evaluation has a different role from the span-level BM25 score used as the fine-tuning target.

### 2. Position sensitivity

To test sensitivity to evidence position, each query–document pair is represented by three controlled variants. The annotated relevance evidence span is placed at:

1. the beginning of the document;
2. the middle of the document;
3. the end of the document.

The content is kept comparable while the location of the relevance evidence is changed. The original and fine-tuned versions of **Ministral-14B** are examined in this analysis because Ministral-14B achieved the strongest overall score-alignment results in the reported experiments.

For a query–document pair, let `S_i` be the set of scores produced for its positional variants. Two measures are calculated:

**Coefficient of Variation (CV)**

\[
CV_i = \frac{\sigma(S_i)}{\mu(S_i)} \times 100
\]

CV measures the dispersion of predicted scores relative to their mean.

**Positional Sensitivity Index (PSI)**

\[
PSI_i = 1 - \frac{\min(S_i)}{\max(S_i)}
\]

PSI measures the relative difference between the minimum and maximum predicted scores. Both measures are calculated for each query–document pair and then averaged over the test set. Lower CV and PSI values indicate lower variation across the evaluated positional configurations.

## Reported Results

### Document-level score alignment

The following values are transcribed from the article's Table 1.

| Model family | Size | Version | Pearson | Spearman | Kendall |
|---|---:|---|---:|---:|---:|
| Ministral | 3B | Original | 0.0254 | 0.0406 | 0.0273 |
| Ministral | 3B | Fine-tuned | **0.8070** | **0.7818** | **0.5941** |
| Ministral | 8B | Original | -0.0113 | -0.0235 | -0.0158 |
| Ministral | 8B | Fine-tuned | **0.7895** | **0.7675** | **0.5809** |
| Ministral | 14B | Original | 0.0968 | 0.0971 | 0.0656 |
| Ministral | 14B | Fine-tuned | **0.8089** | **0.7849** | **0.5982** |
| Qwen3 | 0.6B | Original | -0.0123 | -0.0051 | -0.0040 |
| Qwen3 | 0.6B | Fine-tuned | **0.7312** | **0.7108** | **0.5275** |
| Qwen3 | 4B | Original | 0.240 | 0.0020 | 0.0014 |
| Qwen3 | 4B | Fine-tuned | **0.6954** | **0.6760** | **0.4958** |
| Qwen3 | 8B | Original | 0.0192 | 0.0166 | 0.0114 |
| Qwen3 | 8B | Fine-tuned | **0.7027** | **0.6823** | **0.5004** |

The article reports that fine-tuning increases all three correlation measures across all six evaluated model configurations. Ministral-14B obtains the highest reported fine-tuned correlations overall. The results also indicate that model size alone does not consistently determine the degree of alignment.

> **Value to verify before publication:** The article's Table 1 lists the original Qwen3-4B Pearson coefficient as `0.240`. Confirm this value against the final analysis output before treating it as definitive; it is reproduced here exactly as it appears in the supplied article.

### Position sensitivity

The article reports the following results for Ministral-14B:

| Model | Version | Average CV (%) | Average PSI |
|---|---|---:|---:|
| Ministral-14B | Original | 9.53 | 0.19 |
| Ministral-14B | Fine-tuned | **2.41** | **0.05** |

The reported reductions indicate that the fine-tuned model produces more stable relevance scores across the beginning, middle, and end positional configurations.

## Reproducing the Experiments

The article specifies the experimental methodology, dataset split, model families and sizes, and evaluation metrics. It does **not** specify the exact repository entry-point scripts, dependency versions, hardware configuration, random seed, or command-line arguments. Those implementation-specific details should be documented from the actual code rather than inferred from the paper.

A reproduction of the reported protocol should follow these steps:

1. Obtain the PosIR benchmark and its query–document pairs and annotated relevance evidence spans.
2. Apply the 80/20 training/testing split used for the experiment.
3. For each training instance, calculate BM25 between the query and its annotated relevance evidence span.
4. Normalize the span-level BM25 scores to `[0, 1]` and use them as fine-tuning targets.
5. Fine-tune each selected model using the query and complete document as input.
6. On the held-out test set, obtain document-level predictions from original and fine-tuned models.
7. Calculate full-document BM25 reference scores and compute Pearson, Spearman, and Kendall correlations against the model predictions.
8. For the position-sensitivity experiment, create beginning, middle, and end variants of the annotated relevance evidence span; score the variants with original and fine-tuned Ministral-14B; then calculate average CV and PSI.

### Implementation-specific information to add

Before using this README as a complete software installation and execution guide, fill in the following from the repository:

- [ ] Repository clone URL.
- [ ] Supported Python version and environment setup.
- [ ] Dependency installation command and pinned package versions.
- [ ] Dataset download/access instructions and expected directory structure.
- [ ] Data-preparation script names and invocation examples.
- [ ] Fine-tuning entry point, configuration files, and example command.
- [ ] Inference and evaluation script names and example commands.
- [ ] Hardware requirements and tested GPU configuration.
- [ ] Random seed and exact split-generation procedure, if applicable.
- [ ] Model checkpoint download locations and access requirements.

Do not place private credentials, access tokens, or restricted datasets in the repository.

## Repository and Implementation Details

This README describes the research method and reported experimental protocol. The exact file and directory names should match the implementation in this repository. Add a short repository tree here once the final code structure is fixed, for example:

```text
<repository-root>/
├── README.md
├── <data-preparation-files>/
├── <training-files-and-configurations>/
├── <inference-files>/
└── <evaluation-files>/
```

Replace the placeholders with the actual directories and scripts; they are not intended to represent confirmed filenames.

## Citation

If you use this work, please cite the accompanying study. Update the publication details when the paper has a final venue, volume, page range, or DOI.

```bibtex
@unpublished{karami_position_aware_relevance_scoring,
  title  = {Position-Aware Document Relevance Scoring through Passage-Level Supervision of Large Language Models},
  author = {Karami, Mehdi and Khodabakhsh, Maryam and Karimi, Seyedehfatemeh},
  note   = {Manuscript; publication details to be added}
}
```

## Acknowledgements

The experiments use the PosIR benchmark for query–document pairs and annotated relevance evidence spans. Please consult the benchmark's official documentation for its terms of use and citation requirements.
