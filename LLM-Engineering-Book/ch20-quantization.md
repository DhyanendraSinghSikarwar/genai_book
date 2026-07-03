# Chapter 20: Quantization

Large language models are big. A 7-billion parameter model stored in full precision needs about 28 GB of memory. A 70-billion parameter model needs around 280 GB. Most people do not have that kind of hardware. Quantization is the trick that shrinks these models so they run on a laptop, a single GPU, or a cheap cloud server, while keeping most of their intelligence.

In this chapter we learn what quantization is, why it works, and the main tools used in the real world: GGUF (llama.cpp), GPTQ, AWQ, SmoothQuant, and bitsandbytes. We finish with a comparison table and interview questions.

---

## 20.1 Simple Definition: FP16 to INT8/INT4, Why It Works

**Simple definition:** Quantization means storing the model's numbers (weights) using fewer bits. Instead of keeping each number as a high-precision 16-bit floating point value (FP16), we store it as an 8-bit integer (INT8) or even a 4-bit integer (INT4).

**Intuitive explanation / analogy:** Imagine you are writing down the price of items in a shop. You could write "12.347619 dollars" (very precise, takes lots of space) or you could round it to "12.35 dollars" (less precise, takes less space). For most shopping decisions, the rounded price is good enough. Quantization does the same thing to a model's millions of numbers: it rounds them so they take less memory. The model becomes smaller and faster, and because neural networks are naturally tolerant of small errors, it still gives almost the same answers.

Why does it work? A neural network has billions of weights. Each individual weight only contributes a tiny amount to the final answer. Rounding one weight a little bit barely matters, and the small errors tend to cancel each other out across billions of weights.

Here is the memory math. The number of bits per weight directly sets the model size:

| Precision | Bits per weight | Size of a 7B model | Size of a 70B model |
|-----------|-----------------|--------------------|---------------------|
| FP32 (full) | 32 | ~28 GB | ~280 GB |
| FP16 / BF16 | 16 | ~14 GB | ~140 GB |
| INT8 | 8 | ~7 GB | ~70 GB |
| INT4 | 4 | ~3.5 GB | ~35 GB |

**Runnable code — see quantization in action with a tiny example:**

```python
# Demonstrate the idea of quantization on a simple array of weights.
# We map high-precision float values into low-precision integers.
import numpy as np

# Imagine these are a few model weights stored in high precision (float32)
weights = np.array([0.12, -0.85, 0.44, -0.03, 0.91], dtype=np.float32)

# --- Quantize to INT8 (values 0..255) ---
w_min, w_max = weights.min(), weights.max()       # find the range
scale = (w_max - w_min) / 255                      # size of each "step"
quantized = np.round((weights - w_min) / scale).astype(np.uint8)  # to integers

# --- Dequantize (bring back to float to use the model) ---
dequantized = quantized * scale + w_min

print("Original :", weights)
print("Quantized:", quantized)          # small integers, cheap to store
print("Restored :", dequantized)        # close to original, small error
print("Max error:", np.abs(weights - dequantized).max())
```

The key idea: we store a small integer per weight plus one shared `scale` and `w_min`. At run time we "dequantize" back to a usable number. The rounding introduces a tiny error, which is the price we pay for a much smaller model.

**Use case — Finance:** A bank wants to run a document-summarization model on-premises (inside its own building) for regulatory reasons. The full FP16 model needs an expensive 2-GPU server. After INT4 quantization the same model fits on a single mid-range GPU, cutting hardware cost by more than half while summaries stay accurate enough for analysts to review loan documents.

**Use case — E-commerce:** An online retailer runs a product-recommendation assistant. During Black Friday, traffic explodes. Using an INT8 quantized model, they fit twice as many model copies on the same servers, doubling how many shoppers they can serve per second without buying new hardware.

---

## 20.2 GGUF (llama.cpp): Quant Levels and Conversion

**Simple definition:** GGUF is a file format used by the popular `llama.cpp` project to store quantized models. It packs the model weights, the tokenizer, and settings into a single file that runs efficiently on CPUs and GPUs — even on a laptop.

**Intuitive explanation / analogy:** Think of GGUF as a "zip file" specially designed for LLMs. Just as a zip file bundles many files into one compressed package, a GGUF file bundles a whole quantized model into one portable file you can download and run anywhere.

GGUF offers many quantization levels. The names look cryptic but follow a pattern: `Q<bits>_<variant>`.

