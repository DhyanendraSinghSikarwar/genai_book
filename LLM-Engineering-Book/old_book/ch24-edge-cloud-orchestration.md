# Chapter 24: Edge & Cloud Orchestration

So far we have run models on laptops and on big servers. This chapter covers the two extremes and how to manage cost. On one end is the **edge**: running models directly on phones and inside web browsers, with no server at all. On the other end is the **cloud**: renting GPUs from providers, but doing it smartly to pay as little as possible. Orchestration tools like **SkyPilot** and **spot instances** are the keys to running heavy jobs cheaply.

---

## 24.1 MLC LLM / mnn-llm: On-Device Inference (Phones, Browsers)

**Simple definition:** MLC LLM and mnn-llm are frameworks that compile LLMs to run directly on end-user devices — phones, tablets, laptops, and even inside a web browser — using the device's own chip, with no server involved.

**Intuitive explanation / analogy:** Normally your phone app sends your question to a server, which runs the model and sends back the answer. On-device inference is like having a tiny expert living *inside* your phone: it answers instantly, works with no internet, and your words never leave your pocket. The trade-off is that phone chips are smaller, so you run small quantized models.

- **MLC LLM** compiles models to run on many backends: iPhone, Android, Apple Metal, and **WebGPU** (in the browser). It uses heavy quantization (often 3-4 bit) so models fit in phone memory.
- **mnn-llm** is Alibaba's lightweight engine focused on efficient mobile inference, popular for Android and embedded devices.

**Runnable code — a browser LLM with MLC's WebLLM:**

```html
<!-- Runs a small LLM fully inside the browser using WebGPU. No server. -->
<script type="module">
  import { CreateMLCEngine } from "https://esm.run/@mlc-ai/web-llm";

  // Download and load a small quantized model into the browser
  const engine = await CreateMLCEngine("Llama-3.2-1B-Instruct-q4f16_1-MLC");

  // Chat with it — all computation happens on the user's device
  const reply = await engine.chat.completions.create({
    messages: [{ role: "user", content: "Give one quick saving tip." }],
  });
  console.log(reply.choices[0].message.content);
</script>
```

```bash
# MLC LLM also builds native mobile apps. Rough flow using the Python packager:
pip install mlc-llm
# Compile a model for a target device (e.g. iPhone/Android/WebGPU)
mlc_llm compile Llama-3.2-1B-Instruct-q4f16_1 --device android
# The output is bundled into your mobile app.
```

**Use case — Finance:** A banking app embeds a small on-device model so customers get instant, private budgeting tips even in airplane mode — sensitive spending data never leaves the phone.

**Use case — E-commerce:** A shopping app runs an on-device model to give quick product Q&A offline in stores with weak signal, and to keep browsing behavior private on the device.

---

## 24.2 SkyPilot: Run Training/Serving on the Cheapest Cloud GPU

**Simple definition:** SkyPilot is a tool that runs your training or serving jobs on whichever cloud (AWS, GCP, Azure, and others) has the cheapest available GPU right now, automatically handling setup.

**Intuitive explanation / analogy:** Renting GPUs is like booking flights: prices differ by airline (cloud), route (region), and time, and sometimes cheap "standby" seats (spot instances) appear. SkyPilot is your travel agent: you say "get me a GPU to run this job," and it shops across clouds, books the cheapest seat, sets everything up, runs your job, and even rebooks if a seat is cancelled.

You describe the job in a simple **YAML** file: what resources you need and what commands to run.

**Runnable code — a SkyPilot job definition and launch:**

```yaml
# serve_llm.yaml — describe the job for SkyPilot
resources:
  accelerators: A100:1        # ask for 1 A100 GPU (SkyPilot finds the cheapest)
  use_spot: true              # prefer cheap spot/preemptible instances

setup: |                      # runs once to prepare the machine
  pip install vllm

run: |                        # the actual command to run
  python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct --port 8000
```

```bash
# Install: pip install skypilot
# Launch the job — SkyPilot picks the cheapest cloud/region and runs it
sky launch -c my-llm serve_llm.yaml

# Check status, view logs, and tear down when done (to stop paying)
sky status
sky logs my-llm
sky down my-llm            # important: shut it off to stop charges
```

