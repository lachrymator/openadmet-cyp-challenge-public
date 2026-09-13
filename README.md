# OpenADMET CYP Inhibition Challenge

Stacked ensemble for the [OpenADMET CYP Inhibition Blind Challenge](https://huggingface.co/spaces/openadmet/cyp-challenge)
— **pIC50 regression** across four cytochrome P450 isoforms, and **time-dependent inhibition
(TDI)** classification for two of them.

Best standing to date: **macro rank 10 of 146** on the regression track, with three of the four
endpoints inside the top 11.

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

## Held-out performance

**Regression** (pIC50)

| endpoint | MAE | RMSE | R² | Spearman ρ |
|---|---|---|---|---|
| CYP1A2 | 0.433 | 0.592 | 0.667 | 0.774 |
| CYP2C9 | 0.327 | 0.443 | 0.667 | 0.834 |
| CYP2D6 | 0.442 | 0.654 | 0.487 | 0.788 |
| CYP3A4 | 0.349 | 0.469 | 0.813 | 0.910 |
| **macro average** | **0.388** | **0.540** | **0.658** | **0.827** |

**Time-dependent inhibition** (binary)

| endpoint | MCC | AUC |
|---|---|---|
| CYP2D6 (TDI) | 0.228 | 0.645 |
| CYP3A4 (TDI) | 0.517 | 0.845 |
| **macro average** | **0.372** | **0.745** |

## Leaderboard history

![leaderboard history](leaderboard_history.png)

### Where this sits against the field

Every reported metric, every entry. Red is this submission, grey its predecessors, blue every
other entry on the board.

![regression standing](standing_regression.png)

![TDI standing](standing_tdi.png)

### What changed in the latest submission

![submission delta](submission_delta.png)

**Regression**

| endpoint | first | best | latest | entries |
|---|---|---|---|---|
| CYP1A2 | 123 | **3** | 3 | ~146 |
| CYP2C9 | 110 | **6** | 6 | ~146 |
| CYP2D6 | 115 | **33** | 33 | ~146 |
| CYP3A4 | 81 | **11** | 11 | ~146 |

**Time-dependent inhibition**

| endpoint | first | best | latest | entries |
|---|---|---|---|---|
| CYP2D6 (TDI) | 65 | **13** | 14 | ~81 |
| CYP3A4 (TDI) | 49 | **3** | 27 | ~81 |

## Chemical neighbourhood of the held-out set

The held-out compounds occupy a particular corner of chemical space, and the labelled training
data does not densely cover it. To close that gap, **over 50,000 near-neighbours of the held-out
structures** were retrieved from a large public catalogue — filtered at retrieval time to the
held-out set's own physicochemical envelope and screened for structural liabilities, then kept
only above a similarity floor chosen so that every admitted compound is closer to the scored
molecules than the training set's own average nearest neighbour.

They carry **no assay labels at all** — nothing was measured on them, and nothing is invented.
What they do carry is a *calculated physical property*, computed for every compound in the
project by one model so the value means the same thing on every row.

That single head is what lets them into training. Each backbone is pretrained multi-task with the
loss masked per head, so a row contributes through whichever heads it actually has: a measured
compound trains the assay heads, a neighbour trains only the physicochemical one. Without that
head a neighbour has no label in any task and is dropped before the first gradient step — with
it, the same rows warm-start the shared trunk on the chemistry surrounding the held-out
compounds.

So the neighbours do not teach the model about potency. They teach the *encoder* what this
region of chemical space looks like, before it ever sees a label from it — and the representation
the downstream heads are built on is fitted on that wider neighbourhood rather than only where
labels happen to exist.

## Structural alerts, and letting the held-out set veto them

Auxiliary compounds are screened with [rd_filters](https://github.com/PatWalters/rd_filters) —
roughly 1,150 individually named alerts drawn from eight public rule sets (PAINS, BMS, Dundee,
Glaxo, Inpharmatica, LINT, MLSMR, SureChEMBL). The selection is **subtractive**, in that order:

1. **Every alert is on by default.** An alert encodes a real liability; the burden of proof is on
   switching one *off*.
2. **The held-out set vetoes.** Any alert that fires on even one held-out compound is dropped —
   permanently, no appeal. 75 of ~1,150 are removed this way.
3. **A frequency cut** over what survives, so a rule seen once or twice is not acted on.

Step 2 is the one that earns its place. We are scored on those molecules, so an alert condemning
them is describing chemistry the corpus must **contain**, not chemistry to strip out. Because the
veto is absolute, no surviving alert can fire on a held-out compound *by construction* — which
means the neighbour screen can never reject a candidate for carrying chemistry the held-out set
itself has. That property is free once the ordering is right, and unavailable at any price if the
filter is chosen first and checked afterwards.

The same screen is applied identically to the training pool and to the retrieved neighbours, so
the two populations are filtered on one standard rather than drifting apart.

## Keeping any one learner from dominating

Both constraints below were added after measuring a failure, not chosen up front.

**Non-negative stacking.** The base learners are heavily correlated — pairwise residual
correlations of roughly 0.55 to 0.88, because the shared residual largely tracks *which
compounds are hard* rather than which model is used. Given that, unconstrained ridge and
elastic-net combiners returned large cancelling coefficients: one learner near +1.9 against
another at −0.6, and on one endpoint the best standalone model was pushed **negative** — used as
a correction term rather than as a predictor. That fits held-out folds well and transfers badly.
One revision improved cross-validated error on all four endpoints while real held-out error got
*worse* on three. The stacker is now restricted to non-negative combiners, which forbids that
cancellation structurally rather than hoping a penalty discourages it.

**Down-weighted calculated features.** A head computed from structure alone carries a label on
every row, so left at full weight it supplies the large majority of the pretraining gradient and
the trunk spends its capacity reproducing a deterministic function it can already derive. Those
heads are explicitly down-weighted relative to measured assay signal, as a per-task weight rather
than a per-row one — a constant per-row weight cancels out of a weighted mean and changes
nothing.

**Pruning by correlation, not by count.** More members is not reliably better. Dropping several
highly-correlated learners improved every endpoint at once; later, dropping
*representation-diverse* ones cost nearly as much as it had gained. The axis that matters is how
much independent signal a member adds, not how many members there are.

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
