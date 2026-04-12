# Gemma 4 DGX Spark Blog Post Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish the Gemma 4 DGX Spark benchmarks blog post with two Chart.js visualizations and styled admonition recommendation cards.

**Architecture:** A single reusable `BarChart.astro` component uses `chart.js/auto` to render bar charts. Chart data is serialized as JSON in a `data-barchart` attribute and initialized by a bundled script. Dark mode adapts via `MutationObserver` watching `data-theme` on `<html>`. The post is an MDX file importing the component. Recommendation cards use the existing `:::tip` admonition syntax — no new component needed for those.

**Tech Stack:** Astro 6, chart.js, MDX, Tailwind CSS v4, remark-admonitions (existing)

---

## File Map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `src/components/BarChart.astro` | Reusable bar chart with dark mode + projected bar support |
| Create | `src/content/post/gemma4-dgx-spark.mdx` | The blog post with chart imports and admonition cards |
| Modify | `package.json` (via pnpm) | Add `chart.js` dependency |

---

### Task 1: Install chart.js

**Files:**
- Modify: `package.json`

- [ ] **Step 1: Install the dependency**

```bash
cd /Users/vaishak/Documents/GitHub/blog.vaishak.me
pnpm add chart.js
```

Expected: `chart.js` appears in `dependencies` in `package.json`. No errors.

- [ ] **Step 2: Verify the install**

```bash
grep '"chart.js"' package.json
```

Expected output (version may differ):
```
"chart.js": "^4.x.x",
```

- [ ] **Step 3: Commit**

```bash
git add package.json pnpm-lock.yaml
git commit -m "feat: add chart.js for benchmark visualizations"
```

---

### Task 2: Create BarChart.astro

**Files:**
- Create: `src/components/BarChart.astro`

- [ ] **Step 1: Create the component**

Create `src/components/BarChart.astro` with this exact content:

```astro
---
interface Dataset {
	label: string;
	data: (number | null)[];
	color: string | string[];
}

interface Props {
	labels: (string | string[])[];
	datasets: Dataset[];
	projected?: number;
	yLabel?: string;
	height?: number;
}

const { labels, datasets, projected = null, yLabel = "tok/s", height = 300 } = Astro.props;

const id = `chart-${crypto.randomUUID()}`;
const configJson = JSON.stringify({ labels, datasets, projected, yLabel });
---

<div style={`position:relative;height:${height}px;width:100%`}>
	<canvas id={id} data-barchart={configJson}></canvas>
</div>

<script>
	import Chart from "chart.js/auto";
	import type { ChartDataset, Plugin } from "chart.js";

	interface DatasetConfig {
		label: string;
		data: (number | null)[];
		color: string | string[];
	}

	interface BarChartConfig {
		labels: (string | string[])[];
		datasets: DatasetConfig[];
		projected: number | null;
		yLabel: string;
	}

	function css(varName: string): string {
		return getComputedStyle(document.documentElement).getPropertyValue(varName).trim();
	}

	function resolveColor(color: string | string[], index: number): string {
		return Array.isArray(color) ? (color[index] ?? color[0]) : color;
	}

	function projectedPlugin(index: number, color: string): Plugin<"bar"> {
		return {
			id: "projected-bar",
			afterDatasetsDraw(chart) {
				const meta = chart.getDatasetMeta(0);
				const el = meta.data[index] as any;
				if (!el) return;
				const { x, y, width, base } = el;
				const ctx = chart.ctx;
				ctx.save();
				ctx.setLineDash([5, 4]);
				ctx.strokeStyle = color;
				ctx.lineWidth = 2;
				ctx.strokeRect(x - width / 2, y, width, base - y);
				ctx.setLineDash([]);
				ctx.font = "bold 12px sans-serif";
				ctx.fillStyle = color;
				ctx.textAlign = "center";
				ctx.textBaseline = "bottom";
				ctx.fillText("↑", x, y - 2);
				ctx.fillText("?", x, y - 16);
				ctx.restore();
			},
		};
	}

	function buildDatasets(configs: DatasetConfig[], projected: number | null): ChartDataset<"bar">[] {
		return configs.map((ds) => ({
			label: ds.label,
			data: ds.data,
			backgroundColor: ds.data.map((_, i) =>
				i === projected ? "transparent" : resolveColor(ds.color, i) + "cc",
			),
			borderColor: ds.data.map((_, i) =>
				i === projected ? "transparent" : resolveColor(ds.color, i),
			),
			borderWidth: 1,
			borderRadius: 3,
		}));
	}

	function initChart(canvas: HTMLCanvasElement) {
		const config = JSON.parse(canvas.dataset.barchart!) as BarChartConfig;
		const { labels, datasets, projected, yLabel } = config;

		const muted = () => css("--color-muted");
		const grid = () => css("--color-muted") + "33";

		const plugins: Plugin<"bar">[] = [];
		if (projected !== null) {
			plugins.push(projectedPlugin(projected, resolveColor(datasets[0]!.color, projected)));
		}

		const chart = new Chart(canvas, {
			type: "bar",
			data: { labels, datasets: buildDatasets(datasets, projected) },
			options: {
				responsive: true,
				maintainAspectRatio: false,
				plugins: {
					legend: {
						display: datasets.length > 1,
						labels: { color: muted(), boxWidth: 12, padding: 16 },
					},
					tooltip: {
						callbacks: {
							label: (ctx) => {
								const v = ctx.parsed.y;
								return `${ctx.dataset.label}: ${v === 0 ? "projected" : v + " " + yLabel}`;
							},
						},
					},
				},
				scales: {
					x: {
						ticks: { color: muted(), maxRotation: 40 },
						grid: { color: grid() },
					},
					y: {
						beginAtZero: true,
						title: { display: true, text: yLabel, color: muted() },
						ticks: { color: muted() },
						grid: { color: grid() },
					},
				},
			},
			plugins,
		});

		new MutationObserver(() => {
			const m = muted();
			const g = grid();
			chart.options.plugins!.legend!.labels!.color = m;
			(chart.options.scales!.x!.ticks as any).color = m;
			(chart.options.scales!.x!.grid as any).color = g;
			(chart.options.scales!.y!.title as any).color = m;
			(chart.options.scales!.y!.ticks as any).color = m;
			(chart.options.scales!.y!.grid as any).color = g;
			chart.update("none");
		}).observe(document.documentElement, { attributes: true, attributeFilter: ["data-theme"] });
	}

	document.querySelectorAll<HTMLCanvasElement>("canvas[data-barchart]").forEach(initChart);
</script>
```

- [ ] **Step 2: Run type check**

```bash
cd /Users/vaishak/Documents/GitHub/blog.vaishak.me
pnpm astro check
```

Expected: zero errors. Warnings about `any` in the plugin are acceptable.

- [ ] **Step 3: Commit**

```bash
git add src/components/BarChart.astro
git commit -m "feat: add reusable BarChart component with Chart.js and dark mode support"
```

---

### Task 3: Create the blog post MDX

**Files:**
- Create: `src/content/post/gemma4-dgx-spark.mdx`

- [ ] **Step 1: Create the MDX file**

Create `src/content/post/gemma4-dgx-spark.mdx` with this exact content:

````mdx
---
title: "Benchmarking Gemma 4 on DGX Spark"
description: "Every inference stack on GB10 hardware — llama.cpp, Ollama, vLLM — across quantizations and concurrencies. Model architecture accounts for ~8x of the throughput difference."
publishDate: "2026-04-12"
tags: ["ai", "inference", "benchmarks", "nvidia"]
---

import BarChart from "@/components/BarChart.astro";

The GB10 Grace Blackwell Superchip in NVIDIA's DGX Spark has one property that matters for LLM inference at this model size: 128GB of unified LPDDR5X memory shared between CPU and GPU, with the GPU accessing the same physical pool as the CPU. When Gemma 4 shipped last week, I ran it through every available inference stack to find out what that hardware advantage actually translates to in practice.

