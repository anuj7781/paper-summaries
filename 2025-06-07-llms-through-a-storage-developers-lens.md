# LLMs Through a Storage Developer’s Lens

As a system software or storage developer, you're probably used to thinking about throughput, caching, IOPS, and memory pressure. What you may not realize is that these concerns apply directly to working with **Large Language Models (LLMs)**. This includes training, fine-tuning, and even inference.

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

Even **inference** with LLMs creates significant system load:

- The full model must be **loaded from disk** before serving requests.
- During execution, **activations and key-value (KV) cache** must be stored in **GPU memory**.

This places heavy demands on:
- **Storage latency** (for fast model loading)
- **I/O parallelism** (to avoid GPU stalls)
- **Efficient memory/cache management** (especially with long prompts)

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

## 4. Storage Bottleneck #2: Data Loading

Training datasets like The Pile or OpenWebText can be hundreds of GBs or even TBs. These datasets are often stored on disk in formats like `.jsonl`, `.txt`, or `webdataset` tar archives.

### ⚙️ How it works:
- Data is read from disk → decoded → tokenized → batched.
- Modern pipelines cache data in **CPU RAM** or use **streaming prefetch** to avoid GPU starvation.

### 💾 Storage workload characteristics:
- **Random reads** (depending on shuffle strategy).
- Throughput-sensitive to avoid GPU underutilization.
- Benefits from **parallel I/O** and **prefetching** (e.g., `DataLoader(num_workers=N)`).

---

## 5. Optimizer Offloading

### 💡 Optimizer Offloading Explained

Optimizers like **Adam** maintain additional state per model parameter — typically:
- The weights  
- First moment (momentum)  
- Second moment (variance)  

This results in **3× memory usage per parameter**.

➡️ For example, a 10B parameter model in FP16:
- 20 GB for weights  
- **60 GB for optimizer state** (momentum + variance)  
- Total = **80+ GB per GPU**

When **GPU memory is insufficient**, this state is **offloaded** to:
- **CPU RAM**, or  
- **NVMe SSD** via frameworks like **DeepSpeed ZeRO** or **FSDP**

✅ This reduces **GPU memory usage**  
❌ But adds **PCIe and memory bandwidth pressure**

Frameworks like **DeepSpeed ZeRO Stage 2/3** offload optimizer and even model weights to disk. This converts LLM training into a **storage-bound workload**.

### ⚙️ Optimizer Offloading: Workload Characteristics

While optimizer offloading helps reduce GPU memory usage, it introduces I/O and bandwidth pressure elsewhere:

| Characteristic         | Description                                                                 |
|------------------------|-----------------------------------------------------------------------------|
| **Access Pattern**     | Frequent, small reads and writes (per backward pass)                        |
| **I/O Granularity**    | Fine-grained — often per layer or parameter group                           |
| **Frequency**          | Every training step (during gradient update)                                |
| **Latency Sensitivity**| Moderate to High — slow access can stall the training loop                  |
| **Bandwidth Usage**    | High PCIe / NVMe bandwidth if offloaded to SSD                              |
| **Storage Medium**     | CPU RAM (preferred), or NVMe SSD via frameworks like ZeRO or DeepSpeed      |
| **Caching Strategy**   | LRU cache in RAM; smart partitioning/sharding to reduce I/O pressure        |


---
### 🧠 KV Cache (Key-Value Cache): Workload Characteristics

In autoregressive LLM inference, each new token needs to attend to all previous tokens. To avoid recomputing attention for the entire sequence repeatedly, models use a **KV cache** — storing key and value vectors layer-wise for every token generated.

| Characteristic         | Description                                                                 |
|------------------------|-----------------------------------------------------------------------------|
| **Access Pattern**     | Repeated in-memory lookups and appends (per token generated)                |
| **I/O Granularity**    | Fine-grained — per token, per attention head, per layer                     |
| **Frequency**          | Every inference step (each new token)                                       |
| **Latency Sensitivity**| Very High — any delay directly increases token generation latency           |
| **Bandwidth Usage**    | High GPU HBM bandwidth (all attention heads access cache in parallel)       |
| **Storage Medium**     | GPU HBM (or shared memory in multi-query models)                            |
| **Caching Strategy**   | Append-only, fast indexed access; paged memory for long prompts (if needed) |

> 📌 Note: KV Cache is typically not stored on disk — it creates pressure on GPU memory, not persistent storage.

---
# 6. Summary

| Feature                  | Model/Data Loading                            | Checkpointing                            | Optimizer Offloading                                      | KV Cache (Inference)                             |
|--------------------------|-----------------------------------------------|-------------------------------------------|------------------------------------------------------------|---------------------------------------------------|
| **Primary Purpose**      | Load training data and model weights into memory | Persist training state for fault recovery | Free up GPU memory by moving optimizer state off-device    | Cache keys/values for efficient autoregressive inference |
| **When It Happens**      | At start (model); every step (data batch)     | Every few minutes / epochs                | Every backward pass                                        | Every token generation step                       |
| **Data Moved/Stored**    | Model weights, training batches               | Weights, optimizer state, gradients       | Momentum, variance (e.g. for Adam)                         | Key/Value vectors for each past token and layer   |
| **Storage Location**     | NVMe / NFS / S3 → RAM / GPU                   | Disk (NVMe, NFS, S3)                      | CPU RAM or NVMe                                            | GPU HBM                                            |
| **I/O Characteristics**  | Sustained read, random access (data); sequential read (model) | Sequential, bursty writes     | Frequent small R/W                                         | In-memory access only                             |
| **Optimization Methods** | Dataloader prefetch, caching, model sharding  | Sharding, async writes                    | ZeRO, DeepSpeed, FSDP                                      | KV cache quantization, sliding window attention   |
| **Storage Bottleneck**   | Metadata lookup, read throughput              | High write throughput, fsync overhead     | PCIe/memory bandwidth, SSD latency                         | GPU HBM memory pressure, prompt length scaling     |

---

## 📚 References


---

**Working with LLMs isn't just about GPUs and matrix math — it's about _data layout, memory hierarchy, and IO path design_.**
