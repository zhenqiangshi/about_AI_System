```markdown
# 自动前缀缓存

前缀缓存 KV 缓存块是 LLM 推理中常见的优化手段，用于避免重复计算提示。核心思路很简单——缓存已处理请求的 KV 缓存块，当新请求的前缀与之前请求相同时，直接复用这些块。由于前缀缓存几乎是“免费午餐”，且不会改变模型输出，它已被许多公开接口（如 OpenAI、Anthropic 等）以及大多数开源 LLM 推理框架（如 SGLang）广泛采用。

实现前缀缓存的方式有很多，vLLM 选择了基于哈希的方法。具体来说，我们根据块内的 token 以及该块之前的前缀 token 对每个 KV 缓存块进行哈希：

```text
                    Block 1                  Block 2                  Block 3
         [A gentle breeze stirred] [the leaves as children] [laughed in the distance]
Block 1: |<--- block tokens ---->|
Block 2: |<------- prefix ------>| |<--- block tokens --->|
Block 3: |<------------------ prefix -------------------->| |<--- block tokens ---->|
```

在上面的例子中，第一个块的 KV 缓存可以用 token “A gentle breeze stirred” 唯一标识。第三个块则可以用块内的 token “laughed in the distance” 以及前缀 token “A gentle breeze stirred the leaves as children” 唯一标识。因此，我们可以构建块哈希 `hash(tuple[components])`，其中组件包括：

* 父哈希值：父哈希块的哈希值。
* 块内 token：本块中 token 的元组。包含精确 token 是为了降低哈希值冲突的可能性。
* 额外哈希：使本块唯一所需的其他值，例如 LoRA ID、多模态输入哈希（见下方示例），以及用于在多租户环境中隔离缓存的 cache salt。

!!! note "注意 1"
    我们只缓存完整块。

!!! note "注意 2"
    在之前的版本中，哈希键并不保证无冲突。从 v0.11 开始，默认哈希算法改为 `sha256`，以降低冲突风险。

    对于 `vllm serve`，可通过 `--prefix-caching-hash-algo` 控制哈希算法：
    - `sha256`（默认）：使用 Python 的 `pickle` 进行序列化。哈希结果可能在不同 Python 或 vLLM 版本间不可复现。
    - `sha256_cbor`：使用 `cbor2` 进行序列化，提供可复现、跨语言兼容的哈希。推荐在需要确定性缓存的环境中使用。
    - `xxhash`：使用 Pickle 序列化结合 xxHash（128 位）实现更快的非加密哈希。需要安装可选的 `xxhash` 包。**重要提示**：使用非密码学安全的哈希算法在理论上会增加哈希冲突风险，可能导致未定义行为，甚至在多租户环境中泄露隐私信息。即使冲突概率仍然很低，也请在开启前权衡安全风险与性能收益。
    - `xxhash_cbor`：结合规范 CBOR 序列化与 xxHash，实现可复现哈希。同样需要可选的 `xxhash` 包。

**多模态输入的哈希示例**  
本例说明前缀缓存如何支持多模态输入（如图像）。假设有一个包含以下消息的请求：

```text
messages = [
    {"role": "user",
     "content": [
         {"type": "text",
          "text": "What's in this image?"
         },
         {"type": "image_url",
          "image_url": {"url": image_url},
         },
    ]},
]
```

它会变成如下提示：

```text
Prompt:
    <s>[INST]What's in this image?\n[IMG][/INST]

Tokenized prompt:
    [1, 3, 7493, 1681, 1294, 1593, 3937, 9551, 10, 4]

Prompt with placeholders (<P>):
    [1, 3, 7493, 1681, 1294, 1593, 3937, 9551, <P>, <P>, ..., <P>, 4]
```

可以看到，分词后 `[IMG]` 会被替换为一系列占位符 token，这些占位符在 prefill 阶段再被替换为图像嵌入。前缀缓存支持此场景的挑战在于需要区分图像与占位符。为此，我们编码由前端图像处理器生成的图像哈希。例如，上述提示中各块的哈希如下（假设块大小为 16，共有 41 个占位符 token）：

```text
Block 0
    Parent hash: None
    Token IDs: 1, 3, 7493, 1681, 1294, 1593, 3937, 9551, <p>, ..., <p>
    Extra hash: <image hash>
Block 1
    Parent hash: Block 0 hash
    Token IDs: <p>, ..., <p>
    Extra hash: <image hash>
