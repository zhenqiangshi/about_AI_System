1. https://recipes.vllm.ai/Qwen/Qwen3.6-35B-A3B
2. https://deepwiki.com/search/_83254ce1-71d4-4db6-8bd0-19fde0ca34b9

- PagedAttention 解决了内存碎片问题（KV Cache 可以动态分配/释放）。
- Continuous Batching 解决了请求调度问题。


GPU 显存有限，但同一时刻可能有成百上千个请求在排队等待生成文本。如果每次只处理一个请求，GPU 算力会被大量浪费在等待和小批量计算上；但如果把所有等待中的请求一次性塞进一个批次，KV cache 显存会瞬间爆炸。vLLM 需要一种机制，在**每一步**都动态决定"这一轮该跑哪些请求、每个请求跑多少 token"，同时保证显存块（PagedAttention 的 KV cache 分页）分配合理——这就是调度器（Scheduler）与执行循环存在的原因。

# Details

整个引擎的执行是一个持续运行的循环，每调用一次 `LLMEngine.step()` [4a] 就完成一轮"调度 → 计算 → 采样 → 更新状态"。

1. **调度阶段**：`scheduler.schedule()` [4b] 根据当前 KV cache 空闲块数量、请求等待队列的优先级，决定这一轮哪些请求参与计算、prefill 阶段跑多少 token、decode 阶段跑多少 token。这正是 **continuous batching**（连续批处理）和 **chunked prefill**（分块预填充）的核心实现位置——它让新来的请求可以随时插入正在运行的批次，而不必等当前批次全部结束。
    
2. **执行阶段**：调度结果被交给 `model_executor.execute_model()` [4c]，它把输入张量分发到实际的 GPU worker（`GPUModelRunner`）上跑一次模型前向传播，得到每个请求当前位置的 logits。
    
3. **采样阶段**：在 `GPUModelRunner` 内部的 `sampler(logits, sampling_metadata)` [4d] 处，每个请求各自的 `SamplingParams`（用户指定的 `temperature`、`top_p`、`top_k` 等）被应用到对应的 logits 上，决定下一个生成的 token 是什么。**这里就是用户指定的采样参数真正生效的地方**——无论请求来自离线 `LLM.generate()` 还是在线 HTTP API，最终都会汇聚到这一步。
    
4. **状态更新阶段**：`scheduler.update_from_output()` [4e] 把新采样出的 token 写回各请求的状态，检查是否命中停止条件（stop token、`max_tokens` 上限等），并相应地释放或续用 PagedAttention 的 KV cache 分页块，为下一轮 `step()` 做准备。
    

理解这个循环的关键是：**调度器和执行器是解耦的**——调度器只管"分配显存块和决定批次构成"，执行器只管"跑模型"，采样逻辑则是把用户级参数和底层 logits 连接起来的桥梁。所有上层调用（离线库调用或在线 HTTP 请求）本质上都是在反复驱动这同一个循环。


|镜像源|地址|推荐指数|备注|
|---|---|---|---|
|**清华**|[https://pypi.tuna.tsinghua.edu.cn/simple](https://pypi.tuna.tsinghua.edu.cn/simple)|★★★★★|最稳定常用|
|阿里云|[https://mirrors.aliyun.com/pypi/simple/](https://mirrors.aliyun.com/pypi/simple/)|★★★★|速度也很快|
|中科大|[https://pypi.mirrors.ustc.edu.cn/simple/](https://pypi.mirrors.ustc.edu.cn/simple/)|★★★★|教育网友好|
|豆瓣|[https://pypi.douban.com/simple/](https://pypi.douban.com/simple/)|★★★|偶尔不稳定|
|华为云|[https://mirrors.huaweicloud.com/repository/pypi/simple](https://mirrors.huaweicloud.com/repository/pypi/simple)|★★★★|企业常用|
|腾讯云|[https://mirrors.cloud.tencent.com/pypi/simple](https://mirrors.cloud.tencent.com/pypi/simple)|★★★★|-|

#### 常用命令

```bash
# 推荐写法
uv pip install vllm --torch-backend=auto          # 自动检测驱动
uv pip install vllm --torch-backend=cu130         # 明确指定
uv pip install vllm --torch-backend=cu129         # 老驱动回退

# 直接装特定变体
uv pip install https://github.com/vllm-project/vllm/releases/download/v0.28.0/vllm-0.28.0+cu129-cp38-abi3-manylinux_2_35_x86_64.whl --torch-backend=cu129
```

> 一定注意cuda、驱动、python wheels等的 一些适配！