The main finding: model architecture accounts for roughly 8x of the throughput difference. Stack choice is secondary.

---

## The Hardware

The GB10 integrates a 72-core Arm CPU and a Blackwell GPU on a single die, sharing 128GB of unified LPDDR5X memory at ~273 GB/s bandwidth. CPU and GPU access the same physical memory pool, which eliminates the PCIe transfer overhead present in discrete GPU setups.

For LLM inference, the relevant number is memory capacity. The 30B+ parameter range sits in an awkward spot: too large for consumer GPU VRAM, too small to justify full datacenter hardware. At 128GB unified, running these models at full precision is straightforwardly feasible in principle. Gemma 4 31B at BF16 requires ~62GB. The 26B MoE model at FP16 sits around 55GB. Both fit with 66-73GB remaining for KV cache and system overhead.

---

## What I Tested

- **Models**: Gemma 4 31B (dense) and Gemma 4 26B (Mixture of Experts, ~4B active parameters per token)
- **Stacks**: llama.cpp, Ollama, vLLM
- **Quantizations**: FP16 / BF16, Q8_0, Q4_K_M, NVFP4
- **Tool**: genai-perf 0.0.16, 512 input tokens, 512 output tokens, 5 requests per run
- **Concurrency**: 1 (single user) and 4 (light multi-user load)

:::note[A note on methodology]
Five requests per run is a small sample. The p50 latency numbers in the tables below are directional: they reflect real measurements, but should not be treated as high-precision statistics. The throughput rankings and ratios are consistent enough across runs to be trustworthy; the exact millisecond values less so.
:::

Metrics that matter: **output tok/s** (generation speed), **TTFT p50** (time to first token), **ITL p50** (inter-token latency, the "streaming feel").

For reference: 10-20 tok/s is usable but noticeably slow. 20-40 tok/s feels like comfortable typing speed. 40-80 tok/s is smooth enough that you stop noticing the generation. Above 80 tok/s feels instantaneous.

---

## The Obvious Setup, and Why It Let Me Down

I started with vLLM + Gemma 4 31B at BF16. Makes sense on paper, right? vLLM is the production-standard inference stack. BF16 keeps full model quality. The hardware can hold it. What could possibly go wrong?

**3.51 tok/s at c1.**

That's near reading speed. Technically functional; completely impractical for interactive use.

Next up: NVFP4 quantization using NVIDIA's own `nvidia/Gemma-4-31B-IT-NVFP4` model, purpose-built for Blackwell tensor cores. This should be the throughput ceiling.

**6.34 tok/s.** Roughly on par with a mid-range consumer GPU running a 7B model.

Then Ollama on the same dense 31B model: 9.4 tok/s. Marginally usable for single-user chat. The 106ms inter-token latency makes streaming noticeably choppy.

Something was clearly broken with the NVFP4 result specifically. Worth digging into before I wrote off the whole path.

---

## The NVFP4 Dead End

The most plausible explanation I've found involves two overlapping software gaps. I haven't confirmed these by building custom containers from source, so treat this as the best available theory consistent with the error output and container inspection.

**First**: PyTorch in every available container wasn't compiled with sm_121-specific kernels. The GB10 GPU is sm_121 (CUDA capability 12.1, the Blackwell microarchitecture). Available containers max out at sm_120 or earlier. So the GPU falls back to Hopper or older codepaths for matrix multiply operations instead of running native Blackwell FP4 tensor core kernels.

**Second**: Torch.compile's Triton JIT autotuner (which picks optimized GPU kernels at runtime) fails to find suitable sm_121 kernels and just tells you about it directly:

```
Not enough SMs to use max_autotune_gemm mode
```

I tested every available image:

