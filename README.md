# OpenADMET CYP Inhibition Challenge

Stacked ensemble for the [OpenADMET CYP Inhibition Blind Challenge](https://huggingface.co/spaces/openadmet/cyp-challenge)
— **pIC50 regression** across four cytochrome P450 isoforms, and **time-dependent inhibition
(TDI)** classification for two of them.

On the organisers' **interim leaderboard** — scored on the *entire* held-out test set rather than the half the live board uses — this work placed **rank 28 of 219** on the regression track (Tier 6); and **rank 6 of 114** on the TDI track, inside **Tier 1** — the top tier, the 26 entries the organisers could not separate from first place.

> Write-up only. This repository holds the method description and results; it is not the
> training code.

## Approach

A representation-diverse ensemble, stacked per endpoint.

| stage | what it does |
|---|---|
| **Base learners** | SMILES-sequence transformers, message-passing graph networks, 3D-conformer models, tabular in-context learners over frozen embeddings, a fingerprint baseline, and an additive-fragment model |
| **Auxiliary pretraining** | Masked multi-task pretraining on public bioactivity data, warm-starting each backbone before the high-fidelity fine-tune |
| **Blind-set neighbours** | Tens of thousands of near-neighbours of the held-out compounds, pulled from a large public chemical catalogue and screened before use, so every backbone sees the neighbourhood it will be scored in |
| **Cross-validation** | Butina cluster-disjoint folds — clusters assigned whole to a single fold, so no analogue series spans train and validation |
| **Stacking** | Per-endpoint comparison of ridge / elastic-net / non-negative / NNLS combiners, selected on held-out folds |

Every reported number is out-of-fold. Nothing is scored on data the base learners saw.

## Blind Challenge Competition - Current Leaderboard Standing

**Regression** (pIC50)

{_table("Regression", ["MAE", "RMSE", "R2", "Spearman"], ["MAE", "RMSE", "R²", "Spearman ρ"])}

**Time-dependent inhibition** (binary)

{_table("Time-dependent inhibition", ["MCC", "AUC"], ["MCC", "AUC"])}

## Leaderboard history

![leaderboard history](leaderboard_history.png)

*Coloured lines are the live leaderboard, which scores half the held-out set and moves with every
submission. The dashed vertical rule marks the release of the interim leaderboard, scored on the
whole set; each line continues to its own standing there, with the overall result in purple.*

### Where this sits against the field

Every reported metric, every entry. Red is this submission, grey its predecessors, blue every
other entry on the board.

![regression standing](standing_regression.png)

![TDI standing](standing_tdi.png)

### What changed in the latest submission

![submission delta](submission_delta.png)

## Chemical neighbourhood of the held-out set

The held-out compounds occupy a particular corner of chemical space, and the labelled training
data does not densely cover it. To close that gap, **88,683 near-neighbours of the held-out
structures** were retrieved from a large public catalogue — filtered at retrieval time to the
held-out set's own physicochemical envelope and screened for structural liabilities, then kept
only above a similarity floor chosen so that every admitted compound is closer to the scored
molecules than the training set's own average nearest neighbour. The **4,999 closest** are
admitted to the training pool, entering with only their calculated physical properties and no
assay endpoints.

![chemical space](chemical_space_tsne.png)

*Chemical space of the corpus, t-SNE of AtomPair fingerprints, coloured by data tier. The
held-out compounds are black and the admitted neighbours are their own tier, so how closely the
retrieved chemistry tracks the scored compounds can be read off the map directly.*

## Structural alerts, vetoed by the blind held-out set

Auxiliary compounds are screened with [rd_filters](https://github.com/PatWalters/rd_filters) —
~1,150 named alerts across eight public rule sets. Selection is subtractive: every alert is on by
default, **any alert that fires on even one held-out compound is dropped** (~75 of 1249), then a
frequency cut removes the rest of the long tail.

## Keeping any one learner from dominating

The ensemble mixes fine-tuned molecular encoders with **tabular foundation models fitted over
their frozen embeddings**. Those tabular legs are the ones that need holding back: they are cheap
to add, they fit held-out folds extremely well, and several of them read from the *same*
embeddings, so they arrive pre-correlated and can crowd out the encoders that produced their
input. Four controls, each added after measuring a failure rather than chosen up front.

**Non-negative stacking.** Residual correlations between members run ~0.55–0.88 — the shared
residual mostly tracks *which compounds are hard*, not which model is used. Given that,
unconstrained ridge and elastic-net combiners returned large cancelling coefficients: one member
near +1.9 against another at −0.6, and on one endpoint the best standalone model pushed
**negative**, used as a correction term rather than a predictor. That fits held-out folds and
transfers badly — one revision improved cross-validated error on all four endpoints while real
held-out error got *worse* on three. The stacker is now restricted to non-negative combiners,
which forbids the cancellation structurally instead of penalising it.

**A feature budget on the tabular legs.** Each embedding source is PCA-capped before it reaches a
tabular model. With only one to two thousand labelled rows per endpoint, an uncapped
concatenation of several thousand embedding dimensions lets those models memorise the fold rather
than learn from it.

**Per-source legs, not one opaque column.** Every embedding source is fitted on its own *and* in
the combined block, so the stacker can weigh one backbone's contribution against another's
directly instead of seeing a single blended tabular prediction it must take or leave whole.

**Pruning by correlation, not by count.** More members is not reliably better. Dropping a group of
highly-correlated tabular legs improved every endpoint at once; later, dropping
*representation-diverse* members cost most of that back. What matters is how much independent
signal a member adds — on the classification track the direct encoder heads individually
outranked every tabular embedding leg, and including all of them produced the worst result on
record for one endpoint.

**Down-weighted calculated features.** A property computed from structure alone carries a label on
every row, so at full weight it supplies the large majority of the pretraining gradient and the
trunk spends its capacity reproducing a function it can already derive. Those heads are
down-weighted as a per-task weight — a constant per-row weight cancels out of a weighted mean and
changes nothing.

## Tabular foundation models

The frozen-embedding learners are fitted with **Mitra**, AutoGluon's tabular foundation model,
released under Apache 2.0. Two comparable models were evaluated on identical embeddings, TabICL
and TabPFN. TabPFN was omitted.

```python
from autogluon.tabular.configs.hyperparameter_configs import get_hyperparameter_config

ag_hp = get_hyperparameter_config("zeroshot_2025_tabfm")
for m in ["TABPFNV2"]:
    ag_hp.pop(m, None)
```

Across seven matched embedding sources Mitra beat TabPFN on macro held-out R² in six, and the
two models' residuals correlated 0.94–0.98 — they were making very nearly the same mistakes, so
leaving one out cost almost no ensemble diversity.

> Zhang, X., Maddix, D. C., Yin, J., Erickson, N., Ansari, A. F., Han, B., Zhang, S., Akoglu, L.,
> Faloutsos, C., Mahoney, M. W., Hu, C., Rangwala, H., Karypis, G., & Wang, B. (2025).
> *Mitra: Mixed Synthetic Priors for Enhancing Tabular Foundation Models.* NeurIPS 2025.
> [arXiv:2510.21204](https://arxiv.org/abs/2510.21204) ·
> [AutoGluon](https://github.com/autogluon/autogluon), Apache 2.0

## What moved the needle

- **Cluster-disjoint validation first.** Random splits flattered every model. Butina folds made
  the held-out numbers track the leaderboard, which is what made iteration meaningful at all.
- **Representation diversity beat single-model tuning.** The stacker consistently preferred a
  spread of molecular representations over more capacity in any one of them.
- **Auxiliary pretraining helps where labels are thin**, and the gain scales with how much of
  the corpus each head can actually see.
- **Calibration is separable from ranking.** Several learners ranked compounds well while being
  badly scaled; letting the stacker recalibrate recovered most of that.
- **TDI is a different problem from potency.** Features that predict inhibition strength carry
  little about time-dependent inactivation, which is mechanism-driven.

## Stack

RDKit · PyTorch · Chemprop · scikit-learn · HuggingFace Transformers · AutoGluon

*Open-source components only.*
