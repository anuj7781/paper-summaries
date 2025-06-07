
# LLMs Through a Storage Developer’s Lens

As a system software or storage developer, you're probably used to thinking about throughput, caching, IOPS, and memory pressure.What you may not realize is that these concerns apply directly to working with **Large Language Models (LLMs)**. This includes training, fine-tuning, and even inference.

In this blog, we look at LLMs not just as machine learning artifacts, but as **data-intensive, memory-hungry, storage-stressing system workloads**.

---

## 1. Memory and Storage Demands of Full LLM Training

Let’s say you're training a 10B parameter LLM using FP16 precision:

| Component              | Memory Usage |
|------------------------|--------------|
| Model Weights          | 20 GB        |
| Gradients              | 20 GB        |
| Optimizer State (Adam) | 60 GB        |
| **Total per GPU**      | **100 GB**   |

This means even an A100 80 GB GPU won't be enough without optimization. Multiply across multiple GPUs (in data-parallel mode), and you have **100s of GBs of aggregate memory pressure**.

Even inference workloads must load massive models from disk and cache activations and KV state in GPU memory — placing demands on **latency, IO parallelism, and cache design**.

---

## 2. Fine-Tuning: The Scaled-Down, Still-Heavy Variant

Fine-tuning is a more targeted form of training, used to adapt general-purpose LLMs to domain-specific use cases.

### ✅ Parameter-Efficient Fine-Tuning (PEFT)

- **LoRA (Low-Rank Adaptation)**: Update only small low-rank matrices, saving 10–100x memory.
- **QLoRA**: Store base model weights in 4-bit precision + use LoRA adapters = fine-tune 65B models on a 24 GB GPU.

PEFT reduces GPU memory pressure **without increasing storage IO**, making it more accessible.

---

## 3. Storage Bottleneck #1: Checkpointing

During training (or fine-tuning), models are periodically saved to disk:

| Checkpoint Contents      | Size Estimate (10B model, FP16) |
|--------------------------|----------------------------------|
| Weights                  | 20 GB                           |
| Optimizer state          | 60 GB                           |
| (Optional) Gradients     | 20 GB                           |
| **Total per checkpoint** | **~80–100 GB**                 |

### 💥 Storage Workload Characteristics:

- **Sequential write-heavy**, bursty I/O
- Frequent **fsync() or flushes**
- High **write IOPS** demand if using local SSD
- High **latency** if using network FS (e.g., NFS, S3)

---

## 6. Optimizer Offloading

Optimizers like Adam store **3× parameter state**. When GPU memory is tight:

| Offload Target | IO Pattern         | Trade-off                               |
|----------------|--------------------|------------------------------------------|
| CPU RAM        | Moderate bandwidth | Increased latency                        |
| NVMe SSD       | **High R/W IOPS**  | Offloads memory but adds storage stress  |

Frameworks like **DeepSpeed ZeRO Stage 2/3** offload optimizer and even model weights to disk. This converts LLM training into a **storage-bound workload**.

---

## 📚 References

- [LoRA: Low-Rank Adaptation of Large Language Models (arXiv)](https://arxiv.org/abs/2106.09685)
- [QLoRA: Efficient Finetuning of Quantized LLMs (arXiv)](https://arxiv.org/abs/2305.14314)
- [DeepSpeed ZeRO: Memory Optimization Techniques (arXiv)](https://arxiv.org/abs/1910.02054)
- [NVIDIA GPUDirect Storage Overview](https://developer.nvidia.com/blog/gpudirect-storage/)
- [Hugging Face Blog: QLoRA Overview](https://huggingface.co/blog/qlora)
- [OpenAI GPT-4 Technical Report](https://openai.com/research/gpt-4)
- [DeepSpeed Library](https://www.deepspeed.ai/)
- [PyTorch FSDP (Fully Sharded Data Parallel)](https://pytorch.org/blog/introducing-pytorch-fully-sharded-data-parallel-api/)
- [Hugging Face Accelerate + PEFT Toolkit](https://github.com/huggingface/peft)

---

**Working with LLMs isn't just about GPUs and matrix math — it's about _data layout, memory hierarchy, and IO path design_.**