| Container | ARM64 | sm_121 PyTorch | Gemma 4 transformers | Verdict |
|-----------|-------|----------------|---------------------|---------|
| `vllm/vllm-openai:v0.8.5` | No | No | Yes | x86 only, unusable on GB10 |
| `gemma4-cu130` | Partial (x86 pull) | No | Yes | Falls back to compat path |
| `gemma4-0409-arm64-cu130` | Yes | No | Yes | Best available, still falls back |
| `cu130-nightly-aarch64` | Yes | No | No | sm_121 PyTorch but no Gemma 4 support |

No available image satisfies all three requirements at once. The NVFP4 weights load fine: 4-bit precision, roughly 20GB versus the 62GB BF16 model, and that memory reduction is real. But compute runs on a non-native fallback path, so the throughput benefit of Blackwell's FP4 tensor cores never shows up. What you get looks like BF16-level throughput with a smaller working set. Not the substantial uplift NVFP4 on native Blackwell hardware should deliver.

When NVIDIA eventually ships a container combining ARM64 + sm_121-compiled PyTorch + Gemma 4 transformers support, the NVFP4 number should improve substantially.

<BarChart
  labels={["Actual NVFP4", "Projected NVFP4", "MoE FP16"]}
  datasets={[{
    label: "Output tok/s",
    data: [6.34, 20, 22.49],
    color: ["#ef4444", "#f97316", "#22c55e"]
  }]}
  projected={1}
  height={220}
/>

---

## The Architecture Unlock

After exhausting the obvious options, I ran the same benchmarks against the other Gemma 4 configuration.

Gemma 4 ships in two architectures, and the difference between them is the whole story here.

**Gemma 4 31B** is a standard dense transformer. Every token activates all 31 billion parameters. The full weight matrix participates in every single forward pass, no exceptions.

**Gemma 4 26B** is a Mixture of Experts model. Despite 26 billion total parameters, only approximately **4 billion are active per token**. The inactive experts sit in memory (they do participate in prefill to some degree) but for decode, the dominant cost is that active expert set: roughly 4B parameters per step.

That distinction is the entire story.

I ran the 26B MoE through llama.cpp at FP16. No quantization, full precision, same stack I'd been using on the dense model. Results:

**22.49 tok/s. TTFT 561ms. ITL 43ms.**

Compare that to llama.cpp FP16 on the 31B dense model: **2.86 tok/s. TTFT 1,468ms. ITL 283ms.**

Same stack. Same precision. Same hardware. **~7.9x faster throughput** and **~6.6x better inter-token latency.**

The explanation is pretty direct. LLM decoding is memory-bandwidth-bound: the bottleneck is how fast you can read model weights from memory to compute each token. At the GB10's 273 GB/s unified memory bandwidth, reading 4B FP16 parameters (~8GB) takes about 29ms. Reading 31B FP16 parameters (~62GB) takes about 227ms. That 7.8x bandwidth ratio predicts the 7.9x throughput ratio almost exactly. The memory subsystem is the ceiling; the architecture determines how close you get to it.

So the "smaller" model produces better practical results with no quantization needed. Full FP16 quality on the MoE model, at throughput comparable to a heavily quantized dense model. The quality-vs-speed tradeoff more or less disappears. One honest caveat: I didn't run formal quality benchmarks comparing the two architectures. The 26B MoE and 31B dense are distinct models, not size variants of the same thing. Their output on complex reasoning tasks or long-context work may differ. The "no tradeoff" claim applies specifically to the throughput-vs-quantization question within the MoE model. You can run it at full precision and still be fast.

---

## Full Results

<BarChart
  labels={[
    ["31B", "FP16"],
    ["31B", "Q8"],
    ["31B", "Q4_K_M"],
    ["31B BF16", "(vLLM)"],
    ["31B NVFP4", "(vLLM)"],
    ["MoE", "FP16"],
    ["MoE", "Q8"],
    ["MoE", "Q4_K_M"],
    ["MoE BF16", "(vLLM)"],
  ]}
  datasets={[
    {
      label: "llama.cpp",
      data: [2.86, 5.07, 7.77, null, null, 22.49, 38.62, 57.62, null],
      color: "#f97316",
    },
    {
      label: "vLLM",
      data: [null, null, null, 3.51, 6.34, null, null, null, 21.53],
      color: "#3b82f6",
    },
  ]}
  height={340}
