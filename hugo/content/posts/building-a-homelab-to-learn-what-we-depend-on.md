---
title: Building a Homelab to Learn What We Depend On
date: '2026-11-13T00:00:00+00:00'
description: Why I’m building a homelab to learn self-hosting, practice recovery,
  and test the applications I’m building.
categories:
- Engineering
- Infrastructure
tags:
- homelab
- self-hosting
- infrastructure as code
- terraform
- ansible
slug: building-a-homelab-to-learn-what-we-depend-on
---

When an online service I rely on goes down, I’m reminded how much of our work and daily life depends on systems we don’t control. Cloud services are useful, but outages disrupt real work. That has me thinking about what it takes to host software ourselves.

That’s one reason I’m building a homelab.

I want to learn how to bring systems up, keep them running, and recover when something breaks. I’m using Terraform to describe and provision infrastructure, and Ansible to configure systems and services. Together, they give me a repeatable process and help me understand each step.

The lab will also give me a place to develop and test the applications I’m building. I can use equipment I already own instead of immediately paying for cloud development, testing, or staging environments. If an application grows to where cloud hosting makes sense, I can make that choice based on its needs and costs.

I think more businesses should consider this before moving every environment straight to AWS or another cloud provider. Development and testing systems don’t always need to live in the cloud from day one. An internal environment can be a practical place to build and learn without taking on a cloud bill before the business knows what it needs. The right choice depends on the team, application, security needs, and operating costs.

I’m also using AI to help me learn. It can explain unfamiliar concepts, suggest where to look when something fails, and help me reason through possible causes. But I still need to understand what I’m changing and why. The goal isn’t to have AI build the lab for me; it’s to use it as a guide while I build the skills to operate the systems myself.

This matters beyond a personal lab. Businesses rely on internal software for everyday work, from managing inventory and documents to running projects. Knowing how to host and maintain those systems gives a business more choices about its data, operations, and response to provider outages.

Self-hosting isn’t automatically cheaper or more reliable. It brings responsibility for security, backups, updates, and recovery. A homelab can help build the knowledge to make those decisions with open eyes.

For me, this is about learning through practical work. Terraform and Ansible make the setup repeatable. AI helps me investigate and understand. And the homelab gives me a place to develop, test, and practice before others depend on the software I’m building.

The more we rely on hosted services, the more valuable it is to understand how they work—and when running our own systems makes sense.
