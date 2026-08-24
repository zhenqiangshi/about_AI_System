

`mkdir -p ./bench_qwen36

for conc in 1 2 4 8 16 24 32 40 50; do
  echo "========================================"
  echo "正在测试 max-concurrency = $conc"
  echo "========================================"

  vllm bench serve \
    --backend openai-chat \
    --host 127.0.0.1 \
    --port 9997 \
    --endpoint /v1/chat/completions \
    --model qwen3.6-35B \
    --tokenizer Qwen/Qwen2.5-32B-Instruct \     #使用Xinference部署，名称已经发生变化
    --num-prompts 150 \      #**总任务量**：整个测试过程最终要完成 150 个请求的收发。
    --max-concurrency $conc \  # **并发窗口**：**同时**处于“发送中/等待中”状态的请求数量上限。
    --random-input-len 512 \
    --random-output-len 128 \
    --num-warmups 8 \
    --save-result \
    --result-dir ./bench_qwen36 \
    --result-filename "conc_${conc}.json"
done`



推荐补充测试（输入长度对比）
```
for in_len in 256 512 1024 2048 4096; do
  echo "===== Testing input_len = $in_len ====="
  vllm bench serve \
    --backend openai-chat \
    --host 127.0.0.1 \
    --port 9997 \
    --endpoint /v1/chat/completions \
    --model qwen3.6-35B \
    --tokenizer Qwen/Qwen2.5-32B-Instruct \
    --num-prompts 100 \
    --max-concurrency 8 \
    --random-input-len $in_len \
    --random-output-len 128 \
    --num-warmups 5 \
    --save-result \
    --result-dir ./bench_qwen36_input \
    --result-filename "input_${in_len}.json"
done
```





结果1：


============ Serving Benchmark Result ============
Successful requests:                     100       
Failed requests:                         0         
Maximum request concurrency:             24        
Benchmark duration (s):                  103.40    
Total input tokens:                      453738    
Total generated tokens:                  11618     
Request throughput (req/s):              0.97      
Output token throughput (tok/s):         112.36    
Peak output token throughput (tok/s):    587.00    
Peak concurrent requests:                26.00     
Total token throughput (tok/s):          4500.54   
---------------Time to First Token----------------
Mean TTFT (ms):                          7535.65   
Median TTFT (ms):                        6633.45   
P99 TTFT (ms):                           21107.43    ##**尾部延迟**
-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                          146.79    
Median TPOT (ms):                        157.82    
P99 TPOT (ms):                           204.17    
---------------Inter-token Latency----------------
Mean ITL (ms):                           144.42    
Median ITL (ms):                         202.40    

P99 ITL (ms):                            229.48

==================================================



可修改不同的vllm参数进行权衡prefill（算）与decode（存）。

max_model_len
enable_prefix_caching
enable_chunked_prefill



对 vLLM / Xinference 推理服务进行参数优化，是一个在**吞吐量（Throughput）** 和**延迟（Latency）** 之间寻找平衡的迭代过程。没有一劳永逸的“最优配置”，最佳参数取决于你的**模型、硬件、输入数据特点以及业务场景**（是在线交互还是离线批处理）。

以下是一个系统性的参数优化步骤指南，你可以参考这个流程进行操作。

### 🗺️ 步骤一：明确目标与基准测试

在调整任何参数之前，先建立明确的优化目标和性能基准。

1.  **定义业务目标 (SLA)**：
    *   **在线服务 (交互式)**：首要目标是**低延迟**。应重点关注 **TTFT (Time to First Token，首字延迟)** 和 **TPOT (Time Per Output Token，每个输出token的时间)**。例如，你可能要求 TTFT P95 < 200ms。
    *   **离线批处理 (非交互式)**：首要目标是**高吞吐量**。可以容忍较高的延迟，目标是最大化总Token吞吐量（tokens/s）。
2.  **建立性能基准**：在默认配置下运行你的测试负载（如使用 `benchmark_serving.py`），记录下关键的吞吐量和延迟指标作为基线，用于后续对比。

### ⚙️ 步骤二：核心参数调优（迭代进行）

以下参数是优化的核心，**建议按顺序逐个调整，每次调整后都进行测试和观察**，切忌“一步到位”同时修改多个参数。

#### 1. 显存与内存管理
这是确保服务稳定运行的基础，应优先配置。
*   **`gpu_memory_utilization`**:
    *   **作用**：控制vLLM能使用的GPU显存比例。
    *   **调优**：从保守值 **0.85** 开始，逐步增加到 **0.90** 或 **0.95**。你需要为系统和其他进程预留一些显存，防止内存溢出（OOM）。
*   **`max_model_len`**:
    *   **作用**：限制模型能处理的最大序列长度（输入+输出）。
    *   **调优**：**不要**直接设置为模型支持的最大值（如32K）。应根据你的业务数据，设置为一个合理的上限（例如，大部分请求的输入长度在4K左右，可设为8K）。较小的值可以减少KV Cache的占用，允许更大的批处理。