| GGUF level | Bits (approx) | Meaning | When to use |
|------------|---------------|---------|-------------|
| Q8_0 | 8 | Almost lossless | You have plenty of memory, want top quality |
| Q6_K | ~6.5 | Very high quality | Great balance for capable machines |
| Q5_K_M | ~5.5 | High quality | Good quality, moderate size |
| Q4_K_M | ~4.5 | Recommended default | Best balance of size, speed, quality |
| Q4_K_S | ~4 | Smaller | Tight on memory |
| Q3_K_M | ~3.5 | Noticeable quality loss | Very limited hardware |
| Q2_K | ~2.5 | Big quality loss | Only if desperate for size |

The `_K` means "K-quant", a smarter method that gives better quality at the same size. `_M` (medium) and `_S` (small) tune the size further. **`Q4_K_M` is the most popular choice** because it gives near-full quality at roughly a quarter of the size.

**Runnable code — convert a HuggingFace model to GGUF and quantize it:**

```bash
# Step 1: get llama.cpp (it contains the conversion tools)
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
pip install -r requirements.txt

# Step 2: convert a HuggingFace model to an unquantized GGUF (FP16)
# Assume the model is downloaded into ./models/my-llama-7b
python convert_hf_to_gguf.py ./models/my-llama-7b \
    --outfile my-llama-7b-f16.gguf \
    --outtype f16

# Step 3: build the quantizer, then quantize to Q4_K_M
cmake -B build && cmake --build build --config Release
./build/bin/llama-quantize my-llama-7b-f16.gguf \
    my-llama-7b-Q4_K_M.gguf Q4_K_M

# Step 4: run the quantized model and chat with it
./build/bin/llama-cli -m my-llama-7b-Q4_K_M.gguf \
    -p "Explain compound interest in one sentence." -n 64
```

**Use case — Finance:** A small credit union with no GPU budget runs a Q4_K_M model directly on CPU laptops for staff to draft customer emails. GGUF's CPU efficiency means no cloud costs and no customer data leaving the building.

**Use case — E-commerce:** A merchant packages a Q4_K_M product-description generator as a single GGUF file and ships it to store managers' PCs. One portable file, no complex setup, works offline in the shop's back office.

---

## 20.3 GPTQ and AWQ: GPU-Optimized Quantization

**Simple definition:** GPTQ and AWQ are two methods that quantize a model specifically to run fast on GPUs, usually at 4 bits. Unlike simple rounding, they are "smart" methods that look at real data to decide how to round, so quality stays higher.

**Intuitive explanation / analogy:** Simple quantization rounds every weight the same blunt way. GPTQ and AWQ are like a careful tailor: they measure which weights matter most and protect those, while rounding the less important ones more aggressively. The result is a suit (model) that fits well even though less fabric (memory) was used.

- **GPTQ** (Generative Pre-trained Transformer Quantization): rounds weights one layer at a time while checking how much error each rounding adds, then adjusts the remaining weights to compensate.
- **AWQ** (Activation-aware Weight Quantization): notices that a small number of weights are very important (because they multiply large activations) and keeps those in higher precision. Often gives slightly better quality than GPTQ and is fast.

**Runnable code — quantize with AWQ, then run:**

```python
# Install: pip install autoawq transformers
from awq import AutoAWQForCausalLM
from transformers import AutoTokenizer

model_path = "meta-llama/Llama-2-7b-hf"    # source model
quant_path = "llama2-7b-awq"               # where to save quantized model

# Settings: 4-bit, group size 128 is a common good default
quant_config = {"zero_point": True, "q_group_size": 128,
                "w_bit": 4, "version": "GEMM"}

# Load, quantize using calibration data (AWQ downloads a small dataset), save
model = AutoAWQForCausalLM.from_pretrained(model_path)
tokenizer = AutoTokenizer.from_pretrained(model_path)
model.quantize(tokenizer, quant_config=quant_config)  # the smart quantization
model.save_quantized(quant_path)
tokenizer.save_pretrained(quant_path)
print("Saved 4-bit AWQ model to", quant_path)
```

```python
# Loading a GPTQ model is just as easy (models are on HuggingFace pre-quantized)
# Install: pip install auto-gptq transformers optimum
from transformers import AutoModelForCausalLM, AutoTokenizer

model_id = "TheBloke/Llama-2-7B-GPTQ"   # a ready-made GPTQ model
tok = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, device_map="auto")

inputs = tok("The best way to save money is", return_tensors="pt").to("cuda")
out = model.generate(**inputs, max_new_tokens=40)
print(tok.decode(out[0]))
```

