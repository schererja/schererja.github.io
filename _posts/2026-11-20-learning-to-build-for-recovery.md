---
layout: post
title: "Learning to Build for Recovery"
date: 2026-11-20
description: "What testing VM recovery with Terraform, OpenTofu, and Ansible has taught me about operating infrastructure."
categories: ["Engineering", "Infrastructure"]
tags: ["infrastructure as code", "terraform", "opentofu", "ansible", "recovery", "hybrid infrastructure"]
---

At work, I’ve been using Terraform/OpenTofu and Ansible, and it’s been a great way to learn how infrastructure is built and managed.

AI has made troubleshooting much easier. When I hit an issue I might not have known how to investigate, I can use it to explore possible causes and find useful next steps. But I’m still hands-on: I review what changes, double-check the results, and make sure the solution works in the environment. Getting an answer is only part of the job. Understanding and verifying it matters too.

One of the most satisfying parts has been testing recovery. Seeing a system go down and then bringing up a replacement VM is a great feeling. It turns infrastructure-as-code from an idea into something practical: a way to rebuild a system and see whether the process actually works. The more I test, the more I learn about what can recover smoothly and what still needs attention.

That connects to a knowledge gap I’ve wanted to close for a long time. I’ve wanted to understand how companies provision systems, configure servers, and run the software their teams depend on. Working with these tools is giving me a way to learn by doing.

I also think the choice between cloud and self-hosted infrastructure is too often treated as all or nothing. Cloud services can be a great fit, and internal infrastructure can be a great fit. A hybrid approach gives organizations more options: choose where each workload belongs, and make sure there’s a workable recovery plan when something fails.

The 3-2-1 backup rule is usually about protecting data: keep multiple copies, on different media, with one off-site. Maybe infrastructure planning needs some of that same mindset. A backup plan and a recovery plan solve different problems, but both help avoid depending on one copy, one location, or one path back to operation.

I can imagine more companies looking closely at where they run their systems as outages and costs affect their decisions. I don’t expect every organization to leave the cloud. I do think it’s valuable to understand how to operate systems in more than one environment.

AI helps me learn faster and troubleshoot issues that might once have stopped me. Terraform/OpenTofu and Ansible give me tools to build and rebuild. And testing the recovery process helps me understand whether the system is ready for the next failure.