#### 2. 批处理与并发控制
这是平衡吞吐量和延迟的关键。
*   **`max_num_seqs` (关键参数)**:
    *   **作用**：单个批次（batch）内能同时处理的最大请求数。
    *   **调优**：这是影响最大的参数之一。**调大**可提升吞吐量，但会增加延迟（尤其是TTFT）；**调小**则反之。
    *   **建议策略**：从**保守值（如32）** 开始。然后以**小步长**（如每次增加8或16）逐步增加，同时密切监控**TTFT P99**延迟，直到该延迟达到你的SLA阈值为止。
*   **`max_num_batched_tokens`**:
    *   **作用**：限制单个批次内所有请求的**总token数**。
    *   **调优**：
        *   这个值通常与 `max_num_seqs` 和平均输入长度联动。如果追求**高吞吐量**，官方建议设为 **> 8192**。
        *   如果**对延迟敏感**，可以尝试较小的值（如 **2048**），这有助于改善ITL。
        *   如果启用了 `enable_chunked_prefill`，其默认值可能为 **512** 或 **2048**，需根据实际情况调整。
*   **`max_concurrency`**:
    *   **作用**：在Xinference等工具中，这个参数用于限制整个服务端的最大并发请求数。
    *   **调优**：可以将其设置为一个略高于你预期峰值并发的值。

#### 3. 高级特性与优化选项
在基础参数调优完成后，可以尝试开启一些高级特性来进一步提升性能。
*   **`enable_chunked_prefill` (分块预填充)**:
    *   **作用**：将长输入的预填充（Prefill）过程分成小块，与解码（Decode）请求混合处理，避免长输入请求长时间占用GPU。
    *   **调优**：**强烈建议开启**，尤其是在处理变长输入时。在Xinference中默认是关闭的。开启后，通常能带来 **15-25%** 的性能提升。
*   **`enable_prefix_caching` (前缀缓存)**:
    *   **作用**：缓存相同前缀（如系统提示词）的KV Cache，供后续请求复用。
    *   **调优**：**建议开启**。如果你们的prompt有大量共享前缀（如固定的系统指令），这个功能可以“零成本”加速。在Xinference中可能默认开启。

### 📊 步骤三：监控、验证与迭代

参数调优不是一次性的工作，而是一个持续的循环。

1.  **监控关键指标**：
    *   **系统层**：使用 `nvidia-smi` 监控**GPU显存使用率**和**利用率**。显存使用率建议保持在70%-85%之间。
    *   **应用层**：重点关注 **TTFT (P99)**、**TPOT (P99)**、**吞吐量 (tokens/s)** 以及日志中的 **`preempted`（抢占）次数**。
2.  **分析抢占 (Preemption)**：
    *   如果在日志中看到 `Sequence group ... is preempted` 的警告，说明KV Cache空间不足。
    *   **解决方案**：可以尝试**增加 `gpu_memory_utilization`**、**减少 `max_num_seqs`** 或 **`max_num_batched_tokens`**。
3.  **迭代调整**：
    *   根据监控数据，微调参数。例如，如果显存还有余量，可以尝试略微提高 `max_num_seqs` 或 `max_num_batched_tokens` 来提升吞吐。
    *   如果发现TTFT过高，则需适当降低这两个值。

### ⚠️ 步骤四：故障排查与高级选项

如果遇到性能瓶颈或启动问题，可以尝试以下方法。

*   **启动与调试**：
    *   **`--enforce-eager`**：用于**调试和开发**。它会禁用CUDA Graphs等优化，大幅降低性能，但能提供更详细的错误信息，启动也更快。
    *   **优化级别 (`-O0` 到 `-O3`)**：vLLM提供了4个优化级别。`-O2` 是生产环境的默认推荐，它平衡了启动速度和运行性能。`-O0` 启动最快但性能最低。
*   **分布式部署**：
    *   **`tensor_parallel_size`**：将模型权重切分到多张GPU上。这可以为KV Cache腾出更多显存。
    *   **`pipeline_parallel_size`**：将模型层切分到多张GPU上，也可以间接增加KV Cache可用内存。

### 💎 总结

参数优化的核心思路是：**明确目标 -> 基准测试 -> 迭代调参 -> 持续监控**。

1.  **从保守配置开始**，如：`gpu_memory_utilization=0.85`, `max_num_seqs=32`, `max_model_len=2048`。
2.  使用你的**真实负载**进行测试。
3.  根据监控数据，**小步快跑**，逐步调整 `max_num_seqs` 和 `max_num_batched_tokens`。
4.  开启 **`enable_chunked_prefill`** 和 **`enable_prefix_caching`** 等高级特性。
5.  密切关注 **`preempted`** 警告，它是系统资源不足的重要信号。

每次调整后，记录下配置和对应的性能数据，形成你自己的调优“手册”。