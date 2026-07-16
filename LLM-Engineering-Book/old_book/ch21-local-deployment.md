# Chapter 21: Local Deployment

Sometimes you do not want to send your data to a cloud API. Maybe the data is private (medical records, legal contracts, financial statements), maybe you want zero ongoing cost, or maybe you just want your app to work with no internet. Running a large language model on your own machine — "local deployment" — solves all of these.

This chapter covers the most popular tools for running open models on your own hardware: Ollama, LM Studio, and kobold.cpp. We then look at which open models to choose and how much video memory (VRAM) they need, and finish with a private-assistant use case and interview questions.

---

## 21.1 Ollama: Pull, Run, Modelfile, REST API

**Simple definition:** Ollama is a free tool that lets you download and run open LLMs on your own computer with simple commands, like `ollama run llama3`. It also starts a local web server so your apps can talk to the model.

**Intuitive explanation / analogy:** Ollama is like "Docker for language models." Just as Docker lets you `docker pull` and `docker run` an app without worrying about setup, Ollama lets you `ollama pull` and `ollama run` a model without worrying about weights, tokenizers, or quantization. It handles all of that behind the scenes (using GGUF and llama.cpp internally).

**Runnable code — the basics:**

```bash
# 1. Download and run a model (auto-downloads on first use)
ollama pull llama3.1          # download the model
ollama run llama3.1           # chat with it in your terminal

# 2. List and remove models
ollama list                   # show downloaded models
ollama rm llama3.1            # delete a model to free disk space

# 3. Pick a specific quantization tag
ollama run llama3.1:8b-instruct-q4_K_M   # 8B model, Q4_K_M quant
```

A **Modelfile** lets you customize a model — set a system prompt, temperature, or build on an existing model. It is like a Dockerfile for models.

```bash
# Create a file named "Modelfile" with this content:
```
```text
FROM llama3.1                       # start from an existing model
PARAMETER temperature 0.3           # make answers more focused
SYSTEM """
You are a polite banking assistant. Only answer questions about
personal finance. If asked anything else, say you cannot help.
"""
```
```bash
# Build your custom model and run it
ollama create finance-bot -f Modelfile
ollama run finance-bot
```

Ollama exposes a **REST API** on `http://localhost:11434`, so any app can send requests.

```python
# Talk to Ollama from Python using its REST API
import requests

response = requests.post(
    "http://localhost:11434/api/generate",
    json={
        "model": "llama3.1",
        "prompt": "Explain what a stock dividend is, simply.",
        "stream": False,          # get the full answer at once
    },
)
print(response.json()["response"])
```

```python
# Ollama also speaks the OpenAI format, so existing OpenAI code works:
from openai import OpenAI

client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")  # key ignored
reply = client.chat.completions.create(
    model="llama3.1",
    messages=[{"role": "user", "content": "List 3 budgeting tips."}],
)
print(reply.choices[0].message.content)
```

**Use case — Finance:** A wealth-management firm runs `finance-bot` (a Modelfile-customized model) on office machines so advisors can draft client summaries without any client data touching the internet.

**Use case — E-commerce:** A store builds an internal tool where staff paste product specs and get marketing copy back, all served by Ollama's local API — no per-request API fees.

---

## 21.2 LM Studio: GUI + Local OpenAI-Compatible Server

**Simple definition:** LM Studio is a desktop application (with a friendly graphical interface) for downloading, chatting with, and serving local LLMs. It requires no command line.

**Intuitive explanation / analogy:** If Ollama is the command-line tool for developers, LM Studio is the "app store plus chat window" for everyone. You browse models with your mouse, click download, and start chatting — like using a music app to find and play songs.

Key features:
- A **search-and-download** screen for HuggingFace GGUF models.
- A **chat window** to talk to models directly.
- A **Local Server** tab that starts an OpenAI-compatible API with one click, so your code can connect just like it would to OpenAI.

**Runnable code — connect to the LM Studio local server:**

```python
# After clicking "Start Server" in LM Studio (default port 1234),
# use the standard OpenAI client — just change the base URL.
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:1234/v1",   # LM Studio's local server
    api_key="lm-studio",                    # any string works; it is ignored
)

response = client.chat.completions.create(
    model="local-model",                    # LM Studio uses the loaded model
    messages=[
        {"role": "system", "content": "You are a helpful shopping assistant."},
        {"role": "user", "content": "Suggest a gift under $50 for a coffee lover."},
    ],
    temperature=0.7,
)
print(response.choices[0].message.content)
```

**Use case — Finance:** A compliance officer who does not code uses LM Studio's GUI to test whether different local models correctly refuse to give investment advice, before the IT team deploys the chosen one.

**Use case — E-commerce:** A product manager prototypes a customer-review summarizer in LM Studio's chat window, confirms it works, then flips on the local server so an engineer can wire it into a dashboard.

---

## 21.3 kobold.cpp Overview

