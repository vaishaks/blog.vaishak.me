# Design: Gemma 4 DGX Spark Benchmarks Blog Post

**Date:** 2026-04-12  
**Status:** Approved

---

## Overview

Publish a new blog post benchmarking Gemma 4 inference stacks on NVIDIA DGX Spark (GB10 Grace Blackwell). The post content is written; this design covers the visualizations and formatting enhancements needed to make it publication-ready.

---

## New Files

| File | Purpose |
|------|---------|
| `src/content/post/gemma4-dgx-spark.mdx` | The blog post (MDX for component support) |
| `src/components/BarChart.astro` | Reusable Chart.js bar chart component |

---

## New Dependencies

| Package | Reason |
|---------|--------|
| `chart.js` | Client-side bar charts with dark mode support |

---

## `BarChart.astro` Component

### Props

```typescript
{
  labels: string[]              // x-axis labels
  datasets: {
    label: string
    data: number[]
    color: string
  }[]                           // one entry per series/group
  projected?: number            // index of bar to render as outline+arrow (speculative value)
  yLabel?: string               // y-axis label, defaults to "tok/s"
  height?: number               // canvas height in px, defaults to 300
}
```

### Behavior

- Renders a `<canvas>` element with an inline `<script>` (no Astro `client:*` directive needed)
- Resolves theme colors at init via `getComputedStyle(document.documentElement)` reading CSS custom properties
- Attaches a `MutationObserver` on `<html>[data-theme]` to re-render chart when theme changes
- If `projected` prop is set, registers a custom Chart.js plugin that draws that bar as a dashed outline with an upward arrow and "?" label instead of a solid fill

### Colors

| Use | CSS variable |
|-----|-------------|
| Dense model bars | `--color-accent` |
| MoE model bars | `--color-link` (blue-toned per theme) |
| Grid lines / tick labels | `--color-muted` |
| Projected bar stroke | `--color-accent` at 0% fill, dashed border |

---

## Visualizations in the Post

### 1. NVFP4 Comparison Chart

**Location:** After the "The NVFP4 Dead End" section, before the `---` separator.

**Purpose:** Show where actual NVFP4 performance sits relative to the projected future ceiling and MoE FP16.

**Chart config:**
- Single dataset (no grouping)
- 3 bars: `Actual NVFP4` (6.34 tok/s), `Projected NVFP4` (outline+arrow, `projected=1`), `MoE FP16` (22.49 tok/s)
- Colors: red for actual NVFP4, accent-dashed for projected, green for MoE FP16

### 2. Full Results Chart

**Location:** In the "Full Results" section, above the data table.

**Purpose:** Visual summary of all benchmark results — the MoE cluster should visually dominate the right half.

**Chart config:**
- Grouped bar chart
- X-axis: model + quantization combinations (e.g. "31B FP16", "31B Q8", "31B Q4", "31B BF16 vLLM", "26B MoE FP16", "26B MoE Q8", "26B MoE Q4", "26B MoE BF16 vLLM")
- Two datasets: llama.cpp (c1 only for clarity) and vLLM (c1 only)
- Ollama omitted from chart (different measurement methodology — remains in table)
- Y-axis: output tok/s
- The data table below remains as the authoritative reference

### 3. Recommendation Cards

**Location:** Replaces the markdown table in "What to Actually Run".

**Implementation:** Four `:::tip` admonition blocks (existing remark-admonitions system, no new component).

**Content:**

```
:::tip[Single user, best quality]
26B MoE FP16 + llama.cpp — 22 tok/s, 43ms ITL, no quantization tradeoffs
:::

:::tip[Single user, max throughput]
26B MoE Q4_K_M + llama.cpp — 57 tok/s, 17ms ITL
:::

:::tip[Light multi-user]
26B MoE Q4_K_M + llama.cpp — 91 tok/s at c4, 33ms ITL
:::

:::tip[Multi-user, zero quantization]
26B MoE BF16 + vLLM — 62 tok/s, TTFT stays flat under load
:::
```

---

## Post Frontmatter

```yaml
title: "Benchmarking Gemma 4 on DGX Spark"
description: "Every available inference stack on GB10 hardware — llama.cpp, Ollama, vLLM — across quantizations and concurrencies. The main finding: model architecture accounts for ~8x of the throughput difference."
publishDate: "2026-04-12"
tags: ["ai", "inference", "benchmarks", "nvidia"]
```

---

## What Is Not Changing

- `BlogPost.astro` layout — no changes
- Existing remark plugins — no changes
- No new CSS files — chart styling is inline in `BarChart.astro`
- The full data table in "Full Results" stays as the source of truth
- The `:::important` / `:::note` container syntax for the methodology callout stays as a native admonition

---

## Constraints

- Chart.js renders client-side; charts will not appear in the OG image (Satori-generated, server-side)
- Dark mode reacts to `data-theme` attribute changes on `<html>` — matches the blog's existing ThemeProvider behavior
- `title` field in frontmatter has a 60-character max (content.config.ts schema) — the proposed title fits at 37 characters