/>

| Model | Stack | Quant | c | Output tok/s | TTFT p50 (ms) | ITL p50 (ms) |
|-------|-------|-------|---|-------------|---------------|--------------|
| Gemma 4 31B | llama.cpp | FP16 | 1 | 2.86 | 1,468 | 283 |
| Gemma 4 31B | llama.cpp | Q8_0 | 1 | 5.07 | 1,668 | 166 |
| Gemma 4 31B | llama.cpp | Q8_0 | 4 | 13.77 | 2,934 | 185 |
| Gemma 4 31B | llama.cpp | Q4_K_M | 1 | 7.77 | 1,271 | 104 |
| Gemma 4 31B | llama.cpp | Q4_K_M | 4 | 17.44 | 2,502 | 120 |
| Gemma 4 31B | Ollama | Q4_K_M | 1 | 9.4 | 106 | 106 |
| Gemma 4 31B | vLLM | BF16 | 1 | 3.51 | 596 | 284 |
| Gemma 4 31B | vLLM | BF16 | 4 | 8.66 | 428 | 294 |
| Gemma 4 31B | vLLM | NVFP4 | 1 | 6.34 | 359 | 157 |
| Gemma 4 31B | vLLM | NVFP4 | 4 | 15.84 | 357 | 159 |
| **Gemma 4 26B MoE** | **llama.cpp** | **FP16** | **1** | **22.49** | **561** | **43** |
| Gemma 4 26B MoE | llama.cpp | FP16 | 4 | 41.01 | 978 | 73 |
| Gemma 4 26B MoE | llama.cpp | Q8_0 | 1 | 38.62 | 387 | 25 |
| Gemma 4 26B MoE | llama.cpp | Q8_0 | 4 | 63.33 | 930 | 48 |
| **Gemma 4 26B MoE** | **llama.cpp** | **Q4_K_M** | **1** | **57.62** | **332** | **17** |
| **Gemma 4 26B MoE** | **llama.cpp** | **Q4_K_M** | **4** | **91.80** | **914** | **33** |
| Gemma 4 26B MoE | Ollama | Q4_K_M | 1 | 59.4 | 20 * | 17 |
| Gemma 4 26B MoE | vLLM | BF16 | 1 | 21.53 | 259 | 45 |
| **Gemma 4 26B MoE** | **vLLM** | **BF16** | **4** | **62.81** | **281** | **63** |

*\* Ollama results via native `/api/chat`, not genai-perf. TTFT excludes cold-start first request (model pre-loaded with keep_alive=-1). The 20ms TTFT under Ollama vs 332ms under llama.cpp at the same Q4_K_M quantization is a 16x gap I have not fully explained. Ollama's model-keep-alive behavior likely eliminates loading overhead that llama.cpp may include in its TTFT measurement, but I have not isolated this variable. Treat the Ollama TTFT as a best-case, warm-model figure.*

---

## What to Actually Run

:::tip[Single user, best quality]
**26B MoE FP16 + llama.cpp** — 22 tok/s, 43ms ITL, no quantization tradeoffs
:::

:::tip[Single user, max throughput]
**26B MoE Q4_K_M + llama.cpp** — 57 tok/s, 17ms ITL
:::

:::tip[Light multi-user]
**26B MoE Q4_K_M + llama.cpp** — 91 tok/s at c4, 33ms ITL
:::

:::tip[Multi-user, zero quantization]
**26B MoE BF16 + vLLM** — 62 tok/s, TTFT stays flat under load
:::

A few things worth knowing about the stack tradeoffs:

**llama.cpp vs vLLM**: vLLM consistently wins on TTFT: 2 to 2.5x faster prefill, thanks to fused attention and PagedAttention. At higher concurrency, vLLM's continuous batching starts pulling ahead too; the MoE BF16 c4 result (62 tok/s) actually beats llama.cpp FP16 c4 (41 tok/s). But for single-user raw decode throughput, llama.cpp Q4_K_M wins by a pretty wide margin. The practical split, in my view: vLLM for multi-user APIs or anything latency-sensitive; llama.cpp for single-user throughput with simpler setup.

**Ollama**: Wraps llama.cpp internally, and the throughput numbers track closely (59.4 vs 57.62 tok/s for MoE Q4_K_M at c1). Easiest stack to get running, solid performance for personal use. The TTFT anomaly noted above is worth keeping in mind if you're comparing raw latency numbers across stacks. It's probably not showing you what you think it is.

**Quantization on the MoE model**: Going from FP16 to Q4_K_M on the 26B MoE nets roughly 2.5x throughput (22 to 57 tok/s). At this model size, the practical quality difference between FP16 and Q4_K_M is small. I haven't formally measured it, but anecdotally it's not alarming. FP16 at 22 tok/s is genuinely comfortable for single-user chat. Q4_K_M at 57-91 tok/s gives you real headroom for multi-user or agentic workloads.

---

## What Happens When NVIDIA Closes the Software Gap

**NVFP4 at 6.34 tok/s isn't the final answer.** When NVIDIA ships a container combining ARM64 + sm_121-compiled PyTorch + Gemma 4 transformers support, that number should improve substantially. The compute path currently being bypassed is exactly what Blackwell's FP4 tensor cores are designed for. That changes the stack conversation: NVIDIA's official quantized model running on native hardware, potentially competitive with current llama.cpp numbers through a supported path.

The MoE architecture advantage is independent of that. A model activating 4B parameters per token decodes faster than one activating 31B at the same memory bandwidth, regardless of what the dense NVFP4 path eventually delivers. The bandwidth math doesn't change.

For now: grab the 26B MoE GGUF, serve it through llama.cpp at FP16 or Q4_K_M, and revisit vLLM + NVFP4 when NVIDIA ships that container update.
````

- [ ] **Step 2: Run type check**

```bash
cd /Users/vaishak/Documents/GitHub/blog.vaishak.me
pnpm astro check
```

Expected: zero errors.

- [ ] **Step 3: Start dev server and inspect visually**

```bash
pnpm dev
```

Open `http://localhost:4321` in a browser and navigate to the new post. Verify:
- NVFP4 comparison chart renders with 3 bars — Actual NVFP4 (solid red), Projected NVFP4 (dashed orange outline with ↑ and ?), MoE FP16 (solid green)
- Full results chart renders with 9 x-axis groups, two colored datasets (orange = llama.cpp, blue = vLLM), MoE bars visually taller on the right
- Four `:::tip` admonition blocks appear with green left border and light green background
- Toggle dark mode — chart axis labels, grid lines, and legend text adapt; bar fill colors stay consistent

- [ ] **Step 4: Commit**

```bash
git add src/content/post/gemma4-dgx-spark.mdx
git commit -m "feat: publish Gemma 4 DGX Spark benchmarks post"
```

---

### Task 4: Verify production build

**Files:** (no changes, verification only)

- [ ] **Step 1: Run full build**

```bash
cd /Users/vaishak/Documents/GitHub/blog.vaishak.me
pnpm build
```

Expected: build completes without errors. The post appears in the generated `dist/` output. Pagefind indexes it (postbuild step runs automatically).

- [ ] **Step 2: Preview the production build**

```bash
pnpm preview
```

Open `http://localhost:4321` and verify the post renders identically to dev. Charts should appear (Chart.js is bundled into the JS output by Vite).

- [ ] **Step 3: Commit docs**

```bash
git add docs/
git commit -m "docs: add design spec and implementation plan for Gemma 4 post"
```
