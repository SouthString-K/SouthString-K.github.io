# SCMamba-YOLO: Edge-Guided SS2D Modeling for Submarine Cable Detection

## 1. Document Purpose

This document is written for the current `SCMamba-YOLO` codebase and is used to explain:

- the overall model structure
- the role of Mamba and `SS2D`
- the key modifications introduced in this version
- why the current design is suitable for submarine cable detection

The current main configuration file is:

- `ultralytics/cfg/models/mamba-yolo/Mamba-YOLO-T-EG.yaml`

The current core modified modules are:

- `ultralytics/nn/modules/mamba_yolo.py`
- `ultralytics/nn/modules/block.py`

## 2. Overall Model Positioning

`SCMamba-YOLO` is a submarine cable detection model built on top of the baseline `Mamba-YOLO-T`.

The main idea is not to rewrite the whole detector, but to preserve the original detection pipeline and strengthen the backbone representation for the target task.

The current design keeps:

1. the overall detector framework
2. the original neck
3. the original detection head

The main change is concentrated in the backbone, especially in the feature extraction stage before `SS2D`.

So in essence, this model is not a brand-new detection framework. It is a task-oriented improvement of the original Mamba-YOLO architecture for submarine cable detection.

## 3. Model Structure Overview

From the YAML configuration, `SCMamba-YOLO` still follows the standard three-stage structure:

1. Backbone
2. PAFPN-style Head
3. Detect Head

### 3.1 Backbone

The backbone structure is:

- `SimpleStem`
- `EGVSSBlock x 3`
- `VisionClueMerge`
- `EGVSSBlock x 3`
- `VisionClueMerge`
- `EGVSSBlock x 9`
- `VisionClueMerge`
- `EGVSSBlock x 3`
- `SPPF`

### 3.2 Head

The head is not rewritten. It still keeps the original multi-scale fusion and detection design, including:

- upsampling
- `Concat`
- `XSSBlock`
- final `Detect`

This is important because it means the main contribution of this work is focused on backbone enhancement rather than stacking extra modules in the neck or head.

## 4. The Role of the Mamba Module in SCMamba-YOLO

In the original Mamba-YOLO backbone, the core feature block is `VSSBlock`.

Its major responsibilities can be summarized as:

1. channel projection
2. local structure extraction
3. long-range dependency modeling with `SS2D`
4. gated feature refinement

Its typical feature flow can be written as:

```text
Fp   = Proj(Fin)
Fl   = LSBlock(Fp)
Fs   = SS2D(Norm(Fl))
Fm   = Fp + Fs
Fout = Fm + RGBlock(Norm2(Fm))
```

The strength of the original `VSSBlock` is that it can model long-range dependencies efficiently, which is naturally suitable for elongated targets.

However, for submarine cable detection there is still a key limitation:

> The original block is good at long-range modeling, but it does not explicitly compensate for weak boundaries, low contrast, and blurred local structures in underwater scenes.

This is exactly where the current improvement starts.

## 5. Basic Principle of Mamba and SS2D

To understand why `EGVSSBlock` is meaningful, we first need to clarify the role of `SS2D`.

### 5.1 State Space Modeling

The core idea of Mamba-style modeling comes from state space models.

Its continuous form can be written as:

```text
dh(t) / dt = A * h(t) + B * x(t)
y(t) = C * h(t)
```

Where:

- `x(t)` is the input
- `h(t)` is the hidden state
- `y(t)` is the output
- `A`, `B`, and `C` are state transition related parameters

After discretization, it can be written as:

```text
ht = A_bar * h(t-1) + B_bar * xt
yt = C * ht
```

Where `A_bar` and `B_bar` are the discretized transition parameters.

### 5.2 Why SS2D Is Used in Vision

Visual features are two-dimensional, so they cannot be treated as a plain one-dimensional sequence directly.

The core idea of `SS2D` is:

