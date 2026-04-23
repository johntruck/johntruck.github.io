---
title: Choosing patch type
date: 2026-04-08
categories: [Patch (Kernel), Patch Development]
tags: [patch, kernel]     # TAG names should always be lowercase
---

# Choosing the type of patch:

In today's class me and my work partner picked what type of patch we are going to make. Our monitor, David, presented us with a variety of possible patches, that can be seen [here](https://flusp.ime.usp.br/courses/linux-kernel-patch-suggestions/). We decided to go with 2.1: Replace manual bitfield manipulations with macros. Basically, the patch is about replacing these very common bit manipulations with macros, so that the code becomes (1) more readable, (2) more manageable and (3) safer (there are some safety measures on the macros).