Block 2
    Parent hash: Block 1 hash
    Token IDs: <p>, ..., <p>
    Extra hash: <image hash>
Block 3
    Parent hash: Block 2 hash
    Token IDs: <p>, ..., <p>, 4
    Extra hash: <image hash>
```

本文后续将先介绍 vLLM v1 中用于前缀缓存的数据结构，然后说明主要 KV 缓存操作（如分配、追加、释放、驱逐）的前缀缓存工作流，最后通过一个示例展示端到端的前缀缓存流程。

**安全的缓存隔离**  
为提升共享环境中的隐私保护，vLLM 支持通过可选的请求级 salting 隔离前缀缓存复用。在请求中加入 `cache_salt` 后，该值会被注入到第一个块的哈希中，确保只有使用相同 salt 的请求才能复用缓存的 KV 块。这可防止基于时序的攻击（攻击者通过观察延迟差异推断缓存内容），在不牺牲性能的前提下提供保护。

```json
{
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Here is a document with details about the world series: ..."},
    {"role": "user", "content": "Who won the world series in 2020?"}
  ],
  "cache_salt": "your-cache-salt"
}
```

通过此设置，缓存共享仅限于明确约定相同 salt 的用户或请求，从而实现信任组内的缓存复用，同时隔离其他请求。

## 数据结构

vLLM v1 的前缀缓存在 KV 缓存管理器中实现。基本构建单元是 “Block” 数据类（简化版）：

```python
class KVCacheBlock:
    # 块 ID（不可变）
    block_id: int
    # 块哈希（块满时赋值，块被驱逐时重置）
    block_hash: BlockHash
    # 当前使用该块的请求数量
    ref_cnt: int

    # 构成空闲队列双向链表的指针
    prev_free_block: "KVCacheBlock | None" = None
    next_free_block: "KVCacheBlock | None" = None
```

需要重点说明的两个设计点：

1. 初始化 KV 缓存管理器时，我们一次性分配所有 KVCacheBlock 作为块池。这样可避免 Python 对象创建开销，并能始终轻松追踪所有块。  
2. 我们在 KVCacheBlock 中直接引入双向链表指针，从而可直接构建空闲队列。这带来两个好处：  
    1. 将中间元素移动到队尾的时间复杂度为 O(1)。  
    2. 无需再引入额外的 Python 队列（如 `deque`）及其元素包装。

因此，KV 缓存管理器初始化后会有以下组件：
![Pasted image 20260820201606](../Images/Pasted%20image%2020260820201606.png)

![组件概览](../assets/design/prefix_caching/overview.png)

* Block Pool：KVCacheBlock 列表。  
* Free Block Queue：仅存储头尾块指针以便操作。  
* Cache blocks：从哈希键到块 ID 的映射。  
* Request blocks：从请求 ID 到已分配块 ID 的映射。

## 操作

### 块分配

**新请求：** 调度器为新请求进行 KV 缓存块分配的流程：

1. 调度器调用 `kv_cache_manager.get_computed_blocks()`，获取已计算的块序列。通过哈希请求中的提示 token 并查找缓存块完成。  
2. 调度器调用 `kv_cache_manager.allocate_slots()`，执行以下步骤：  
    1. 计算所需新块数量，若可用块不足则返回。  
    2. “触摸”已计算块。将已计算块的引用计数加一，若该块未被其他请求使用，则从空闲队列中移除，避免被驱逐。详见下一节示例。  
    3. 从空闲队列头部弹出块进行分配。若头部块是缓存块，则同时“驱逐”该块，使其之后无法被其他请求复用。  
    4. 若已分配的块已满，立即将其加入缓存块，以便同一批次中的其他请求复用。

**运行中请求：** 调度器为运行中请求进行 KV 缓存块分配的流程：

1. 调度器调用 `kv_cache_manager.allocate_slots()`，执行以下步骤：  
    1. 计算所需新块数量，若可用块不足则返回。  
    2. 从空闲队列头部弹出块进行分配。若头部块是缓存块，则同时“驱逐”该块，使其之后无法被其他请求复用。  
    3. 将 token ID 追加到已有块和新块的槽位中。块满后将其加入缓存块进行缓存。

**重复块**  
假设块大小为 4，发送一个请求（Request 1），提示为 ABCDEF，解码长度为 3：

```text
Prompt: [A, B, C, D, E, F]
Output: [G, H, I]