**Use case — Finance:** A trading-signals startup serves an AWQ 4-bit model on a single GPU to answer analyst questions in real time. AWQ's high quality means numeric reasoning about earnings reports stays reliable, and the small size lets them batch many requests together.

**Use case — E-commerce:** A marketplace uses a GPTQ model to auto-moderate seller listings at scale on GPUs. The 4-bit model runs fast enough to check thousands of new listings per minute for banned words and policy violations.

---

## 20.4 SmoothQuant and bitsandbytes (`load_in_4bit`)

**Simple definition:** SmoothQuant is a technique that makes INT8 quantization work better by "smoothing" difficult values. bitsandbytes is an easy-to-use library that lets you load any HuggingFace model in 8-bit or 4-bit with a single line of code.

**Intuitive explanation / analogy:** Some layers in a model have a few huge "outlier" numbers that are hard to quantize — like trying to fit both a mouse and an elephant on the same scale. SmoothQuant shifts some of that difficulty from the activations into the weights so both become easier to quantize. bitsandbytes, on the other hand, is the "easy button": you do not pre-process anything; it quantizes on the fly as the model loads.

**Runnable code — load any model in 4-bit with bitsandbytes:**

```python
# Install: pip install bitsandbytes transformers accelerate
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
import torch

# Configure 4-bit (NF4) quantization — the modern high-quality setting
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,                       # turn on 4-bit loading
    bnb_4bit_quant_type="nf4",               # NF4 works well for LLM weights
    bnb_4bit_compute_dtype=torch.float16,    # do the math in fp16 for accuracy
    bnb_4bit_use_double_quant=True,          # quantize the scales too (extra savings)
)

model_id = "mistralai/Mistral-7B-v0.1"
tok = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    quantization_config=bnb_config,          # apply the 4-bit config
    device_map="auto",                       # place layers on available GPUs
)

# The 7B model now fits in about 4-5 GB of VRAM instead of 14 GB
inputs = tok("A safe investment for beginners is", return_tensors="pt").to(model.device)
print(tok.decode(model.generate(**inputs, max_new_tokens=30)[0]))
```

```python
# For 8-bit, just switch the flag:
bnb_8bit = BitsAndBytesConfig(load_in_8bit=True)
# model = AutoModelForCausalLM.from_pretrained(model_id,
#     quantization_config=bnb_8bit, device_map="auto")
```

bitsandbytes is the go-to choice during **fine-tuning** (for example with QLoRA), because you can load a big model in 4-bit and train small adapters on top.

**Use case — Finance:** A fintech team fine-tunes a 13B model to classify transactions as fraud or normal. Using `load_in_4bit`, they fit the model on one affordable GPU and train it overnight, instead of renting a costly multi-GPU machine.

**Use case — E-commerce:** A retailer loads its customer-support model with `load_in_8bit` so it fits on existing hardware during a seasonal spike, buying time before a bigger hardware upgrade.

---

## 20.5 Quality vs Size vs Speed (7B Model Across Quant Levels)

The whole point of quantization is a trade-off: smaller and faster, but with some quality loss. The table below gives approximate numbers for a typical 7B model. Exact values depend on hardware and the specific model, but the relationships hold in practice.

| Format | Bits | File size | VRAM needed | Relative speed | Quality (higher = better) | Notes |
|--------|------|-----------|-------------|----------------|---------------------------|-------|
| FP16 | 16 | ~14 GB | ~15 GB | 1.0x (baseline) | 100% | Full quality, biggest |
| INT8 (bnb) | 8 | ~7 GB | ~8 GB | ~0.9x | ~99% | Easy, minor loss |
| Q8_0 (GGUF) | 8 | ~7.2 GB | ~8 GB | Fast on CPU/GPU | ~99.5% | Near lossless |
| Q6_K (GGUF) | 6.5 | ~5.5 GB | ~6.5 GB | Fast | ~99% | Excellent balance |
| Q5_K_M (GGUF) | 5.5 | ~4.8 GB | ~6 GB | Fast | ~98.5% | High quality |
| AWQ 4-bit | 4 | ~4 GB | ~5 GB | Very fast on GPU | ~98% | Great for GPU serving |
| GPTQ 4-bit | 4 | ~4 GB | ~5 GB | Very fast on GPU | ~97.5% | Strong GPU option |
| Q4_K_M (GGUF) | 4.5 | ~4.1 GB | ~5.5 GB | Fast everywhere | ~97.5% | Most popular default |
| NF4 (bnb) | 4 | ~4 GB | ~5 GB | Fast | ~97% | Great for fine-tuning |
| Q3_K_M (GGUF) | 3.5 | ~3.3 GB | ~4.5 GB | Fast | ~93% | Visible quality drop |
| Q2_K (GGUF) | 2.5 | ~2.6 GB | ~3.5 GB | Fastest | ~85% | Use only if forced |

