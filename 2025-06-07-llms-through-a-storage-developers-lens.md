# LLMs Through a Storage Developer’s Lens

As a system software or storage developer, you're probably used to thinking about throughput, caching, IOPS, and memory pressure. What you may not realize is that these concerns apply directly to working with **Large Language Models (LLMs)**. This includes training, fine-tuning, and even inference.

In this blog, we look at LLMs not just as machine learning artifacts, but as **data-intensive, memory-hungry, storage-stressing system workloads**.

---

## 1. Memory and Storage Demands of Full LLM Training

Let’s say you're training a 10 Billion parameters LLM using FP16 precision:

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

## 5. Bottleneck #3: Optimizer Offloading

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
### 🧠 Bottleneck #4: KV Cache (Key-Value Cache)

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
## 🧪 Next Steps: Emulating LLM Storage Pressure Without GPUs

(Note : I need to cross check the sources and proposed benchmarks myself)

You don’t need GPUs to understand where LLM workloads stress the storage stack. The following open-source tools can simulate key I/O and memory behaviors relevant to model training and inference:

### 🔸 [Fugaku LLM I/O Benchmark](https://github.com/riken-rccs/fugaku-llm-benchmark)
- Developed by RIKEN for exascale research.
- Emulates checkpointing, parameter sharding, and optimizer offload patterns for LLM-scale models.
- Useful for profiling **write bursts**, **fsync behavior**, and **I/O scalability** on NVMe or NFS setups.

### 🔸 [DeepIO](https://github.com/alibaba/DeepIO)
- Built by Alibaba Cloud for large-scale recommendation and LLM workloads.
- Profiles **training data ingestion**, **shuffling**, and **dataloader cache behavior**.
- Supports plug-and-play deployment on CPU-only clusters.

## 🤔 Why Not MLPerf for Storage Testing?

(Note : I need to cross check this part)

[MLPerf](https://mlcommons.org/en/) is the industry standard for benchmarking ML performance. However, it may not be the best tool for analyzing storage and memory bottlenecks in a CPU-only or dev test environment.

Here’s why:

| Limitation                  | Explanation                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| **GPU-Centric**            | Designed for benchmarking with GPUs like A100s or TPUs — not CPU-only setups. |
| **Heavy Dependencies**     | Requires CUDA, NCCL, cuDNN, and other vendor-optimized libraries.           |
| **Not Modular for Storage**| Focuses on overall throughput, not isolating disk/memory impact.            |
| **High Resource Demand**   | Even a single BERT benchmark requires 100+ GB RAM, fast SSD, and multiple GPUs. |
| **Storage Benchmark Immature** | MLPerf Storage is still evolving and not widely adopted.                     |

### ✅ What You *Can* Do
- Study the [MLPerf benchmark specs](https://github.com/mlcommons/training) to understand typical I/O behavior.
- Use `strace`, `blktrace`, or `perf` to trace I/O on CPU-mode runs.
- Reproduce checkpoint/data ingestion I/O using `fio` or tools like [Fugaku](https://github.com/riken-rccs/fugaku-llm-benchmark) and [DeepIO](https://github.com/alibaba/DeepIO).

> MLPerf is best as a **reference** or **target workload**, but not a direct test tool for isolated storage benchmarking — yet.

---

## 📚 Key Reads on AI Storage and Memory Bottlenecks

A good next step good be finding blogs/papers that talk about memory/storage problems for LLMs. I found some, but need to go through these.

### 1. [AI and Memory Wall – Amir Gholami et al.](https://arxiv.org/abs/2403.14123)
- **Summary**: This paper discusses how memory bandwidth has become a primary bottleneck in AI applications, especially for transformer models. It highlights the disparity between the rapid growth of compute capabilities and the slower advancement of memory technologies, leading to a "memory wall" that hampers performance.

### 2. [Data Movement Bottlenecks to Large-Scale Model Training – Epoch AI](https://epoch.ai/blog/data-movement-bottlenecks-scaling-past-1e28-flop)
- **Summary**: This blog post analyzes the challenges of data movement in large-scale model training. It identifies intra-GPU and inter-GPU data transfers as significant bottlenecks and discusses the concept of a "latency wall" that limits scaling.

### 3. [Understanding Data Storage and Ingestion for Large-Scale Deep Recommendation Model Training – Mark Zhao et al.](https://arxiv.org/abs/2108.09373)
- **Summary**: This paper presents Meta's data storage and ingestion pipeline for training deep recommendation models. It details the architecture, challenges, and optimizations involved in handling vast amounts of training data efficiently.

### 4. [Stop Wasting Money on AI Storage – Lightbits Labs](https://www.lightbitslabs.com/blog/stop-wasting-money-on-ai-storage-a-smarter-leaner-approach/)
- **Summary**: This blog critiques the use of traditional parallel file systems for AI training workloads. It argues that such systems, while powerful, may be overkill and not cost-effective for AI applications, suggesting alternative storage strategies.

### 5. [Solving AI's Bottlenecks – IQT](https://www.iqt.org/library/solving-ais-bottlenecks---the-critical-role-of-physical-and-software-layers)
- **Summary**: This article discusses the challenges in AI workloads related to storage and memory. It emphasizes the need for advancements in memory technologies and software optimizations to overcome current bottlenecks.


---

**Working with LLMs isn't just about GPUs and matrix math — it's about _data layout, memory hierarchy, and IO path design_.**
