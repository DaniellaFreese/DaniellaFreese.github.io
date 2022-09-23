---
title: "Understanding Envoy Proxy"
date: 2022-09-06
draft: false

categories: ["Service Mesh"]
tags: ["service mesh", "Kong Mesh", "Kuma", "Envoy Proxy"]
toc: false
author: "djfreese"
---

At this point Service Mesh related techologies have been on the rise since around 2017 and many of us are familiar with the solutions out there: Istio, Linkerd, Kong Mesh. Many of the existing service mesh technologies are built on top of Envoy Proxy (certainly not all of them are but many are built on Enovy) which was originally designed to support mesh-like architectures.

Well, Kong Mesh was the first mesh I had used that exposed the ablity to actually configure the Envoy Proxy sidecar - with the [Proxy Template](https://kuma.io/docs/1.7.x/policies/proxy-template/). In order to leverage this resource properly does require some familiarity with Envoy and how to configure it. Because of this I took a small detour to familiarize myself with the fundamentals of Envoy Proxy. Below is the list of my favorite resources that helped grasp the fundamentels of Envoy.

[Service Mesh 102: Envoy Configuration](https://konghq.com/blog/envoy-service-mesh-configuration): this helps break down the fundmentals Envoy Proxy into its core components into a really easy to digest blog.

[Course - Envoy Fundamentals](https://academy.tetrate.io/): it basically runs through Envoy Documenation but the course does a great job with the videos, providing sample configurations, labs that can actually run on your machine (assuming you have docker), and the material is short.
