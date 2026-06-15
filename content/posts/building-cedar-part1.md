---
title: "Building Cedar, part 1"
date: 2026-06-10T00:00:00Z
---

A systems engineer begins with a problem: can infrastructure feel as honest and simple as a carefully written terminal session?

This post explores the first steps of designing a toolchain with composable pipelines, immutable state, and small, deliberate abstractions.

## Notes

- Keep the surface area small.
- Prefer clarity over cleverness.
- Treat every configuration file like source code.

```sh
mkdir cedar
cd cedar
git init
```
