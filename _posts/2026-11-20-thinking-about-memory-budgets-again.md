---
layout: post
title: "Thinking About Memory Budgets Again"
date: 2026-11-20 08:00:00 -0800
description: "Seeing how prominently AI featured at CES made me think about what software should ask of a machine."
categories: ["Engineering"]
tags: ["performance","memory","hardware"]
---

Seeing how prominently AI featured in coverage of CES made me think about the cost of the machines people use. I wasn’t at the show. What I kept wondering from the coverage was how demand for memory might affect the hardware a person can buy, and what that means for the software we build.

If I want a tool to run on a small machine, it helps to decide early what that means and try it on something close to the target. How much memory does it use at idle? What happens when it’s doing the main thing I built it for? Can it do that comfortably with the other services that share the machine?

I’m curious how much interest this will bring back to efficient runtimes and frameworks. A practical starting point is to measure the program and figure out where its memory and CPU time are going. That gives me a better basis for deciding what to optimize or whether a change in tools would help.

*Adapted from my writing in the related posts on this LinkedIn thread.* [Read the LinkedIn thread](https://www.linkedin.com/posts/jasonrscherer_lookingforwork-ai-lasvegas-activity-7409283317516165121-Bzf2)