**Simple definition:** kobold.cpp is a single-file program that runs GGUF models locally, originally built for AI-assisted creative writing and role-play, with a built-in web interface.

**Intuitive explanation / analogy:** kobold.cpp is the "all-in-one portable" option. You download one executable file, point it at a GGUF model, and it opens a web page where you can chat or write stories — no installation, no dependencies.

Highlights:
- **Zero install:** one executable runs on Windows, Mac, and Linux.
- **Built-in web UI** for chat and long-form writing.
- **OpenAI-compatible API** so it can also serve apps.
- Popular for **long context** and creative tasks, with fine control over sampling.

```bash
# Run kobold.cpp with a GGUF model (example on Linux/Mac)
./koboldcpp --model mistral-7b-Q4_K_M.gguf --port 5001
# Then open http://localhost:5001 in your browser to chat.
```

```python
# It exposes an OpenAI-compatible endpoint too:
from openai import OpenAI
client = OpenAI(base_url="http://localhost:5001/v1", api_key="none")
print(client.chat.completions.create(
    model="koboldcpp",
    messages=[{"role": "user", "content": "Write a product tagline for eco sneakers."}],
).choices[0].message.content)
```

**Use case — E-commerce:** A marketing team uses kobold.cpp's writing interface to brainstorm dozens of catchy product descriptions offline during a company retreat with no reliable internet.

**Use case — Finance:** An analyst runs kobold.cpp on an air-gapped (no internet) machine to draft internal market-commentary narratives from sensitive numbers that must never leave the network.

---

## 21.4 Running Open Models Locally: DeepSeek, Llama, Qwen, Gemma (VRAM Sizing)

**Simple definition:** These are the leading families of open-weight models you can download and run yourself. Choosing one depends on your task and, crucially, how much GPU memory (VRAM) you have.

**Intuitive explanation / analogy:** Picking a local model is like picking a car for your garage. A big model is a powerful truck (needs a big garage / lots of VRAM); a small model is a compact car (fits anywhere, less powerful). You must match the model's "size" to your hardware's "garage space."

Quick guide to the families:
- **Llama (Meta):** strong all-round general models, huge community support.
- **Qwen (Alibaba):** excellent at coding and multilingual tasks, many sizes.
- **DeepSeek:** strong reasoning and coding; some very large mixture-of-experts models.
- **Gemma (Google):** lightweight, efficient models that run well on modest hardware.

**VRAM sizing table (approximate, for Q4 quantization).** VRAM must hold the model weights *plus* extra room for the context (KV cache), so always leave a buffer.

| Model size | Q4 weights size | Recommended VRAM | Example GPU | Notes |
|------------|-----------------|------------------|-------------|-------|
| 1-3B | ~1-2 GB | 4 GB | Any modern GPU / laptop | Runs on CPU too |
| 7-8B | ~4-5 GB | 6-8 GB | RTX 3060 / 4060 | Great everyday size |
| 13-14B | ~8 GB | 12 GB | RTX 3080 / 4070 | Noticeably smarter |
| 30-34B | ~18-20 GB | 24 GB | RTX 3090 / 4090 | Prosumer sweet spot |
| 70B | ~35-40 GB | 48 GB | 2x 24 GB or 1x A6000 | Needs serious hardware |
| 100B+ (MoE) | varies | 80 GB+ | A100 / H100 | Data-center class |

**Runnable code — check your VRAM and estimate what fits:**

```python
import torch

if torch.cuda.is_available():
    total_gb = torch.cuda.get_device_properties(0).total_memory / 1e9
    print(f"GPU: {torch.cuda.get_device_name(0)}  VRAM: {total_gb:.1f} GB")
    # Rough rule: usable for weights ~= 80% of VRAM (rest for context/overhead)
    budget = total_gb * 0.8
    print(f"Budget for Q4 weights: ~{budget:.1f} GB")
    # A Q4 model needs roughly (params_in_billions * 0.55) GB
    max_params = budget / 0.55
    print(f"Largest Q4 model you can comfortably run: ~{max_params:.0f}B params")
else:
    print("No GPU found — use small models (1-8B) on CPU via Ollama/GGUF.")
```

**Use case — Finance:** A hedge fund with a single RTX 4090 (24 GB) runs a 34B Qwen model at Q4 for in-depth analysis of earnings-call transcripts, choosing 34B over 70B because it fits their one GPU.

**Use case — E-commerce:** A startup on RTX 4060 laptops (8 GB) standardizes on Gemma 7B at Q4 for product-tagging, because it fits comfortably and runs fast on the hardware employees already have.

---

## 21.5 Use Case: Fully Offline Private Assistant

**The scenario — a law firm.** A law firm handles confidential case files. Sending client documents to a cloud AI would violate attorney-client privilege and data-protection rules. They need an assistant that summarizes contracts, drafts letters, and answers questions about internal documents — with **zero data leaving the building**.

