# Appendix B: GPU / VRAM Sizing Cheat Sheet

The number one question in local LLM work is: "Will this model fit on my GPU?" VRAM (the memory on your GPU) is the limit. This appendix gives you quick tables to answer that, plus GPU picks and rough cloud costs.

## B.1 The Simple Rule of Thumb

A model's weights take up memory based on how many bytes each parameter uses:

| Precision | Bytes per parameter | Quick memory formula |
| --- | --- | --- |
| FP16 / BF16 (full) | 2 bytes | Params (in billions) × 2 GB |
| INT8 (8-bit) | 1 byte | Params × 1 GB |
| INT4 (4-bit) | 0.5 byte | Params × 0.5 GB |

Example: a 7B model in 4-bit ≈ 7 × 0.5 = **3.5 GB** for weights. Then add roughly **20–40% extra** for activations, the KV cache (grows with context length), and overhead.

## B.2 Inference VRAM by Model Size and Quantization

Approximate VRAM to *run* (not train) a model, including overhead and a modest context window. Round up.

| Model size | FP16 | INT8 | INT4 (Q4) |
| --- | --- | --- | --- |
| 3B | ~7 GB | ~4 GB | ~2.5 GB |
| 7–8B | ~16 GB | ~9 GB | ~5–6 GB |
| 13B | ~28 GB | ~15 GB | ~8–9 GB |
| 34B | ~70 GB | ~38 GB | ~20 GB |
| 70B | ~140 GB | ~75 GB | ~40 GB |
| 8x7B (Mixtral) | ~90 GB | ~50 GB | ~28 GB |

Long contexts (32k+ tokens) add a lot via the KV cache — add several more GB for those.

## B.3 Training / QLoRA VRAM

Training needs much more memory than inference because you also store gradients and optimizer states. Full fine-tuning is very expensive; **QLoRA** (4-bit base + small LoRA adapters) is the practical choice on modest hardware.

| Model size | Full fine-tune (FP16) | LoRA (FP16 base) | QLoRA (4-bit base) |
| --- | --- | --- | --- |
| 7–8B | ~60–80 GB | ~18–24 GB | ~8–12 GB |
| 13B | ~120 GB+ | ~30–40 GB | ~14–18 GB |
| 34B | multi-GPU | ~70 GB | ~24–30 GB |
| 70B | multi-GPU | multi-GPU | ~40–48 GB |

Takeaway: with QLoRA you can fine-tune a 7B model on a single 12 GB consumer GPU, and a 70B on a single 48 GB card.

## B.4 Recommended GPUs

| GPU | VRAM | Good for | Notes |
| --- | --- | --- | --- |
| RTX 3060 | 12 GB | Run 7B (4-bit), QLoRA 7B | Cheap entry point |
| RTX 4070 Ti / 4080 | 12–16 GB | 7B–13B inference | Fast for the price |
| RTX 3090 / 4090 | 24 GB | 13B inference, QLoRA 13–34B | Best consumer value for LLMs |
| RTX A6000 / A6000 Ada | 48 GB | 34B inference, QLoRA 70B | Workstation card |
| L4 | 24 GB | Cost-efficient cloud serving | Low power |
| A100 40GB | 40 GB | Production 13–34B, training | Data-center standard |
| A100 80GB / H100 | 80 GB | 70B serving, big training | Top tier |

Multiple smaller GPUs can be combined (tensor/pipeline parallelism) to run larger models, e.g. 2× 24 GB to serve a 70B in 4-bit.

## B.5 Rough Cloud GPU Costs (Per Hour)

Prices change often — treat these as ballpark, USD per hour, on-demand.

| GPU | VRAM | Rough $/hour |
| --- | --- | --- |
| T4 | 16 GB | $0.35–0.60 |
| L4 | 24 GB | $0.70–1.10 |
| RTX 4090 (community clouds) | 24 GB | $0.40–0.80 |
| A100 40GB | 40 GB | $1.50–2.50 |
| A100 80GB | 80 GB | $2.00–4.00 |
| H100 80GB | 80 GB | $3.00–6.00 |

Spot/preemptible instances can be 50–70% cheaper if your job can tolerate interruptions.

## B.6 Worked Examples

**E-commerce (inference):** You want to serve a Llama-3-8B shopping assistant. In 4-bit it needs ~6 GB, so a single 24 GB RTX 4090 or L4 handles it comfortably with room for a decent context window and batching. At ~$0.80/hour on a community cloud, running it 24/7 for a month costs roughly $580 — often cheaper than per-token API fees at high volume.

**Finance (fine-tuning):** You fine-tune Llama-3-8B on compliance Q&A using QLoRA (~10 GB). A single RTX 3090/4090 (24 GB) or a cloud A100 40GB works. A 3-hour training run on an A100 at ~$2/hour costs about $6 — cheap enough to iterate several times a day.

## B.7 Quick Decision Guide

1. **Just experimenting?** 7B model, 4-bit, on a 12–24 GB GPU or free Colab.
2. **Fine-tuning on a budget?** QLoRA + a 24 GB card.
3. **Serving real traffic?** vLLM on A100/L4; scale out with more GPUs.
4. **Need a 70B?** 4-bit on a single 48 GB card, or two 24 GB cards, or an 80 GB data-center GPU.

When in doubt: quantize first (it's almost free quality-wise at 4-bit), and only rent bigger hardware when a real limit forces you to.
