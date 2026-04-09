---
layout: post
title: "Deploying a systolic CNN on the Zynq NPU"
tags: [hardware, fpga]
---

Walking through the full pipeline from trained INT8 model to AXI-DMA 
transfer on the Zynq-7020 fabric.

## The problem with tutorials

Every tutorial stops at "send your data over AXI." What happens 
between that sentence and a working system is where the time goes.

## The DMA transfer path

Allocate directly from `pynq.allocate` and fill in-place — never 
copy from a NumPy buffer. The copy crosses coherence boundaries twice.

```python
buf = pynq.allocate(shape=(784,), dtype=np.int8)
buf[:] = image.flatten()
dma.sendchannel.transfer(buf)
dma.sendchannel.wait()
```

## Cache coherence

The PS and PL do not share a cache. AXI HP ports bypass L2. 
If results are wrong on the first run and correct on the second, 
that is a cache coherence problem, not a model problem.