**How to read this:** As you move down the table, the model gets smaller and often faster, but quality slowly drops. The "sweet spot" for most people is around 4 to 5 bits (Q4_K_M, AWQ, or NF4), where you cut the size by ~70% but lose only 2-3% quality.

**Rule of thumb:** Do not go below 4 bits unless memory forces you to. Below Q3, the model starts making noticeably more mistakes.

**Use case — Finance:** A risk team A/B tests Q4_K_M vs Q6_K for a compliance chatbot. They find Q4_K_M answers 99% of test questions identically but uses 25% less memory, so they ship Q4_K_M and run more copies for peak hours.

**Use case — E-commerce:** A search team measures that AWQ 4-bit is 2x faster than FP16 for their query-rewriting model with almost no quality loss, letting them handle a holiday traffic surge on the same GPU fleet.

---

## 20.6 Interview Q&A: GGUF vs GPTQ vs AWQ

**Q1. In one sentence, what is quantization and why do we use it?**
Quantization stores a model's weights using fewer bits (for example 4-bit instead of 16-bit), which shrinks the model and speeds it up, so it fits on cheaper hardware while keeping most of its accuracy.

**Q2. What is the main difference between GGUF, GPTQ, and AWQ?**
GGUF is a file format (used by llama.cpp) optimized to run on CPUs and GPUs, great for laptops and offline use. GPTQ and AWQ are GPU-focused 4-bit methods for fast serving. GPTQ minimizes rounding error layer by layer; AWQ protects the most important weights based on activations, often giving slightly better quality.

**Q3. If a customer wants to run an LLM on a laptop with no GPU, what do you recommend?**
GGUF with llama.cpp (or Ollama, which uses it under the hood). GGUF is built for efficient CPU inference, and a Q4_K_M file gives good quality at a small size.

**Q4. Why is a 4-bit model not four times worse than a 16-bit model?**
Because neural networks have billions of weights, and each one contributes only a tiny bit to the answer. Rounding errors are small and tend to cancel out across so many weights, so quality drops only a few percent, not proportionally to the bit reduction.

**Q5. When would you choose AWQ over GPTQ?**
When you want top quality at 4-bit on a GPU and can tolerate a slightly longer quantization step. AWQ's activation-aware approach often preserves accuracy a little better, especially for smaller models. Both are excellent; the best choice can depend on the specific model, so testing both is wise.

**Q6. What does `load_in_4bit=True` in bitsandbytes do, and when is it most useful?**
It loads a HuggingFace model in 4-bit precision on the fly, cutting VRAM by about 4x with one config line. It is most useful for fine-tuning (QLoRA) and quick experiments, because it needs no separate pre-quantization step.

**Q7. What does the "M" in Q4_K_M mean?**
The K means it uses the improved K-quant method; the M means "medium" size within that family (there are also S for small and L for large). Q4_K_M is the widely recommended balance of quality and size.

**Q8. What is the biggest risk of going below 4 bits (e.g., Q2)?**
Quality drops sharply. The model makes more factual mistakes, follows instructions less reliably, and reasons worse. Very low-bit quantization is only worth it when memory is extremely tight and some quality loss is acceptable.

---

## Key Takeaways

- **Quantization** stores model weights with fewer bits (16 to 8 to 4) to shrink size and boost speed, losing only a little quality because errors average out over billions of weights.
- **GGUF (llama.cpp)** is the go-to format for CPUs, laptops, and offline use; `Q4_K_M` is the popular default balance.
- **GPTQ and AWQ** are smart 4-bit methods optimized for fast GPU serving; AWQ protects the most important weights, GPTQ minimizes layer-by-layer error.
- **bitsandbytes** offers the easiest path with `load_in_4bit` / `load_in_8bit`, ideal for fine-tuning and quick tests; **SmoothQuant** improves INT8 by smoothing outlier values.
- The **sweet spot** for most deployments is 4 to 5 bits: about 70% smaller with only 2-3% quality loss. Avoid going below 4 bits unless memory forces you.
- Always **measure** quality on your own tasks before shipping a quantized model — the right level depends on your data.
