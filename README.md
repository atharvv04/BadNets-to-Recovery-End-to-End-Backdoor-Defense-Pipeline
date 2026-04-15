# BadNets to Recovery: End-to-End Backdoor Defense Pipeline

This repository implements the full end-to-end pipeline for a **BadNets-style all-to-one backdoor attack**, **Activation Clustering-based poison detection**, and **repair by filtering suspicious samples and retraining** on **CIFAR-10** with **ResNet-18**.

## Objective

The goal of this project is to study the full lifecycle of a backdoor attack in a controlled setting:

1. train a clean image classifier,
2. poison a small fraction of the training data with a fixed trigger,
3. train a compromised model,
4. detect suspicious poisoned samples using hidden representations,
5. remove suspicious samples,
6. retrain a repaired model,
7. compare clean accuracy and attack success before and after repair.

This setup demonstrates why a model can appear healthy under ordinary evaluation while still containing a hidden attacker-controlled behavior.

---

## Implemented Pipeline

### 1. Dataset and model setup

- **Dataset:** CIFAR-10
- **Model:** ResNet-18
- **Target class:** class `0`
- **Validation split:** balanced clean validation set with **500 images per class**
- **Training split:** remaining **45,000** training images used for clean training and poison construction.

### 2. Backdoor attack

I implemented a **BadNets-style all-to-one attack** with the following design:

- **Poison rate:** `5%` of non-target training examples
- **Trigger:** `3×3` white square
- **Placement:** bottom-right corner with a `1-pixel` offset from the border
- **Poisoning rule:** apply trigger to selected non-target samples and relabel them to the target class
- **Number of poisoned samples:** `2025` out of `40,500` non-target training examples.

This produces a compromised training set while keeping the attack simple, reproducible, and easy to describe.

### 3. Clean and poisoned model training

Two separate models were trained:

- a **clean baseline model** on the clean training split
- a **poisoned model** on the poisoned training split

Both models use the same architecture and training setup so that the only meaningful difference is the presence of poisoned samples.

### 4. Attack evaluation

The attack is evaluated using two metrics:

- **Clean Test Accuracy:** accuracy on the untouched CIFAR-10 test set
- **Attack Success Rate (ASR):** fraction of triggered **non-target** test images predicted as the target class

This separation is important because clean accuracy measures standard utility, while ASR measures whether the backdoor has been learned.

### 5. Activation Clustering

To detect poison, I implemented **Activation Clustering** on the **penultimate-layer features** of samples whose **observed label equals the target class**.

Pipeline:
1. extract penultimate-layer features from the trained poisoned model,
2. restrict to target-labeled training samples,
3. standardize features,
4. reduce dimensionality using **FastICA** to `10` dimensions,
5. cluster the reduced features using **k-means** with `k = 2`,
6. mark the **smaller cluster** as suspicious.

This works because poisoned target-labeled samples tend to form a minority structure in representation space.

### 6. Repair

The repair stage removes all samples assigned to the suspicious cluster, then retrains a **fresh ResNet-18 from scratch** on the filtered dataset. The repaired model is evaluated on:

- the clean CIFAR-10 test set
- the triggered non-target test set used for ASR evaluation.

---

## Key Results

## Task 1A - Backdoor attack results

| Model | Clean Test Accuracy | ASR |
|---|---:|---:|
| Clean baseline | 93.27% | 0.76% |
| Poisoned model | 92.58% | 98.06% |

Interpretation:
- the poisoned model preserves high clean accuracy,
- but the trigger causes almost all non-target test samples to be redirected to the target class,
- so the backdoor attack is highly successful.

## Task 1B - Activation Clustering results

Target-labeled subset used for clustering:
- **Total clustered samples:** `6525`
- **True poisoned samples in this subset:** `2025`

Cluster sizes:
- **Cluster 0:** `4527`
- **Cluster 1:** `1998`
- **Suspicious cluster:** smaller cluster = `Cluster 1`

Poison detection performance:

| Metric | Value |
|---|---:|
| Precision | 99.85% |
| Recall | 98.52% |
| F1 Score | 99.18% |

Interpretation:
- the suspicious cluster aligns almost perfectly with the poisoned samples,
- poisoned and clean target-labeled examples are strongly separable in hidden representation space,
- Activation Clustering works extremely well in this controlled backdoor setting.

## Task 1C - Repair results

| Model | Clean Test Accuracy | ASR |
|---|---:|---:|
| Clean baseline | 93.27% | 0.76% |
| Poisoned model | 92.58% | 98.06% |
| Repaired model | 92.62% | 1.91% |

Interpretation:
- repair reduces ASR from **98.06%** to **1.91%**
- clean accuracy remains close to the poisoned and clean baselines
- the defense is highly effective, though not perfect, since repaired ASR remains slightly above the clean baseline.

---

## Why the pipeline works

### Why the attack works
The poisoned model learns a shortcut: the trigger becomes a strong feature associated with the target label. Because the poison rate is small, ordinary clean accuracy remains high, so the model still looks normal under standard testing.

### Why Activation Clustering works
The penultimate layer encodes semantically meaningful structure. Once the model has learned the backdoor, poisoned target-labeled samples tend to occupy a different subregion of feature space from genuine target-class samples. Clustering in hidden space is therefore much more effective than clustering in raw pixel space.

### Why repair works
Filtering suspicious samples removes most of the poisoned data responsible for the backdoor. Retraining from scratch on the filtered dataset weakens the trigger-target association and restores near-clean behavior.

---

## Repository Outputs

The implementation generates:

- trained checkpoints for:
  - clean baseline model
  - poisoned model
  - repaired model
- saved metrics in JSON/CSV form
- metadata files for train/validation splits and poison assignments
- visualizations:
  - `attack_summary.png`
  - `clustering_visualization.png`
  - `repair_summary.png`

---

## Technical Summary

This project implements a complete and interpretable backdoor analysis pipeline:
- a simple but effective BadNets poisoning attack,
- representation-based poison detection using Activation Clustering,
- repair through suspicious-sample removal and retraining.

The results show three main things:
1. **clean accuracy alone is not a sufficient safety metric**, because the poisoned model still achieved high clean accuracy while being heavily compromised;
2. **hidden representations are useful for detecting poisoned samples**, since they made poisoned and clean target-labeled examples strongly separable;
3. **filtering and retraining can substantially mitigate a simple backdoor**, reducing ASR from **98.06%** to **1.91%** while preserving most clean performance.

---

## Notes

This implementation is designed for a controlled setting, not as a production-grade backdoor defense. The smaller-cluster heuristic works well here because:
- the poison rate is low,
- the trigger is fixed and visible,
- poisoned samples form a separable minority structure.

More adaptive backdoor attacks, such as semantic or distributed triggers, may weaken this defense and would require stronger validation strategies beyond simple size-based clustering.