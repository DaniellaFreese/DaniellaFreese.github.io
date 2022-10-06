---
title: "Resources for Understanding Envoy Proxy"
date: 2022-09-06
draft: false

categories: ["Service Mesh"]
tags: ["service mesh", "Kong Mesh", "Kuma", "Envoy Proxy"]
toc: false
author: "djfreese"
---

At this point Service Mesh related techologies have been on the rise since around 2017 and many of us are familiar with the solutions out there: Istio, Linkerd, Kong Mesh. Many of the existing service mesh technologies are built on top of Envoy Proxy (certainly not all of them are but many are built on Enovy) which was originally designed to support mesh-like architectures.

Well, Kong Mesh was the first mesh I had used that exposed the ablity to actually configure the Envoy Proxy sidecar - with the [Proxy Template](https://kuma.io/docs/1.7.x/policies/proxy-template/). In order to leverage this resource properly does require some familiarity with Envoy and how to configure it. Because of this I took a small detour to familiarize myself with the fundamentals of Envoy Proxy. Below is the list of my favorite resources that helped me grasp the fundamentels of Envoy.

[Service Mesh 102: Envoy Configuration](https://konghq.com/blog/envoy-service-mesh-configuration): this helps break down the fundmentals Envoy Proxy into its core components into a really easy to digest blog by Scott Lowe.

[Scott Lowe - Understanding the Basics of Envoy Configuration](https://www.youtube.com/watch?v=E-UpGmj6B9M): 40 min webinar on the basics of envoy configuration also done by Scott Lowe, does cover much of what is in Service Mesh 102 blog, but matches Kong Mesh Policies to actual Envoy Proxy Configurations.

[Course - Envoy Fundamentals](https://academy.tetrate.io/): this is a course that provides video and lab content. The course basically runs through Envoy Documenation but the author does a great job with the videos - concise, examples are pretty clear, sample configurations, the labs can actually run on your machine (assuming you have docker).

[Life of a Request - Envoy Proxy Docs](https://www.envoyproxy.io/docs/envoy/v1.19.0/intro/life_of_a_request): after going through all the above materials, I felt ready to digest this doc.
