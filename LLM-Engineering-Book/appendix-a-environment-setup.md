# Appendix A: Environment Setup

This appendix helps you set up a working machine for LLM work: Python, virtual environments, CUDA, PyTorch, and how to check your GPU. We keep it simple and give you exact commands. Follow it top to bottom the first time.

## A.1 What You Need

| Thing | Why |
| --- | --- |
| Python 3.10 or 3.11 | Most LLM libraries support these best |
| A virtual environment | Keeps each project's packages separate |
| An NVIDIA GPU (optional but big) | Training and fast inference need one |
| CUDA drivers | Let PyTorch talk to the GPU |
| PyTorch | The core deep-learning library |

You can do a lot with no GPU (using APIs or small local models on CPU), but fine-tuning and fast local inference really want an NVIDIA GPU.

## A.2 Virtual Environments (Do This First)

A virtual environment is a private box of packages for one project. It stops project A from breaking project B.

### Option 1: `venv` (built into Python)

```bash
# Create
python -m venv .venv

# Activate (Windows PowerShell)
.venv\Scripts\Activate.ps1

# Activate (Mac/Linux)
source .venv/bin/activate

# Now install packages here
pip install --upgrade pip
```

### Option 2: `conda` (better for CUDA/GPU work)

Conda can install the right CUDA libraries for you, which avoids many headaches.

```bash
# Install Miniconda first (from the Anaconda site), then:
conda create -n llm python=3.11 -y
conda activate llm
```

**Tip:** Use `conda` when you need GPU/CUDA and want the easy path. Use `venv` for lightweight, API-only projects.

## A.3 Installing PyTorch (The Right Way)

Do NOT just `pip install torch` blindly if you have a GPU — you may get the CPU-only version. Instead, go to the official PyTorch site's selector and copy the command for your CUDA version. Below is a typical example for CUDA 12.1:

```bash
# GPU version (CUDA 12.1) — check pytorch.org for the current line
pip install torch torchvision torchaudio \
  --index-url https://download.pytorch.org/whl/cu121

# CPU only (no NVIDIA GPU)
pip install torch torchvision torchaudio
```

## A.4 Check CUDA and Your Driver

Before installing PyTorch, confirm the GPU and driver are visible:

```bash
nvidia-smi
```

You should see a table with your GPU name, driver version, and the highest CUDA version the driver supports. If this command is "not found," install the NVIDIA driver first.

Key idea: the **driver's** CUDA version (top-right of `nvidia-smi`) must be **equal to or higher** than the CUDA version PyTorch was built for.

## A.5 Verify GPU Works in Python

After installing PyTorch, run this small script:

```python
import torch

print("PyTorch version:", torch.__version__)
print("CUDA available:", torch.cuda.is_available())

if torch.cuda.is_available():
    print("GPU name:", torch.cuda.get_device_name(0))
    print("GPU count:", torch.cuda.device_count())
    x = torch.rand(3, 3).cuda()      # move a tensor to the GPU
    print("Tensor on GPU:", x.device)
else:
    print("Running on CPU only.")
```

If `CUDA available` prints `True` and you see your GPU name, you are ready.

## A.6 Core LLM Packages

Once PyTorch works, install the common toolkit:

```bash
pip install transformers accelerate datasets
pip install sentence-transformers          # embeddings
pip install langchain langchain-community  # RAG / agents
pip install chromadb                       # local vector DB
pip install trl peft bitsandbytes          # fine-tuning + quantization
pip install wandb                          # experiment tracking
```

Freeze your working setup so you can reproduce it later:

```bash
pip freeze > requirements.txt
```

## A.7 Common Pitfalls (And Fixes)

| Problem | Cause | Fix |
| --- | --- | --- |
| `torch.cuda.is_available()` is `False` | Installed CPU-only PyTorch | Reinstall using the CUDA `--index-url` |
| `CUDA out of memory` | Model too big for VRAM | Use quantization (4-bit) or a smaller model; lower batch size |
| `bitsandbytes` won't load on Windows | Old versions were Linux-only | Use a recent version, or run under WSL2 / Linux |
| Version conflicts between libs | Mixed installs in one env | Make a fresh env; pin versions in `requirements.txt` |
| Driver/CUDA mismatch | Driver older than PyTorch's CUDA | Update the NVIDIA driver |
| Slow install / timeouts | Big wheels | Upgrade pip; use a stable connection |
| Model download fails | No auth for gated models | Run `huggingface-cli login` and accept the model license |

## A.8 A Reproducible First Project (Copy-Paste)

```bash
conda create -n llm python=3.11 -y
conda activate llm
pip install --upgrade pip
pip install torch --index-url https://download.pytorch.org/whl/cu121
pip install transformers accelerate datasets sentence-transformers
python -c "import torch; print(torch.cuda.is_available())"
```

If the last line prints `True`, your environment is ready for the rest of this book. Save the four setup lines in a `setup.sh` (or `setup.ps1`) file so you never have to remember them again.

## A.9 Using Cloud GPUs Instead

No local GPU? Rent one. Services like Google Colab, Kaggle, RunPod, Lambda, and Vast.ai give you a GPU by the hour. On these, CUDA and drivers are usually pre-installed — you just create your venv/conda env and `pip install` your packages. See Appendix B for rough costs.