1. flatten the 2D feature map into sequences under multiple scan orders
2. perform state space modeling in sequence space
3. map the modeled features back to the 2D feature map

This process can be abstracted as:

```text
Fs = SS2D(F)
```

So `SS2D` is not just another convolution. It usually includes:

- local transform
- selective scan in multiple directions
- sequence-to-image restoration

### 5.3 Why SS2D Fits Submarine Cable Detection

Submarine cable targets have several distinctive properties:

- elongated
- continuous
- cross-region extension
- locally weak boundaries but globally coherent structure

This means local convolution alone may easily lead to:

- broken boundaries
- local target disappearance
- insufficient long-range continuity modeling

The advantage of `SS2D` is that it can bring distant structural relations into one modeling process and is therefore naturally friendly to elongated continuous targets.

That is why `SS2D` is the irreplaceable long-range continuity modeling part in this task.

## 6. Core Modification: EGVSSBlock

### 6.1 Modification Location

This work does not discard the original `VSSBlock`.

Instead, an edge-guided enhancement unit is inserted before `LSBlock + SS2D`, forming a new block:

```text
EGVSSBlock = Projection + EdgeGuidedEnhance + LSBlock + SS2D + RGBlock
```

And the enhanced residual branch can be written as:

```text
Fr = Fp + Fe
```

This means the original backbone now receives a structure-enhanced feature before entering the local topology and long-range modeling stages.

### 6.2 Motivation of the Modification

The motivation comes directly from the submarine cable task:

1. underwater scattering and turbidity weaken boundaries
2. submarine cables are thin and elongated, so local breakage is harmful
3. `SS2D` also needs relatively clean structural input
4. therefore, explicit boundary enhancement is needed before state space modeling

In one sentence:

> `EGVSSBlock` does not replace `SS2D`; it prepares better input features for `SS2D` in submarine cable detection.

## 7. Design of EdgeGuidedEnhance

`EdgeGuidedEnhance` is defined in:

- `ultralytics/nn/modules/block.py`

It is the most important newly introduced module in the current version.

### 7.1 Module Structure

It contains two branches:

1. boundary response branch
2. context branch

#### Branch A: Boundary Response Branch

For the input feature `Fp`, Sobel operators are used to extract horizontal and vertical gradients:

```text
Gx = Sobel_x(Fp)
Gy = Sobel_y(Fp)
```

Then the gradient magnitude is calculated:

```text
G = sqrt(Gx * Gx + Gy * Gy + eps)
```

After that, a `1x1` convolution is used to map it into boundary features:

```text
Fbd = Conv1x1(G)
```

#### Branch B: Context Branch

The context branch applies average pooling on the same input:

```text
Fctx = Conv1x1(AvgPool3x3(Fp))
```

Its purpose is:

- to preserve local contextual structure
- to avoid making the enhancement branch overly dependent on gradients only

#### Branch Fusion

The two branches are concatenated and fused:

```text
Fe = Conv3x3(Conv1x1(Concat(Fbd, Fctx)))
```

The final output is the enhanced feature `Fe`.

### 7.2 Why This Design Matters

This design has several advantages:

- the Sobel branch strengthens weak boundary response
- the pooling branch supplements local context
- the fused result is not a pure edge map but a structure-enhanced feature map
- the output keeps the same shape and channel size, so it can be inserted into the backbone directly

## 8. Full Forward Flow of EGVSSBlock

According to the current implementation, the forward process of `EGVSSBlock` can be summarized as follows.

### 8.1 Projection

```text
Fp = P(Fin)
```

Where `P(.)` denotes:

- `1x1 Conv`
- `BatchNorm`
- `SiLU`

### 8.2 Edge-Guided Residual Enhancement

```text
Fe = E(Fp)
Fr = Fp + Fe
```

Where `E(.)` denotes `EdgeGuidedEnhance`.

### 8.3 Local Topology Aggregation