SkyPilot can also expose the endpoint and, with `sky serve`, manage a serving cluster with autoscaling and automatic recovery across clouds.

**Use case — Finance:** A quant team runs nightly model fine-tuning with SkyPilot, which finds the cheapest A100s across three clouds, cutting their compute bill significantly versus sticking to one provider.

**Use case — E-commerce:** A retailer uses SkyPilot to spin up serving capacity in whichever region is cheapest for a weekend sale, then tears it down Monday morning.

---

## 24.3 Spot Instances + Cost Optimization Patterns

**Simple definition:** Spot instances (also called preemptible instances) are spare cloud GPUs offered at a big discount — often 60-90% cheaper — with one catch: the provider can take them back with little warning.

**Intuitive explanation / analogy:** Spot instances are like standby airline seats sold cheap at the gate. You get a great deal, but if a full-fare passenger shows up, you may be bumped. The trick is to use them for work that can survive being interrupted, and to save your progress often so a bump costs you little.

**Cost optimization patterns:**

| Pattern | What it means | Best for |
|---------|---------------|----------|
| **Use spot for interruptible work** | Run jobs that can restart safely on cheap spot GPUs | Training, batch jobs |
| **Checkpoint often** | Save progress regularly so a preemption loses only minutes | Long training runs |
| **Autoscale to demand** | Add GPUs only when traffic is high; remove when low | Serving with variable load |
| **Right-size the model** | Use quantized / smaller models that need fewer GPUs | Everything |
| **Pick cheap regions/clouds** | Prices vary by region and provider — shop around | Any job (SkyPilot automates this) |
| **Shut down idle resources** | Turn off GPUs the moment a job finishes | Avoiding surprise bills |
| **Fallback to on-demand** | If spot is unavailable, fall back to regular pricing to avoid stalls | Time-sensitive jobs |

**Runnable code — spot with automatic checkpointing and recovery:**

```yaml
# train.yaml — cheap spot training that recovers from interruptions
resources:
  accelerators: A100:4
  use_spot: true            # ~70% cheaper GPUs

run: |
  # Resume from the last checkpoint if one exists (survives preemption)
  python train.py \
    --save-every 100 \
    --checkpoint-dir /checkpoints \
    --resume-if-exists
```

```bash
# SkyPilot Managed Jobs auto-recover spot jobs: if the GPU is taken back,
# it finds a new spot GPU and resumes from the last checkpoint automatically.
sky jobs launch -n train-run train.yaml
sky jobs queue          # watch the managed job and its recoveries
```

**A simple cost rule of thumb:**
- **Training / batch jobs:** use spot + checkpointing. Interruptions are fine; you just resume. Big savings.
- **Real-time serving:** use a small base of on-demand GPUs for reliability, plus spot GPUs for extra burst capacity, with autoscaling.

**Use case — Finance:** A bank runs its large monthly risk-model retraining on spot instances with checkpointing every few minutes. When GPUs get reclaimed, the job resumes automatically, and the finance team saves roughly 70% on that compute.

**Use case — E-commerce:** During a holiday sale, a retailer serves its assistant on a small on-demand base for reliability and scales up with cheap spot GPUs for the traffic surge; if spot capacity is pulled, autoscaling and an on-demand fallback keep the service up.

---

## Key Takeaways

- **Edge inference** (MLC LLM, mnn-llm) runs small quantized models directly on phones and in browsers (via WebGPU) — instant, offline, and fully private, at the cost of using smaller models.
- **SkyPilot** is a "travel agent for GPUs": you describe the job in YAML, and it finds and runs it on the cheapest available cloud, handling setup and recovery.
- **Spot instances** are 60-90% cheaper GPUs that can be reclaimed anytime; use them for interruptible work and **checkpoint often** so preemptions cost little.
- Core **cost patterns**: spot for batch/training, autoscale serving to demand, right-size (quantize) models, shop regions/clouds, shut down idle GPUs, and keep an on-demand fallback.
- A smart hybrid — **on-demand base + spot burst + autoscaling** — gives reliable serving at low cost, while **spot + checkpointing** slashes training bills.
- Always remember to **tear down** resources when a job finishes; forgotten GPUs are the most common cause of surprise cloud bills.