**The solution (fully local stack):**
1. **Hardware:** one on-prem server with a 48 GB GPU (or two 24 GB GPUs).
2. **Model:** a 70B Llama model quantized to Q4, or a 34B Qwen model for a lighter setup.
3. **Serving:** Ollama runs the model and exposes a local API on the firm's private network only.
4. **Interface:** an internal web app (built with the OpenAI-compatible API) that lawyers use in the browser.
5. **Privacy:** the server has **no outbound internet access**, so documents physically cannot leave.

```python
# The internal app calls the on-prem Ollama server — no cloud involved.
from openai import OpenAI

client = OpenAI(base_url="http://internal-ai.lawfirm.local:11434/v1",
                api_key="local")

contract_text = open("nda_draft.txt").read()   # sensitive document, stays local
resp = client.chat.completions.create(
    model="llama3.1:70b-instruct-q4_K_M",
    messages=[
        {"role": "system", "content": "You are a legal assistant. Summarize clearly."},
        {"role": "user", "content": f"Summarize the key risks in this NDA:\n{contract_text}"},
    ],
)
print(resp.choices[0].message.content)
```

**Finance version of the same pattern.** A bank uses an identical on-premises setup for regulatory reasons. Customer account data and internal risk models are highly regulated; running the LLM on-prem means auditors can confirm no customer data ever leaves the bank's controlled environment. The bank uses the local assistant to draft compliance reports, explain policies to staff, and answer internal questions — all offline.

**Why local wins here:** privacy (data never leaves), compliance (auditable, controlled), cost (no per-token fees at scale), and availability (works even if the internet is down).

---

## 21.6 Interview Q&A: How Much VRAM for a 70B Model at Q4?

**Q1. How much VRAM does a 70B model need at Q4?**
The weights alone are about 35-40 GB at 4-bit. Adding room for the context (KV cache) and overhead, you want roughly 48 GB of VRAM — for example, two 24 GB GPUs or one 48 GB card. Longer contexts and larger batches need even more.

**Q2. Give the quick formula to estimate Q4 model VRAM.**
Roughly: `VRAM (GB) ≈ params_in_billions × 0.5 to 0.6` for the weights, then add 10-30% for context and overhead. So 70B × 0.55 ≈ 38 GB weights, plus buffer ≈ 48 GB total.

**Q3. What is the KV cache and why does it affect VRAM?**
The KV cache stores the attention keys and values for tokens already processed, so the model does not recompute them. It grows with context length and batch size. Long conversations or many simultaneous users can consume several extra GB, so you must leave headroom beyond the weights.

**Q4. My GPU has 24 GB. Can I run a 70B model?**
Not comfortably at Q4 on one card. Options: use two 24 GB GPUs, run a smaller model (34B fits nicely in 24 GB at Q4), use a more aggressive quant like Q3 (quality drops), or offload some layers to CPU RAM (much slower).

**Q5. What is CPU offloading and what is the trade-off?**
Offloading means keeping some model layers in system RAM instead of VRAM when the model does not fully fit. It lets you run bigger models on smaller GPUs, but it is much slower because data must move between RAM and GPU on every token.

**Q6. Ollama vs LM Studio vs kobold.cpp — when to use each?**
Ollama: best for developers who want simple commands and a clean API. LM Studio: best for non-coders who want a GUI and easy model browsing. kobold.cpp: best for a zero-install, portable single file, popular for creative writing and long context. All run GGUF models locally.

**Q7. Why choose local deployment over a cloud API?**
Privacy (data stays on your hardware), compliance (auditable and controlled), predictable cost (no per-token fees), and offline availability. The trade-offs are upfront hardware cost and the effort to run and maintain it yourself.

**Q8. How can I fit a 70B model on modest hardware if I must?**
Use lower-bit quantization (Q3 or Q2, accepting quality loss), enable CPU offloading, reduce the context length to shrink the KV cache, or serve one request at a time (batch size 1). Better still, pick a smaller model that fits your GPU well.

---

## Key Takeaways

- **Local deployment** runs open LLMs on your own hardware for privacy, compliance, cost control, and offline use.
- **Ollama** is the developer-friendly "Docker for models" with `pull`/`run`, customizable **Modelfiles**, and a REST + OpenAI-compatible API on port 11434.
- **LM Studio** is the GUI option — browse, download, chat, and start a local OpenAI-compatible server (port 1234) with no coding.
- **kobold.cpp** is a portable single-file runner with a built-in web UI, great for creative writing and offline use.
- Leading open families are **Llama, Qwen, DeepSeek, and Gemma**; match model size to your **VRAM** using the sizing table.
- A **70B model at Q4** needs roughly **48 GB VRAM** (weights ~38 GB plus KV-cache/overhead). Estimate with `params × ~0.55` plus a buffer.
- Fully offline stacks (law firm, bank) keep sensitive data on-prem while still giving powerful AI assistance.