```text
Fl = L(Fr)
```

Where `L(.)` denotes `LSBlock`.

### 8.4 SS2D Long-Range Modeling

```text
Fs = S(Norm(Fl))
```

Where `S(.)` denotes `SS2D`.

### 8.5 Second Residual Fusion

```text
Fm = Fr + Fs
```

### 8.6 Gated Refinement

```text
Fg   = G(Norm2(Fm))
Fout = Fm + Fg
```

Where `G(.)` denotes `RGBlock`.

## 9. Relation Between LSBlock, RGBlock, and the Current Modification

### 9.1 LSBlock

`LSBlock` already exists in the original model and is used for local topology aggregation:

```text
Fl = LSBlock(Fr)
```

In the current model, the code of `LSBlock` is not rewritten, but its input is changed from the original projected feature to the enhanced feature `Fr`.

That means `LSBlock` now works on clearer structural cues.

### 9.2 RGBlock

`RGBlock` is also not newly introduced. It is the gated refinement branch inherited from the original `VSSBlock`.

Its internal logic can be described as:

```text
[X, V] = Split(Conv1x1(Fm))
X_hat  = Sigmoid(DWConv(X) + X)
Fg     = Conv1x1(X_hat * V)
```

Where:

- `DWConv` denotes depthwise convolution
- `*` denotes element-wise multiplication
- `V` acts as a gating modulation branch

In this work, `RGBlock` is not rewritten, but it receives better features after edge enhancement and long-range modeling, which helps suppress noisy backgrounds while preserving meaningful cable responses.

## 10. Difference from the Baseline

At the code level, the difference between the baseline and the current version is very clear.

### Original VSSBlock

```text
Fp -> LSBlock -> SS2D -> RGBlock
```

### Current EGVSSBlock

```text
Fp -> EdgeGuidedEnhance -> (Fp + Fe) -> LSBlock -> SS2D -> RGBlock
```

The essential idea is:

> for submarine cable targets with weak boundaries, elongated continuity, and complex background interference, an explicit edge-guided enhancement stage is inserted before `LSBlock + SS2D`, so that the subsequent state space modeling can start from more suitable task-oriented features.

## 11. Why This Modification Fits Submarine Cable Detection

From the task perspective, submarine cable detection has three major difficulties:

1. weak boundaries
2. elongated and continuous target structure
3. strong background interference

The current design addresses them in a one-to-one way:

- `EdgeGuidedEnhance` strengthens weak boundaries
- `LSBlock` preserves local topology
- `SS2D` models long-range continuity
- `RGBlock` performs gated refinement and background suppression

So `EGVSSBlock` is not just a generic enhancement block. It is a task-oriented feature block designed around the pain points of submarine cable detection.

## 12. Engineering Characteristics of the Current Configuration

From an engineering point of view, the current version also has several clear advantages:

1. input and output interfaces remain unchanged
2. neck and head are not rewritten
3. it is convenient for ablation experiments
4. it is easy to maintain and deploy

This means you can clearly compare:

- original `T`
- `T-EG`
- future lightweight or gated variants

and attribute performance changes more convincingly to the backbone modification itself.

## 13. Conclusion

For the current code version, the real personal contribution can be summarized as:

1. reconstructing the original `VSSBlock` into `EGVSSBlock`
2. designing `EdgeGuidedEnhance` with a Sobel branch and a context branch
3. improving the feature quality before `SS2D` without changing the backbone interface
4. making the backbone more suitable for elongated, continuous, and weak-boundary submarine cable detection

In one sentence:

> `SCMamba-YOLO-T-EG` is a submarine cable detection model that introduces edge-guided enhancement into the backbone, and its core value lies in the collaborative mechanism of boundary enhancement, local topology preservation, SS2D long-range modeling, and gated refinement.

## 14. Open-Source Repository

The open-source repository of this project is:

https://github.com/SouthString-K/SCMamba-YOLO
