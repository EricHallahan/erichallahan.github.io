---
title: Building a Cohesive ML Stack from Scratch for Containers with System Packages
date: 2023-04-30
description: Sometimes it pays to be able to build from source. 
tags: ["Linux", "Machine Learning", "Deep Learning", "Fedora", "PyTorch", "JAX", "CUDA", "Containers", "Docker", "Kubernetes"]
draft: true
---



## Why build from scratch?

Having 


## Writing packages for Red Hat-adjacent distributions

Having originally targeted EPEL 9, I switched to targeting Fedora 37 as to leverage it's greater availablity of packages. This also forced me into using 

### Introduction to the RPM workflow



## Building containers in Kubernetes

One of the perrenial issues I ran into using CoreWeave's infrastructure was that, in order to deploy an image, you have to have one you want first. 

### Kaniko

#### Limitations

Kaniko is quite limited in what it can do, and it isn't al

