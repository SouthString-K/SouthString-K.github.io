# SCMamba-YOLO: Edge-Guided SS2D Modeling for Submarine Cable Detection

## 1. Document Purpose

This document is written for the current `SCMamba-YOLO` codebase and is used to explain:

- the overall model structure
- the role of Mamba and `SS2D`
- the key modifications introduced in this version
- the enhancement front-end integrated before detection
- why the current design is suitable for submarine cable detection

The current main configuration file is:

- `ultralytics/cfg/models/mamba-yolo/Mamba-YOLO-T-EG.yaml`

The current core modified modules are:

- `ultralytics/nn/modules/mamba_yolo.py`
- `ultralytics/nn/modules/block.py`
- `ultralytics/nn/modules/enhance_front.py`

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

- `EnhanceFrontEnd`
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

## 13. Enhancement Front-End Integration

Besides the backbone-level modification, the current code also integrates an underwater image enhancement network as a front-end module before the detector.

Its overall inference pipeline becomes:

```text
input image -> enhancement front-end -> SCMamba-YOLO detector -> detection result
```

The purpose of this design is straightforward:

- underwater images often suffer from low contrast
- color information is frequently degraded
- local boundaries are often blurred before detection even starts

So instead of sending the raw degraded image directly into the detector, the model first enhances the visual quality and then performs detection on the enhanced result.

### 13.1 Integration Strategy

This fusion is implemented as a network-level integration rather than an offline preprocessing pipeline.

That means:

- the enhancement module is written into the model `yaml`
- the image first passes through the enhancement front-end during forward propagation
- the enhanced image is then fed into the original SCMamba-YOLO backbone, neck, and head
- the main detector structure is kept unchanged

Its logic can be written as:

```text
I_enh = E(I)
Y     = D(I_enh)
```

Where:

- `E(.)` denotes the enhancement front-end
- `D(.)` denotes the SCMamba-YOLO detector
- `I` is the raw input image
- `I_enh` is the enhanced image
- `Y` is the final detection output

### 13.2 Code Modification Locations

The front-end enhancement integration is distributed across several parts of the codebase.

#### New enhancement module

New file:

- `ultralytics/nn/modules/enhance_front.py`

Its role is:

- to wrap `InteractNet` from `Enhancement-main/src/CDF_UIE_arch.py`
- to place it as the first network layer before detection
- to output the enhanced image used by the detector

#### Module registration

The enhancement module is registered in:

- `ultralytics/nn/modules/__init__.py`
- `ultralytics/nn/tasks.py`

This makes sure:

- `EnhanceFront` can be parsed directly in `yaml`
- Ultralytics can recognize and build the module correctly

#### Enhanced model configurations

New enhanced configurations include:

- `SCMamba-YOLO-T-Enhance.yaml`
- `SCMamba-YOLO-B-Enhance.yaml`
- `SCMamba-YOLO-L-Enhance.yaml`

The original configurations are still preserved:

- `SCMamba-YOLO-T.yaml`
- `SCMamba-YOLO-B.yaml`
- `SCMamba-YOLO-L.yaml`

The difference is simple:

- original versions: raw image goes directly into the detector
- enhanced versions: image first passes through `EnhanceFront`, then enters the detector

### 13.3 Training and Validation Entry

The related entry files remain:

- training entry: `train.py`
- validation entry: `val.py`

Additional arguments include:

- `--enhance_weights`
- `--enhance_freeze`

Their purposes are:

- `--enhance_weights`: load pretrained weights for the enhancement front-end
- `--enhance_freeze`: freeze enhancement parameters and train only the detector

### 13.4 Actual Training Behavior

During training, the workflow is not:

- enhance all images offline first
- then train the detector on saved enhanced images

Instead, the enhancement is performed online during batch forwarding.

For example:

- input batch: `[B, 3, 640, 640]`
- first passes through the enhancement front-end
- then the enhanced batch is fed into the detector

This can be written as:

```text
X_enh = E(X)
Y_hat = D(X_enh)
```

Where `X` denotes a batch of input images.

This means:

- every image is enhanced before detection
- but the computation is still done batch-wise for GPU efficiency

### 13.5 Weight Loading Logic

For the enhanced versions:

- `SCMamba-YOLO-*-Enhance.yaml`

the training script requires enhancement weights to be provided through:

- `--enhance_weights /path/to/interactnet.pth`

If the weights are not given, the script should report an error and remind the user to train the enhancement model separately through:

- `Enhancement-main/src/train.py`

The front-end is also designed to handle checkpoints saved with `DataParallel`, including the common:

- `module.xxx`

parameter prefix.

### 13.6 Why This Front-End Matters

This enhancement front-end is not introduced just to improve image appearance.

Its importance lies in the fact that it provides a better input domain for the detector:

- clearer color contrast before feature extraction
- more recoverable local boundaries
- reduced underwater visual degradation before Mamba-based modeling starts

This is especially meaningful for submarine cable detection because the target is:

- elongated
- often low-contrast
- frequently submerged in visually degraded backgrounds

So the enhancement front-end and the detector backbone play complementary roles:

- `EnhanceFront` improves the input image domain
- `EGVSSBlock` improves the internal feature representation
- `SS2D` preserves long-range structural continuity

In other words, the current model is no longer only a backbone-enhanced detector. It is a detector with both:

- input-level enhancement
- feature-level enhancement

### 13.7 Updated Model Perspective

With the front-end enhancement module included, the full logic of the model can now be summarized as:

```text
raw image
-> enhancement front-end
-> edge-guided local and global modeling
-> neck fusion
-> detection head
```

This makes the current codebase more suitable for underwater deployment than a plain detector that directly consumes degraded input images.

## 14. Conclusion

For the current code version, the real personal contribution can be summarized as:

1. reconstructing the original `VSSBlock` into `EGVSSBlock`
2. designing `EdgeGuidedEnhance` with a Sobel branch and a context branch
3. improving the feature quality before `SS2D` without changing the backbone interface
4. integrating an enhancement front-end before the detector for underwater visual restoration
5. making the full pipeline more suitable for elongated, continuous, and weak-boundary submarine cable detection

In one sentence:

> `SCMamba-YOLO-T-EG` is a submarine cable detection model that combines front-end image enhancement with edge-guided backbone modeling, and its core value lies in the collaborative mechanism of visual restoration, boundary enhancement, local topology preservation, SS2D long-range modeling, and gated refinement.

## 15. Open-Source Repository

The open-source repository of this project is:

https://github.com/SouthString-K/SCMamba-YOLO
