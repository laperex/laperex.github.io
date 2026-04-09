---
layout: post
title: "INT8 quantization without losing your mind"
date: 2025-01-28
description: "Post-training quantization, calibration datasets, and the three failure modes that eat your accuracy silently. A minimal PyTorch recipe that works."
tags: [ml, quantization]
reading_time: 11
---

Post-training quantization is straightforward until it isn't. Here are the failure modes I've hit and how to avoid them.

<!--more-->

## Why INT8

Not because it's fashionable. A systolic array running INT8 MACs is roughly 4× cheaper in silicon than FP32, and on MNIST-class tasks the accuracy hit is under 0.5% when you quantize correctly. The phrase "when you quantize correctly" is doing a lot of work there.

## Three silent failures

**Calibrating on training data.** Your calibration set should be drawn from the test distribution. Training data has augmentation artifacts that shift activation distributions. Use a small, clean subset of held-out data — 512 samples is enough for most vision tasks.

**Symmetric vs. asymmetric mismatch.** If your hardware uses symmetric INT8 (range −127 to 127) but PyTorch's quantizer defaults to asymmetric (0 to 255 with a zero-point offset), your exported model will produce wrong outputs on hardware with no error. Check the quantization scheme against your target explicitly. Check it again.

**Per-tensor vs. per-channel.** Per-channel weight quantization almost always outperforms per-tensor for convolutional layers. Most modern NPUs support it. Use it unless you have a specific reason not to.

## The minimal recipe

```python
import torch
import torch.ao.quantization

model.eval()

# Match this qconfig to your hardware's quantization scheme
model.qconfig = torch.ao.quantization.get_default_qconfig('x86')

torch.ao.quantization.prepare(model, inplace=True)

# Calibrate — no gradients, held-out data only
with torch.no_grad():
    for images, _ in calibration_loader:
        model(images)

torch.ao.quantization.convert(model, inplace=True)
```

After conversion, verify accuracy on the full test set before exporting. If you see more than 1% degradation, the cause is almost always one of the three failures above.

## Checking what you actually got

```python
# Inspect quantization parameters of a layer
print(model.conv1.weight().q_per_channel_scales())
print(model.conv1.weight().q_per_channel_zero_points())
```

If zero points are all zero, you're using symmetric quantization. If they're non-zero, asymmetric. Know which one your hardware expects before you export.