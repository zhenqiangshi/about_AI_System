
**!!! warning**
    **本文为基于 [vLLM 原始论文](https://arxiv.org/abs/2309.06180) 的历史文档，**
    **已不再描述当前 vLLM 中使用的代码。**

目前，vLLM 使用自研的多头查询注意力内核（`csrc/attention/attention_kernels.cu`）。
该内核设计为兼容 vLLM 的分页 KV 缓存，其中 key 与 value 缓存分别存储在独立的块中
（注意：此处的“块”概念不同于 GPU 线程块。后续文档中，我将把 vLLM 分页注意力块称为“块”，
而把 GPU 线程块称为“线程块”）。

为实现高性能，该内核依赖专门设计的内存布局与访问方式，尤其是在线程从全局内存读取数据到共享内存时。
本文旨在分步提供内核实现的高层解释，帮助希望了解 vLLM 多头查询注意力内核的读者。
阅读本文后，读者应能更好地理解实际实现，并更容易跟进代码。

请注意，本文可能无法覆盖所有细节，例如如何计算对应数据的正确索引，或点乘的具体实现。
但在阅读本文并熟悉高层逻辑流程后，阅读实际代码并理解细节应会更容易。

## 输入

内核函数接收一系列参数，供当前线程执行其分配的工作。其中最重要的三个参数是输入指针
`q`、`k_cache` 和 `v_cache`，它们指向全局内存中需要读取和处理的 query、key 与 value 数据。
输出指针 `out` 指向应写入结果的全局内存。这四个指针实际指向多维数组，但每个线程只访问分配给它的数据部分。
为简洁起见，此处省略了所有其他运行时参数。

```cpp
template<typename scalar_t, int HEAD_SIZE, int BLOCK_SIZE, int NUM_THREADS, int PARTITION_SIZE = 0>
__device__ void paged_attention_kernel(
    ... // 其他辅助参数
    const scalar_t* __restrict__ out, // [num_seqs, num_heads, max_num_partitions, head_size]
    const scalar_t* __restrict__ q, // [num_seqs, num_heads, head_size]
    const scalar_t* __restrict__ k_cache, // [num_blocks, num_kv_heads, head_size/x, block_size, x]
    const scalar_t* __restrict__ v_cache, // [num_blocks, num_kv_heads, head_size, block_size]
    ... // 其他辅助参数
)
```

函数签名上方还有一系列模板参数，它们在编译时确定。`scalar_t` 表示 query、key、value 数据元素的数据类型，例如 FP16。
`HEAD_SIZE` 表示每个头的元素数量。`BLOCK_SIZE` 表示每个块中的 token 数量。
`NUM_THREADS` 表示每个线程块中的线程数。`PARTITION_SIZE` 表示张量并行 GPU 的数量
（为简化，我们假设其为 0，即禁用张量并行）。

有了这些参数后，需要进行一系列准备工作，包括计算当前头索引、块索引以及其他必要变量。
但目前我们可以先忽略这些准备，直接进入实际计算。在掌握整个流程后，理解这些准备会更容易。

## 概念

在深入计算流程之前，先介绍后续章节会用到的几个概念。如果你在阅读中遇到不熟悉的术语，可先跳过本节，稍后再回来查阅。

- **序列（Sequence）**：序列代表一个客户端请求。例如，`q` 指向的数据形状为
  `[num_seqs, num_heads, head_size]`，表示共有 `num_seqs` 个 query 序列数据。
  由于本内核是单 query 注意力内核，每个序列只有一个 query token，因此 `num_seqs` 等于批次中处理的 token 总数。

- **上下文（Context）**：上下文由序列已生成的 token 组成。例如，`["What", "is", "your"]` 是上下文 token，
  输入的 query token 是 `"name"`，模型可能生成 token `"?"`。

- **向量（Vec）**：向量是一起获取和计算的元素列表。对于 query 和 key 数据，向量大小（`VEC_SIZE`）
  被设定为使每个线程组一次可获取并计算 16 字节数据。对于 value 数据，向量大小（`V_VEC_SIZE`）
  被设定为使每个线程一次可获取并计算 16 字节数据。例如，若 `scalar_t` 为 FP16（2 字节）且
  `THREAD_GROUP_SIZE` 为 2，则 `VEC_SIZE` 为 4，而 `V_VEC_SIZE` 为 8。

- **线程组（Thread group）**：线程组是一小群线程（`THREAD_GROUP_SIZE`），一次获取并计算一个 query token 和一个 key token。
  每个线程只处理 token 数据的一部分。一个线程组处理的元素总数称为 `x`。例如，若线程组包含 2 个线程且头大小为 8，
  则线程 0 处理索引 0、2、4、6 的 query 和 key 元素，线程 1 处理索引 1、3、5、7 的元素。

- **块（Block）**：vLLM 中的 key 和 value 缓存数据被分割成块。每个块存储一个头上固定数量（`BLOCK_SIZE`）的 token 数据。
  每个块可能只包含全部上下文 token 的一部分。例如，若块大小为 16、头大小为 128，则一个头上一个块可存储
  16 * 128 = 2048 个元素。

- **Warp**：Warp 是一组同时在流多处理器（SM）上执行的 32 个线程（`WARP_SIZE`）。
  在本内核中，每个 warp 一次处理一个 query token 与一个完整块的 key token 的计算（可能在多次迭代中处理多个块）。
  例如，若有 4 个 warp 和 6 个上下文块，则分配方式为：warp 0 处理第 0、4 块，warp 1 处理第 1、5 块，
  warp 2 处理第 2 块，warp 3 处理第 3 块。

- **线程块（Thread block）**：线程块是一组可访问同一共享内存的线程（`NUM_THREADS`）。
  每个线程块包含多个 warp（`NUM_WARPS`），在本内核中，每个线程块处理一个 query token 与整个上下文的 key token 的计算。

- **网格（Grid）**：网格是线程块的集合，并定义了该集合的形状。在本内核中，形状为
  `(num_heads, num_seqs, max_num_partitions)`。因此，每个线程块只处理一个头、一个序列和一个分区的计算。

## Query

本节介绍 query 数据在内存中的存储方式以及每个线程如何获取。如前所述，每个线程组获取一个 query token 数据，
而每个线程本身只处理该 token 数据的一部分。在每个 warp 内，所有线程组会获取相同的 query token 数据，
但会与不同的 key token 数据进行相乘。

```cpp
const scalar_t* q_ptr = q + seq_idx * q_stride + head_idx * HEAD_SIZE;
```

![[../Images/Pasted image 20260821110423.png]]

每个线程定义自己的 `q_ptr`，指向全局内存中分配给它的 query token 数据。例如，若 `VEC_SIZE` 为 4 且
`HEAD_SIZE` 为 128，则 `q_ptr` 指向包含共 128 个元素的数据，这些元素被划分为 128 / 4 = 32 个向量。![[../Images/Pasted image 20260821110648.png]]
```cpp
__shared__ Q_vec q_vecs[THREAD_GROUP_SIZE][NUM_VECS_PER_THREAD];
```

接下来，需要将 `q_ptr` 指向的全局内存数据读取到共享内存的 `q_vecs` 中。需要注意的是，每个向量被分配到不同的行。
例如，若 `THREAD_GROUP_SIZE` 为 2，线程 0 处理第 0 行的向量，线程 1 处理第 1 行的向量。
通过这种方式读取 query 数据，相邻线程（如线程 0 和线程 1）可以读取相邻内存，从而实现内存合并访问以提升性能。

## Key

与 “Query” 部分类似，本节介绍 key 的内存布局与分配。每个线程组在一次内核运行中只处理一个 query token，
但可能在多次迭代中处理多个 key token。同时，每个 warp 会在多次迭代中处理多个 key token 块，
确保整个线程组在内核运行结束后处理完所有上下文 token。此处的“处理”指对 query 数据与 key 数据执行点乘。

```cpp
const scalar_t* k_ptr = k_cache + physical_block_number * kv_block_stride
                    + kv_head_idx * kv_head_stride
                    + physical_block_offset * x;
```

与 `q_ptr` 不同，每个线程中的 `k_ptr` 会在不同迭代中指向不同的 key token。如上所示，`k_ptr`
根据分配的块、头和 token，指向 `k_cache` 中的 key token 数据。
![[../Images/Pasted image 20260821110726.png]]
上图展示了 key 数据的内存布局。假设 `BLOCK_SIZE` 为 16，`HEAD_SIZE` 为 128，`x` 为 8，
`THREAD_GROUP_SIZE` 为 2，共有 4 个 warp。每个矩形代表一个头上一个 key token 的全部元素，
由一个线程组处理。左半部分显示 warp 0 的共 16 个 key token 数据块，右半部分表示其他 warp 或迭代的剩余 key token 数据。
每个矩形内部共有 32 个向量（一个 token 的 128 个元素），由 2 个线程（一个线程组）分别处理。

![[../Images/Pasted image 20260821110748.png]]

```cpp
K_vec k_vecs[NUM_VECS_PER_THREAD]
```

接下来，需要从 `k_ptr` 读取 key token 数据，并存储到寄存器内存的 `k_vecs` 中。
我们使用寄存器内存存放 `k_vecs`，因为它只会被一个线程访问一次，而 `q_vecs` 会被多个线程多次访问。
每个 `k_vecs` 包含多个向量，供后续计算使用。每个向量在每次内层迭代中设置。
向量的分配方式使 warp 中的相邻线程能够一起读取相邻内存，再次促进内存合并访问。
例如，线程 0 读取向量 0，线程 1 读取向量 1；在下一次内层循环中，线程 0 读取向量 2，线程 1 读取向量 3，以此类推。

你可能对整体流程仍有些困惑。别担心，请继续阅读下一节 “QK”。它将以更清晰、更高层的方式说明 query 与 key 的计算流程。

## QK

如下伪代码所示，在整个 for 循环块之前，我们获取一个 token 的 query 数据并存储到 `q_vecs`。
然后在外层 for 循环中，我们迭代指向不同 token 的不同 `k_ptr`，并在内层 for 循环中准备 `k_vecs`。
最后，对 `q_vecs` 与每个 `k_vecs` 执行点乘。

```cpp
q_vecs = ...
for ... {
    k_ptr = ...
    for ... {
        k_vecs[i] = ...
    }
    ...
    float qk = scale * Qk_dot<scalar_t, THREAD_GROUP_SIZE>::dot(q_vecs[thread_group_offset], k_vecs);
}
```

如前所述，每个线程一次只获取部分 query 和 key token 数据。但在 `Qk_dot<>::dot` 中会发生跨线程组归约。
因此，此处返回的 `qk` 并非仅是部分 query 与 key token 的点乘结果，而是整个 query 与 key token 数据的完整结果。

例如，若 `HEAD_SIZE` 为 128 且 `THREAD_GROUP_SIZE` 为 2，每个线程的 `k_vecs` 将包含共 64 个元素。
但返回的 `qk` 实际是 128 个 query 元素与 128 个 key 元素的点乘结果。若想了解点乘与归约的更多细节，
可参考 `Qk_dot<>::dot` 的实现。但为简洁起见，本文不展开讨论。

## Softmax

接下来，需要对所有 `qk` 计算归一化 softmax，如下所示，其中每个 $x$ 代表一个 `qk`。
为此，必须获得所有 `qk` 的归约值 `qk_max`（$m(x)$）和 `exp_sum`（$\ell(x)$）。
归约应在整个线程块上执行，涵盖 query token 与所有上下文 key token 之间的结果。

$$
\begin{gather*}
m(x):=\max _i \quad x_i \\ \quad f(x):=\left[\begin{array}{lll}e^{x_1-m(x)} & \ldots & e^{x_B-m(x)}\end{array}\right]\\ \quad \ell(x):=\sum_i f(x)_i \\
\quad \operatorname{softmax}(x):=\frac{f(x)}{\ell(x)}
\end{gather*}
$$

### `qk_max` 与 `logits`

在得到 `qk` 结果后，即可用 `qk` 设置临时的 `logits` 结果（最终 `logits` 应存储归一化 softmax 结果）。
同时，可比较并收集当前线程组计算出的所有 `qk` 的 `qk_max`。

```cpp
if (thread_group_offset == 0) {
    const bool mask = token_idx >= context_len;
    logits[token_idx - start_token_idx] = mask ? 0.f : qk;
    qk_max = mask ? qk_max : fmaxf(qk_max, qk);
}
```

请注意，此处的 `logits` 位于共享内存，因此每个线程组会为其分配的上下文 token 设置字段。
总体而言，`logits` 的大小应等于上下文 token 的数量。

```cpp
for (int mask = WARP_SIZE / 2; mask >= THREAD_GROUP_SIZE; mask /= 2) {
    qk_max = fmaxf(qk_max, VLLM_SHFL_XOR_SYNC(qk_max, mask));
}
if (lane == 0) {
    red_smem[warp_idx] = qk_max;
}
```

然后需要在每个 warp 内获取归约后的 `qk_max`。主要思路是让 warp 内的线程相互通信，得到最终的最大 `qk`。

```cpp
for (int mask = NUM_WARPS / 2; mask >= 1; mask /= 2) {
    qk_max = fmaxf(qk_max, VLLM_SHFL_XOR_SYNC(qk_max, mask));
}
qk_max = VLLM_SHFL_SYNC(qk_max, 0);
```

最后，通过比较本线程块中所有 warp 的 `qk_max`，得到整个线程块的归约 `qk_max`，
然后将最终结果广播给每个线程。

### `exp_sum`

与 `qk_max` 类似，我们也需要从整个线程块获取归约后的求和值。

```cpp
for (int i = thread_idx; i < num_tokens; i += NUM_THREADS) {
    float val = __expf(logits[i] - qk_max);
    logits[i] = val;
    exp_sum += val;
}
...
exp_sum = block_sum<NUM_WARPS>(&red_smem[NUM_WARPS], exp_sum);
```

首先，对每个线程组的所有 exp 值求和，同时将 `logits` 中的每个条目从 `qk` 转换为 `exp(qk - qk_max)`。
请注意，此处的 `qk_max` 已是整个线程块的最大 `qk`。然后，像处理 `qk_max` 一样，对整个线程块进行 `exp_sum` 的归约。

```cpp
const float inv_sum = __fdividef(1.f, exp_sum + 1e-6f);
for (int i = thread_idx; i < num_tokens; i += NUM_THREADS) {
    logits[i] *= inv_sum;
}
```

最终，利用归约后的 `qk_max` 和 `exp_sum`，即可得到最终的归一化 softmax 结果，存入 `logits`。
该 `logits` 变量将在后续步骤中与 value 数据执行点乘。此时，它应存储所有分配的上下文 token 的 `qk` 归一化 softmax 结果。

## Value

![[../Images/Pasted image 20260821111846.png]]
![[../Images/Pasted image 20260821111855.png]]
![[../Images/Pasted image 20260821111912.png]]
现在需要获取 value 数据，并与 `logits` 执行点乘。与 query 和 key 不同，value 数据没有线程组概念。
如图所示，与 key token 的内存布局不同，同一列的元素对应同一个 value token。对于一个 value 数据块，
有 `HEAD_SIZE` 行和 `BLOCK_SIZE` 列，被分割为多个 `v_vec`。

每个线程一次总是从相同的 `V_VEC_SIZE` 个 token 中获取 `V_VEC_SIZE` 个元素。
因此，单个线程通过多次内层迭代，从不同行但相同列中检索多个 `v_vec`。
对于每个 `v_vec`，需要与对应的 `logits_vec`（同样来自 `logits` 的 `V_VEC_SIZE` 个元素）执行点乘。
总体而言，通过多次内层迭代，每个 warp 处理一个 value token 块；通过多次外层迭代，处理整个上下文的 value token。

```cpp
float accs[NUM_ROWS_PER_THREAD];
for ... { // 迭代不同块
    logits_vec = ...
    for ... { // 迭代不同行
        v_vec = ...
        ...
        accs[i] += dot(logits_vec, v_vec);
    }
}
```

如上伪代码所示，在外层循环中，类似于 `k_ptr`，`logits_vec` 迭代不同块，并从 `logits` 中读取
`V_VEC_SIZE` 个元素。在内层循环中，每个线程从相同 token 读取 `V_VEC_SIZE` 个元素作为 `v_vec`，并执行点乘。
需要注意的是，在每次内层迭代中，线程获取相同 token 的不同头位置元素。点乘结果随后累加到 `accs` 中。
因此，`accs` 的每个条目映射到当前线程分配的头位置。

例如，若 `BLOCK_SIZE` 为 16 且 `V_VEC_SIZE` 为 8，每个线程一次获取 8 个 token 的 8 个 value 元素。
每个元素来自不同 token 的相同头位置。若 `HEAD_SIZE` 为 128 且 `WARP_SIZE` 为 32，则每次内层循环中，
一个 warp 需要获取 `WARP_SIZE * V_VEC_SIZE = 256` 个元素。这意味着一个 warp 处理一个完整 value token 块需要
共 128 * 16 / 256 = 8 次内层迭代。每个线程的 `accs` 包含 8 个元素，分别累加了 8 个不同头位置的结果。
对于线程 0，`accs` 变量将有 8 个元素，分别是 value 头中第 0、32…224 个元素，这些元素来自所有分配的 8 个 token 的累加。

## LV

现在需要对每个 warp 内的 `accs` 执行归约。该过程使每个线程能够累加一个块中所有 token 在其分配头位置上的 `accs`。

```cpp
for (int i = 0; i < NUM_ROWS_PER_THREAD; i++) {
    float acc = accs[i];
    for (int mask = NUM_V_VECS_PER_ROW / 2; mask >= 1; mask /= 2) {
        acc += VLLM_SHFL_XOR_SYNC(acc, mask);
    }
    accs[i] = acc;
}
```

接下来，对所有 warp 执行 `accs` 的归约，使每个线程拥有所有上下文 token 在其分配头位置上的 `accs` 累加结果。
请注意，每个线程中的 `accs` 仅存储整个头中部分元素对所有上下文 token 的累加结果。
但总体而言，输出的所有结果都已计算完成，只是存储在不同线程的寄存器内存中。


    
    float* out_smem = reinterpret_cast<float*>(shared_mem);
    for (int i = NUM_WARPS; i > 1; i /= 2) {
        // 上层 warp 写入共享内存
        ...
        float* dst = &out_smem[(warp_idx - mid) * HEAD_SIZE];
        for (int i = 0; i < NUM_ROWS_PER_THREAD; i++) {
            ...
            dst[row_idx] = accs[i];
        }
        // 下层 warp 更新输出
        const float* src = &out_smem[warp_idx * HEAD_SIZE];
        for (int i = 0; i < NUM_ROWS_PER_THREAD; i++) {
            ...
            accs[i] += src[row_idx];
        }
        // 写出 accs
    }
   

## 输出

现在可以将所有计算结果从本地寄存器内存写入最终输出全局内存。

```cpp
scalar_t* out_ptr = out + seq_idx * num_heads * max_num_partitions * HEAD_SIZE
                + head_idx * max_num_partitions * HEAD_SIZE
                + partition_idx * HEAD_SIZE;
```

首先，需要定义 `out_ptr` 变量，它指向分配序列和分配头的起始地址。

```cpp
for (int i = 0; i < NUM_ROWS_PER_THREAD; i++) {
    const int row_idx = lane / NUM_V_VECS_PER_ROW + i * NUM_ROWS_PER_ITER;
    if (row_idx < HEAD_SIZE && lane % NUM_V_VECS_PER_ROW == 0) {
        from_float(*(out_ptr + row_idx), accs[i]);
    }
}
```

最后，需要迭代不同的分配头位置，并根据 `out_ptr` 写出对应的累加结果。

