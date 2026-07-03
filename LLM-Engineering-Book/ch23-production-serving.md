# Chapter 23: Production Serving

A demo serves one person at a time. Production serves thousands at once — reliably, fast, and cheaply. Serving LLMs at scale is hard because they are big, slow, and memory-hungry. Naively calling `model.generate()` in a loop wastes most of your expensive GPU.

This chapter covers the tools and ideas that make LLM serving efficient: **vLLM** (with PagedAttention and continuous batching), **TGI** (Text Generation Inference), and **Ray Serve** (autoscaling). We explain the throughput-vs-latency trade-off, then walk through serving a 70B model to 1,000 concurrent users, and finish with interview questions.

---

## 23.1 vLLM: PagedAttention and Continuous Batching

**Simple definition:** vLLM is a high-performance library for serving LLMs. Its two key innovations are **PagedAttention** (smart memory management for the KV cache) and **continuous batching** (always keeping the GPU busy).

**Intuitive explanation / analogy:**

*PagedAttention* — The KV cache (the model's memory of tokens so far) is the biggest memory user during serving. Naive systems reserve one big fixed block per request, wasting space like booking a whole 10-seat table for every diner even if most come alone. PagedAttention breaks memory into small "pages" (like an operating system managing RAM) and hands out only what each request actually needs. This fits far more requests in the same GPU.

*Continuous batching* — Old systems processed requests in fixed groups: everyone had to finish before the next group started, so fast requests waited for slow ones. Continuous batching is like a rolling checkout line: as soon as one request finishes, a new one takes its place immediately. The GPU never sits idle waiting.

Together these give vLLM many times the throughput of naive `transformers`.

**Runnable code — serve a model with vLLM:**

```bash
# Install: pip install vllm
# Start an OpenAI-compatible server (this is the whole command)
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --port 8000 \
    --gpu-memory-utilization 0.90       # use 90% of VRAM for KV cache + weights
```

```python
# Call the vLLM server with the standard OpenAI client
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="none")
resp = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[{"role": "user", "content": "Summarize why diversification lowers risk."}],
)
print(resp.choices[0].message.content)
```

```python
# For offline batch jobs, use vLLM directly for maximum throughput
from vllm import LLM, SamplingParams

llm = LLM(model="meta-llama/Llama-3.1-8B-Instruct")
prompts = ["Rewrite this product title: red running shoes",
           "Rewrite this product title: blue yoga mat"]
outputs = llm.generate(prompts, SamplingParams(max_tokens=30))
for o in outputs:
    print(o.outputs[0].text)     # vLLM batches these efficiently under the hood
```

**Use case — Finance:** A brokerage serves a market-summary model with vLLM; continuous batching lets it handle a burst of queries at market open on the same GPUs instead of over-provisioning.

**Use case — E-commerce:** A retailer uses vLLM to serve a search-query rewriter; PagedAttention packs many short queries into memory at once, tripling requests-per-second versus naive serving.

---

## 23.2 TGI (Text Generation Inference): Docker Deployment

**Simple definition:** TGI is Hugging Face's production server for LLMs. It is designed to be deployed as a Docker container and comes with continuous batching, quantization support, and monitoring built in.

**Intuitive explanation / analogy:** If vLLM is a high-performance engine, TGI is a ready-to-drive car from Hugging Face: it packages the fast serving engine plus everything around it (metrics, safety, easy config) into one Docker image you can run anywhere Docker runs — your server, AWS, or Kubernetes.

**Runnable code — deploy with Docker:**

```bash
# Run TGI as a Docker container serving a model
# --gpus all gives the container access to the GPUs
model=meta-llama/Llama-3.1-8B-Instruct
volume=$PWD/data          # cache downloaded models here

docker run --gpus all --shm-size 1g -p 8080:80 \
    -v $volume:/data \
    ghcr.io/huggingface/text-generation-inference:latest \
    --model-id $model \
    --max-batch-prefill-tokens 4096      # tune batching for your traffic
```

```python
# Query the TGI server
import requests

resp = requests.post(
    "http://localhost:8080/generate",
    json={"inputs": "Explain what an ETF is.",
          "parameters": {"max_new_tokens": 60}},
)
print(resp.json()["generated_text"])
```

TGI also exposes an OpenAI-compatible endpoint and a `/metrics` endpoint (for Prometheus) so you can watch latency and throughput in production.

**Use case — Finance:** A bank standardizes on TGI Docker images across its Kubernetes cluster so every team deploys models the same monitored, secure way.

**Use case — E-commerce:** A marketplace runs TGI behind an autoscaler; the built-in metrics let the ops team see when to add GPUs during a sale.

---

## 23.3 Ray Serve: Autoscaling LLM Endpoints

**Simple definition:** Ray Serve is a framework for deploying and scaling Python services, including LLMs, across many machines. It can automatically add or remove model copies (replicas) based on traffic.

**Intuitive explanation / analogy:** Ray Serve is like a restaurant manager who watches the queue. When many customers arrive, they open more checkout counters (add replicas); when it is quiet, they close some to save money (remove replicas). You describe how each "counter" works, and Ray handles opening and closing them across your servers.

**Runnable code — an autoscaling vLLM endpoint on Ray Serve:**

```python
# Install: pip install "ray[serve]" vllm
from ray import serve
from vllm import LLM, SamplingParams

# @serve.deployment defines a scalable service.
# autoscaling_config tells Ray to run between 1 and 5 copies based on load.
@serve.deployment(
    autoscaling_config={"min_replicas": 1, "max_replicas": 5,
                        "target_ongoing_requests": 8},   # scale up if backlog grows
    ray_actor_options={"num_gpus": 1},                    # each replica uses 1 GPU
)
class LLMService:
    def __init__(self):
        self.llm = LLM(model="meta-llama/Llama-3.1-8B-Instruct")

    async def __call__(self, request):
        data = await request.json()
        out = self.llm.generate([data["prompt"]], SamplingParams(max_tokens=100))
        return {"text": out[0].outputs[0].text}

# Deploy it — Ray now manages replicas and load balancing for you
serve.run(LLMService.bind(), route_prefix="/generate")
```

With this, if traffic spikes, Ray automatically launches more replicas (up to 5) on available GPUs; when traffic falls, it scales back down to save cost.

**Use case — Finance:** A robo-advisor uses Ray Serve to scale from 1 to 10 replicas during the evening when users check portfolios after work, then scales down overnight.

**Use case — E-commerce:** During a flash sale, Ray Serve automatically scales an LLM endpoint from 2 to 20 GPUs to absorb the traffic wave, then releases the extra GPUs afterward.

---

## 23.4 Throughput vs Latency: Batching, KV Cache, Speculative Decoding

**Simple definition:** *Throughput* is how many requests you finish per second (system-wide). *Latency* is how long one user waits for their answer. Serving is the art of balancing these two, because pushing one often hurts the other.

**Intuitive explanation / analogy:** Think of a bus versus a taxi. A bus (large batch) carries many people at once — high throughput, but each rider waits for others to board (higher latency). A taxi (batch of one) takes you immediately — low latency, but the road carries fewer people overall (low throughput). Good serving finds the right "bus size" for your needs.

The key levers:

| Lever | What it does | Effect |
|-------|--------------|--------|
| **Batching** | Process many requests together on the GPU | ↑ throughput, may ↑ latency |
| **Continuous batching** | Swap finished requests for new ones instantly | ↑ throughput with little latency cost |
| **KV cache** | Reuse attention values so tokens are not recomputed | ↑ speed; uses more memory |
| **Speculative decoding** | A small "draft" model guesses several tokens; the big model checks them at once | ↓ latency (often 2x faster) |

**Speculative decoding explained simply:** A tiny fast model proposes the next few words as a guess. The big accurate model then verifies all of them in a single step. Correct guesses are accepted for free, so the big model produces several tokens per step instead of one — like a fast typist drafting a sentence and an editor approving it in one glance.

```bash
# vLLM supports speculative decoding: a small draft model speeds up a big one
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-70B-Instruct \
    --speculative-model meta-llama/Llama-3.1-8B-Instruct \
    --num-speculative-tokens 5      # let the small model guess 5 tokens ahead
```

**Use case — Finance:** A real-time trading assistant uses speculative decoding to cut answer latency in half, so traders get responses before a market moment passes.

**Use case — E-commerce:** A checkout chatbot tunes batching for high throughput during sales (a slightly longer wait is fine) but switches to low-latency settings for its live-chat feature where speed feels important.

---

## 23.5 Use Case: Serving Llama-70B to 1,000 Concurrent Users

**The scenario.** An online store launches an AI shopping assistant. At peak (a holiday sale), up to **1,000 shoppers** chat with it at the same time, expecting fast replies. The model is **Llama-70B**, chosen for quality.

**The architecture:**

1. **Quantize** the 70B model to Q4/AWQ so it fits on fewer GPUs (~40 GB weights per copy).
2. **Serve with vLLM** for PagedAttention (fit more users in memory) and continuous batching (keep GPUs full).
3. **Autoscale with Ray Serve**, running enough replicas across a GPU cluster to hold 1,000 concurrent conversations.
4. **Load balancer** in front spreads users evenly across replicas.
5. **Speculative decoding** trims latency so replies feel snappy.
6. **Monitoring** (TGI-style metrics or vLLM metrics) watches latency; the autoscaler adds GPUs before users notice slowness.

```python
# Sketch: an autoscaling vLLM-on-Ray deployment sized for peak traffic
from ray import serve
from vllm import LLM, SamplingParams

@serve.deployment(
    autoscaling_config={
        "min_replicas": 4,          # baseline for normal traffic
        "max_replicas": 40,         # burst capacity for the holiday peak
        "target_ongoing_requests": 25,
    },
    ray_actor_options={"num_gpus": 2},   # 70B AWQ needs ~2 GPUs per replica
)
class ShoppingAssistant:
    def __init__(self):
        self.llm = LLM(model="TheBloke/Llama-70B-AWQ", quantization="awq")

    async def __call__(self, req):
        body = await req.json()
        out = self.llm.generate([body["prompt"]], SamplingParams(max_tokens=150))
        return {"reply": out[0].outputs[0].text}

serve.run(ShoppingAssistant.bind(), route_prefix="/assistant")
```

**Rough capacity math:** If each replica (2 GPUs, vLLM continuous batching) comfortably handles ~30 concurrent conversations, then 1,000 users need about **34 replicas**, i.e. ~68 GPUs at peak. Autoscaling means you only pay for those 68 GPUs during the sale; the rest of the year you run just 4 replicas.

**Finance version:** A bank serving a Llama-70B advisor to employees at market open uses the identical pattern — vLLM for efficiency, Ray Serve autoscaling for the morning surge — but keeps everything inside its private cloud for compliance.

**Why this stack:** vLLM maximizes GPUs-worth of users, Ray Serve matches capacity to demand (saving money off-peak), and speculative decoding keeps each reply fast. Quantization makes the 70B model affordable to run at all.

---

## 23.6 Interview Q&A: Why Is vLLM Faster Than Naive Transformers?

**Q1. Why is vLLM faster than naive `transformers` serving?**
Two reasons. **PagedAttention** manages the KV cache in small pages, wasting far less memory and fitting many more requests per GPU. **Continuous batching** keeps the GPU fully busy by swapping in new requests the instant others finish, instead of waiting for a whole batch to complete. Together they multiply throughput.

**Q2. What is PagedAttention in one sentence?**
It is a memory technique that stores the KV cache in small fixed-size pages (like operating-system virtual memory) so the GPU wastes almost no memory and can serve many more concurrent requests.

**Q3. What is continuous batching and why does it matter?**
Instead of grouping requests into fixed batches where everyone waits for the slowest to finish, continuous batching adds a new request the moment a slot frees up. This keeps the GPU busy and greatly raises throughput with little added latency.

**Q4. Explain the difference between throughput and latency.**
Throughput is total requests completed per second across the system; latency is how long a single user waits for their reply. Bigger batches raise throughput but can raise latency; smaller batches lower latency but reduce throughput.

**Q5. What is the KV cache and why is it important for serving?**
The KV cache stores attention keys and values for already-processed tokens so the model does not recompute them each step, which speeds up generation. It is important because it is the main consumer of GPU memory during serving, so managing it well (as PagedAttention does) determines how many users you can serve.

**Q6. What is speculative decoding?**
A small fast "draft" model guesses several upcoming tokens, and the large model verifies all of them in one step. Accepted guesses mean the big model outputs multiple tokens per step, often cutting latency roughly in half with no quality loss.

**Q7. When would you pick TGI over vLLM, or vice versa?**
vLLM is a great default for raw throughput and easy OpenAI-compatible serving. TGI is attractive when you want a polished, Docker-packaged Hugging Face solution with built-in metrics and Kubernetes-friendly deployment. Both use continuous batching; the choice often comes down to ecosystem fit and ops preferences.

**Q8. How does Ray Serve help in production?**
It runs and load-balances multiple model replicas across machines and autoscales them based on traffic, adding replicas during spikes and removing them when quiet. This keeps latency low during peaks and cuts cost during quiet periods.

---

## Key Takeaways

- **Production serving** means handling many users at once, fast and cheaply — naive `model.generate()` loops waste most of the GPU.
- **vLLM** is fast because of **PagedAttention** (page-based KV-cache memory, far less waste) and **continuous batching** (GPU never idles).
- **TGI** packages an efficient serving engine into a Docker image with built-in metrics — great for standardized, Kubernetes-friendly deployment.
- **Ray Serve** autoscales replicas up during spikes and down during quiet times, load-balancing across a GPU cluster to control cost.
- **Throughput vs latency** is the core trade-off; tune with batching, reuse the **KV cache**, and use **speculative decoding** (small draft model + big verifier) to cut latency.
- Serving **Llama-70B to 1,000 users** combines quantization (fits the model), vLLM (fits the users), Ray autoscaling (matches demand), and monitoring — paying for peak GPUs only during the peak.