Time 0:
  Tokens: [A, B, C, D, E, F, G]
  Block Table: [0 (ABCD), 1 (EFG)]
  Cache Blocks: 0
Time 1:
  Tokens: [A, B, C, D, E, F, G, H]
  Block Table: [0 (ABCD), 1 (EFGH)]
  Cache Blocks: 0, 1
Time 2:
  Tokens: [A, B, C, D, E, F, G, H, I]
  Block Table: [0 (ABCD), 1 (EFGH), 2 (I)]
  Cache Blocks: 0, 1
```

此时块 0 和块 1 已被缓存。再次发送相同请求（Request 2），并使用贪心采样，使其产生与 Request 1 完全相同的输出：

```text
Prompt: [A, B, C, D, E, F]
Output: [G, H, I]

Time 0:
  Tokens: [A, B, C, D, E, F, G]
  Block Table: [0 (ABCD), 3 (EFG)]
  Cache Blocks: 0, 1
Time 1:
  Tokens: [A, B, C, D, E, F, G, H]
  Block Table: [0 (ABCD), 3 (EFGH)]
  Cache Blocks: 0, 1, 3
```

可以看到，块 3 是新的完整块并被缓存，但与块 1 重复，即同一内容被缓存了两次。在 v0 中，检测到块 3 重复后会释放块 3，并让 Request 2 使用块 1，使其块表在 Time 1 变为 `[0, 1]`。但在 vLLM v1 中，块表是只追加的，不允许将块表从 `[0, 3]` 改为 `[0, 1]`。因此，哈希键 E-H 会出现重复块。该重复会在请求被释放时消除。

### 释放

当请求完成时，若没有其他请求使用其块（引用计数 = 0），则释放所有相关块。在本例中，我们释放 Request 1 及其关联的块 2、3、4、8。可以看到，被释放的块按**逆序**加入空闲队列尾部。这是因为请求的最后一个块哈希了更多 token，被其他请求复用的可能性更低，因此应优先被驱逐。

![Pasted image 20260820201657](../Images/Pasted%20image%2020260820201657.png)
![请求释放后的空闲队列](../assets/design/prefix_caching/free.png)

### 驱逐（LRU）

当空闲队列的头部块（最近最少使用的块）是缓存块时，必须驱逐该块，以防止其他请求继续使用。驱逐具体步骤如下：

1. 从空闲队列头部弹出该块。这是将被驱逐的 LRU 块。  
2. 从缓存块中移除该块 ID。  
3. 移除块哈希。

## 示例

本例假设块大小为 4（每个块可缓存 4 个 token），KV 缓存管理器共有 10 个块。

**Time 1：缓存为空，新请求到来。** 分配 4 个块。其中 3 个已满并被缓存，第 4 个块部分填充（4 个槽位中有 3 个 token）。
![Pasted image 20260820201741](../Images/Pasted%20image%2020260820201741.png)

**Time 2：Request 0 使块 3 填满，并请求新块以继续解码。** 缓存块 3，并分配块 4

![Pasted image 20260820201906](../Images/Pasted%20image%2020260820201906.png)

**Time 3：Request 1 到来，提示共 14 个 token，前 10 个与 Request 0 相同。** 可以看到只有前 2 个块（8 个 token）命中缓存，因为第 3 个块仅匹配 4 个 token 中的 2 个。
![Pasted image 20260820201932](../Images/Pasted%20image%2020260820201932.png)



**Time 4：Request 0 完成并释放。** 块 2、3、4 按逆序加入空闲队列（但块 2 和 3 仍被缓存）。块 0 和 1 因仍被 Request 1 使用而未加入空闲队列。

![Pasted image 20260820202050](../Images/Pasted%20image%2020260820202050.png)

**Time 5：Request 1 完成并释放。

![Pasted image 20260820202102](../Images/Pasted%20image%2020260820202102.png)

**Time 6：Request 2 到来，提示共 29 个 token，前 12 个与 Request 0 相同。** 注意，即使空闲队列顺序原为 `7 - 8 - 9 - 4 - 3 - 2 - 6 - 5 - 1 - 0`，缓存命中块（即 0、1、2）会在分配前被触摸并从队列中移除，因此空闲队列变为 `7 - 8 - 9 - 4 - 3 - 6 - 5`。最终分配的块为 0（缓存）、1（缓存）、2（缓存）、7、8、9、4、3（被驱逐）。

![Pasted image 20260820202119](../Images/Pasted%20image%2020260820202119.png)