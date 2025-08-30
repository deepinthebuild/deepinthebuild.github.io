---
layout: post
title: "Affected Target Analysis with Bazel"
description: "Cute tricks with bazel-diff"
date: 2025-08-30
tags: [bazel, build]
---

# Changed target analysis

Code snippet:
```bash
bazel query 'set('"${RELEVANT_TARGETS}"')'
```