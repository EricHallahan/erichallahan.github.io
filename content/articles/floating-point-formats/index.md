---
title: Floating Point Formats and Where to Find Them
date: 2023-04-07
description: A look at how I came to compensate for my hearing when I am not wearing a hearing aid.
tags: ["Computer Engineering"]
type: blogpost
draft: true
---

## IEEE 754-1985: The introduction of standard floating-point formats


## IEEE 754-2008: A  


## bfloat16: Google Brain cuts 

As far as I am aware, it is indeed the TensorFlow whitepaper that introduces the concept of bfloat16, though not for the reason we would suspect given the applications it is popular for today: Google Brain wanted to reduce bandwidth between nodes, and to do so  


## TensorFloat32: NVIDIA misleadingly markets a frankenformat


Because customers expect to use both binary16 and bfloat16 on Tensor Cores, NVIDIA needed to implement both.  


This raises the question: If TensorFloat32 is not a 32-bit format, why does have “32” in the name? Well, that is the part that makes things complicated.

In order to keep the aspect of a “free” expanded-precision Tensor Core floating-point format, NVIDIA had to make a compromise somewhere, and that compromise comes in the form of how the data is stored in memory—because TensorFloat32 is greater than 16 bits long, to access it efficiently each data element *needs* to be four bytes long. Otherwise, expensive additional circuitry would need to be integrated for a dedicated memory access pattern, defeating the point of a clever additional feature like this.

This means that, despite being smaller than four bytes, there is no opportunity to save memory—TensorFloat32 is purely a format to accelerate computation by leveraging Tensor Cores in a way they wouldn't otherwise be able to. This justifies the name well: From perspective of memory, it is no different than if Tensor Cores could actually do computation in IEEE 754 single-precision floating point.

However, when shifting to the numerical perspective, TensorFloat32 is simply an expanded replacement for 16-bit floating-point formats for when they are unsuitable or unstable and the memory and bandwidth penalty of TensorFloat32 is acceptable. This means that **enabling TensorFloat32 is *not* a suitable replacement for doing computation in IEEE 754 single-precision floating point, and should never be used as such unless loss of precision is acceptable**. Many software packages mistakenly assumed (based upon NVIDIA's unclear marketing) that it would be safe to enable TensorFloat32 computation by default on A100s. This is not true in the slightest, and since then many of those packages have reversed that decision.