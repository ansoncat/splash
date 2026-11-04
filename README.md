# Splash

[![CI](https://github.com/incoai/splash/actions/workflows/ci.yml/badge.svg)](https://github.com/incoai/splash/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Apple%20silicon-black.svg)](#quick-start)

**A local inference engine for Apple silicon, built around the model.**

> **This is a fork of [incoai/splash](https://github.com/incoai/splash).** It adds
> NVIDIA Nemotron 3.5 Bolt (74B-A9B) and Nemotron 3.5 Super VL (120B-A12B), with
> NVIDIA's LiLiCorr drafts, RADIO vision, and SSD expert streaming for both
> ([Nemotron 3.5](#nemotron-35-this-fork)). Everything else is upstream.
>
> - `nemotron` (this default branch): all of the fork's changes, rebased on upstream.
> - `main`: tracks upstream unchanged.
> - Other branches each stack on the one before, for changes that may be
>   proposed upstream:
>   - `nemotron-3.5-bolt`: Bolt with its LiLiCorr draft and RADIO vision, as
>     proposed in [incoai/splash#633](https://github.com/incoai/splash/issues/633).
>   - `bolt-streaming`: streams Nemotron-H's routed experts from the SSD.
>   - `bench-tight-memory`: `backend-benchmark` warm-up under a tight `--max-memory`.
>   - `super-vl`: Nemotron 3.5 Super VL, and NVFP4G64 for its official NVFP4 GGUF.
>     `nemotron` is this branch plus this README.
>
> The desktop app and Homebrew formula below are upstream's and don't include
> these changes. Build this branch from source instead
> ([Build and run](docs/server.md#build-and-run)).

Splash runs coding agents and OpenAI or Anthropic compatible applications on
one Mac. It combines [DFlash 2](https://inco.ai/blog/dflash2/) speculative
decoding, specialized Metal kernels, and automatic memory planning, with
vision, tool calling, and a built-in chat page.
It reuses cached prefixes and batches concurrent requests automatically.

## Quick start

[Download the Splash desktop app](https://github.com/incoai/splash/releases/latest/download/Splash.dmg)
for an Apple M3 or newer on macOS 26.4 or later, open the disk image and drag
Splash into Applications.

The command line needs the same Mac and [Homebrew](https://brew.sh):

```bash
brew install incoai/tap/splash
splash serve --model unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M
```

The first run downloads the model and its matching draft and starts serving
on `127.0.0.1:8000`. Later starts reuse the downloads; leave room on disk for
them ([storage requirements](docs/server.md#model-storage)).

Once it prints `Ready`, leave this terminal open. Open <http://127.0.0.1:8000>
in your browser, or run an installed coding agent from another terminal:

```bash
splash opencode    # or: splash claude / splash codex / splash hermes / splash pi
```

For a server started with another `--port`, pass the same `--port` to the agent.

Press Ctrl+C in the server terminal to stop Splash.
For LM Studio Bionic, follow its [Splash setup guide](https://lmstudio.ai/blog/splash-engine).

## Use the API

OpenAI Chat Completions, Responses and Completions, and Anthropic Messages,
with streaming, tool calls, JSON Schema output, and base64 images and PDFs:

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M",
    "messages": [{"role": "user", "content": "Explain speculative decoding in one sentence."}]
  }'
```

Reasoning follows the model default; `"reasoning_effort": "none"` turns it off
in Chat. Anthropic Messages requests reason only when they set `thinking`.
[Reasoning settings](docs/server.md#default-reasoning-effort) ·
[Tool calls](docs/api.md#tool-calls) ·
[API details](docs/api.md)

## Models

Splash supports these model families, with a matching DFlash2 draft selected
automatically:

| Model | GGUF example | MLX 4-bit |
| --- | --- | --- |
| Qwen3.8-27B | `unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_M` | `mlx-community/Qwen3.8-27B-4bit` |
| Qwen3.6-35B-A3B | `unsloth/Qwen3.6-35B-A3B-GGUF:UD-Q4_K_M` | `mlx-community/Qwen3.6-35B-A3B-4bit` |
| Ornith-1.5-122B | `unsloth/Ornith-1.5-122B-GGUF:UD-Q4_K_XL` | `ornith-ai/Ornith-1.5-122B-MLX-4bit` |
| Qwen3.8-Flash-Next | `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF:Q2_0` | — |
| Flint | `unsloth/flint-GGUF:UD-Q4_K_XL` | — |

Unsloth GGUF variants span **1–8 bits**, including mixed-precision UD formats
up to `UD-Q8_K_XL`, whose BF16 tensors load too. Whole-BF16 and F16 models are
not supported: the installer refuses a GGUF whose projections and experts are
all BF16, F16 or F32.
[Prism ML Ternary Bonsai 2](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf)
is also supported in PQ2_0 (7.2 GB), including vision. Pass `OWNER/REPO:VARIANT`
to `--model`, as in the quick start. MLX targets can be affine 2–8-bit,
including mixed precision, mxfp4 or nvfp4. NVIDIA's and unsloth's NVFP4
checkpoints and NVFP4 GGUFs also load. Smaller variants run on
[24 GB Macs](docs/performance.md#smaller-ggufs-on-24-gb-macs).
[27B variants](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF/tree/main) ·
[35B variants](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF/tree/main) ·
[Ornith-1.5-122B variants](https://huggingface.co/unsloth/Ornith-1.5-122B-GGUF/tree/main) ·
[Flash-Next variants](https://huggingface.co/unsloth/Qwen3.8-Flash-Next-GGUF/tree/main)

Vision and the tokenizer come from the target model's source.
[Model loading and compatibility](docs/models.md#upstream-model-loading) ·
[Supported formats](docs/models.md#gguf-targets)

To serve files already on this Mac, pass the target's GGUF file or GGUF/MLX
directory directly, and a compatible local DFlash2 draft:

```bash
splash serve --model local/qwen \
  --model-dir /models/Qwen3.8-27B-GGUF/Qwen3.8-27B-UD-Q4_K_M.gguf \
  --draft-model /models/Qwen3.8-27B-DFlash2
```

This prepares local files without contacting the Hub; `--model` is the name
reported by the API. Omit `--draft-model` to use the family's draft from the
Hub cache, or add `--mmproj /models/mmproj-BF16.gguf` for a GGUF's vision.
See [local model files](docs/models.md#local-model-files) for details.

### Nemotron 3.5 (this fork)

Bolt's draft is NVIDIA's `NVIDIA-Nemotron-3.5-Bolt-74B-A9B-24GB-LiLiCorr-GGUF`,
picked automatically. NVIDIA ships it only as a GGUF, so the installer converts
it once into a BF16 DFlash2 checkpoint.

```bash
./splash serve --model nvidia/NVIDIA-Nemotron-3.5-Bolt-74B-A9B-24GB-GGUF
```

Super has no automatic draft. Download the target, the LiLiCorr draft from
`ggml-org/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-GGUF` and an mmproj, install
them from local files, and serve the root it prints:

```bash
.venv/bin/python -m install.local /models/Super/UD-IQ2_XXS/<first shard>.gguf \
  --draft-model /models/Super/ggml-org/LiLiCorr --mmproj /models/Super/mmproj-BF16.gguf \
  --models /models/splash
./splash serve --model local/Super --model-dir <printed root>
```

- When the experts don't fit, `--expert-memory auto` streams the routed experts
  from the SSD, as for the other MoE families.
- Super's official NVFP4 GGUF loads too: its routed down projection has
  2,688-wide rows, which load as NVFP4G64
  ([formats](docs/models.md#gguf-targets)).

Limits outside this fork:

- Use ggml-org's Super LiLiCorr, not unsloth's: unsloth's doesn't match the
  official config (2 KV heads, no window, YaRN).
- Expert streaming was measured on a 64 GB Mac under `--max-memory`, not yet on
  a 32 GB one.

Not done yet:

- The Mamba-2 state is fp32 only; `--state-format bf16` is refused. A BF16 state
  needs its own kernels and a check of its error over long conversations.
- Vision takes still images only. The models' audio tower and video patch
  embedding aren't ported.
- The kernels cover these two models' shapes only, as for every family; other
  Nemotron-H geometries are refused until they get their own variants.
- The Super draft runs with three approximations: its window is capped at the
  engine's 2,048-token draft context (Super's is 8,192), ε is the kernels' 1e-6
  (Super's is 1e-5), and the target's output head stands in for the draft's
  own. They lower acceptance only, the output is the target's, and acceptance
  already matches Bolt's.

## Settings

Memory and context are sized automatically, up to the model's native context
window. To set your own limits or cache options, add these to `splash serve`:

| Option | Purpose |
| --- | --- |
| `--max-memory 28G` | Cap Metal memory use. |
| `--idle-release off` | Keep the model in memory while idle (default: release after 10m). |
| `--max-context 100K` | Set the context limit. Up to `1M` tokens. |
| `--language-only` | Skip vision; serve text only. |
| `--kv-format bf16` | Use BF16 KV cache. Default: 8-bit (INT8). |
| `--state-format bf16` | Use BF16 recurrent states, half the size. Default: FP32. |
| `--disable-ane` | Prefill on the GPU alone. Default: the 27B also uses the Neural Engine. |
| `--max-cache-disk 16G` | Cap the SSD cache (default: auto, up to 32 GiB; `0` turns it off). |
| `--persistent-cache` | Keep the SSD cache across restarts. Off by default. |

Use `--max-memory` to leave room for other applications.
The server listens on localhost without authentication by default. For LAN
access, authentication, browser apps on other origins, and other options, see
[server configuration](docs/server.md#server-configuration) or
`splash serve --help`.
[KV precision](docs/server.md#kv-cache-precision) ·
[SSD cache](docs/server.md#ssd-cache) ·
[Advanced memory settings](docs/server.md#memory-and-context)

## Performance

Measured with Splash 1.0 on an M5 Pro (16-core GPU, 48 GB), using its model
packages and selected SPEED-Bench coding prompts over HTTP. Ratios compare
with the next-fastest engine measured in that benchmark. Qwen3.8-Flash-Next
(`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF:Q2_0`) was measured the same way
with this release on an M5 Pro (20-core GPU, 64 GB); its ratios compare with
Strata serving the same GGUF on that Mac.

| Metric | Qwen3.6-35B-A3B | Qwen3.8-27B | Qwen3.8-Flash-Next |
| --- | ---: | ---: | ---: |
| Decode · short prompt | 210 tok/s (1.7×) | 74 tok/s (2.0×) | 95 tok/s (1.3×) |
| Prefill · 32K prompt | 2,011 tok/s (1.3×) | 363 tok/s (1.2×) | 1,312 tok/s (1.3×) |
| Cached time to first token · 32K replay | 123 ms (6.6×) | 282 ms (7.3×) | 171 ms (212×) |
| Aggregate decode · 4 concurrent short prompts | 357 tok/s (2.0×) | 170 tok/s (3.9×) | 171 tok/s (2.4×) |

[Launch benchmarks](https://inco.ai/blog/splash/) ·
[Measurement details](docs/performance.md#splash-10-launch-benchmarks) ·
[Run benchmarks locally](docs/validate.md#local-benchmarks)

### GGUF against llama.cpp

Same Unsloth UD-Q4_K_M weights on Metal. Decode speed in tok/s:

| Model | Engine | M5 Pro, 20-core GPU | M3 Max, 40-core GPU |
| --- | --- | ---: | ---: |
| 27B | llama.cpp | 16 | 17 |
| | llama.cpp with MTP | 27 | 20 |
| | **Splash** | **74** | **92** |
| 35B-A3B | llama.cpp | 69 | 66 |
| | **Splash** | **175** | **209** |

That is **2.5–3.2×** as fast on the 35B and **4.5–5.3×** on the 27B
(**2.7–4.6×** against MTP).

**Closely matches llama.cpp's predictions.**

| Next-token agreement ↑ | 27B | 35B-A3B |
| --- | ---: | ---: |
| llama.cpp: CPU vs. GPU | 97.8% | 96.5–96.9% |
| llama.cpp: single-token vs. batched | 99.65–99.75% | 97.95% |
| **Splash vs. llama.cpp** | **99.30–99.45%** | **97.83–98.14%** |

Splash uses BF16 KV and `--disable-ane` in this comparison.
[Benchmark details](docs/performance.md#gguf-against-llamacpp)

### Nemotron 3.5 on this fork

Measured by the fork's maintainer on an M5 Pro (20-core GPU, 64 GB): greedy,
thinking off, prefix cache off, runs of each engine interleaved in one session.
These are not upstream's numbers.

Bolt against llama.cpp (b11681 with its DFlash draft), decode speed in tok/s:

| Prompt | llama.cpp | **Splash** | Ratio |
| --- | ---: | ---: | ---: |
| Code | 45–46 | **115–120** | 2.5–2.6× |
| English prose | 34–40 | **62–68** | 1.6–2.0× |
| Traditional Chinese prose | 30–32 | **49–55** | 1.6–1.9× |
| 33K-token document, explained | 30 | **61** | 2.1× |

Time to first token is 1.6× as fast at 2K, 5K and 33K tokens. llama.cpp runs a
copy of the official GGUF with the Mamba tensors in Q8_0, since its Metal
backend doesn't run them in NVFP4; Splash runs the official GGUF.

- **Bolt with streamed experts**, `--max-memory 20G`: about 0.7× the resident
  decode speed, with logits bit-identical to resident. This was simulated on the
  64 GB Mac and hasn't been run on a 32 GB one.
- **Super VL**, UD-IQ2_XXS: all experts resident on 64 GB, 147K-token automatic
  context, decode 0.54–0.75× Bolt's.

| Model | GGUF | Experts on 64 GB | Decode |
| --- | --- | --- | --- |
| Bolt | NVIDIA NVFP4 (24 GB) | resident | baseline |
| Bolt | unsloth UD-IQ2_XXS | resident | 0.94–1.19× |
| Super VL | unsloth UD-IQ2_XXS (47.6 GiB) | resident | baseline |
| Super VL | unsloth UD-IQ3_XXS (55.7 GiB) | 78% resident, streamed | 0.56–0.70× |
| Super VL | ggml-org NVFP4 (71.6 GiB) | 47% resident, streamed | 0.16–0.29× |

On HumanEval+, BFCL and a Traditional Chinese check, the measured quantizations
of each model showed no clear quality difference (Super's NVFP4 wasn't
measured). The maintainer runs Bolt's NVFP4 and Super's UD-IQ2_XXS.

## Design

Each supported model pairs a trained DFlash2 draft with Metal kernels for its
shapes. The runtime, scheduler, cache, and API are shared. Weights are
converted to the kernels' layouts as they load, with no copy on disk; kernels
ship precompiled, with no Xcode or local tuning required.
[How Splash works](https://inco.ai/blog/splash/)

## More

- [Documentation](docs/README.md): configuration, API, internals, building, and testing.
- [Issues and feedback](https://github.com/incoai/splash/issues)
- [Apache-2.0](LICENSE). GGUF kernels include MIT-licensed material from
  llama.cpp; see [third-party notices](THIRD_PARTY_NOTICES). Model weights keep their own licenses.
