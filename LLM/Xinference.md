Xinference 在 Linux, Windows, MacOS 上都可以通过 `pip` 来安装。如果需要使用 Xinference 进行模型推理，可以根据不同的模型指定不同的引擎。

如果你希望能够推理所有支持的模型，可以用以下命令安装所有需要的依赖:

pip install "xinference[all]"

![[Pasted image 20260828195907.png]]

注意事项：
1、Xinference 会为每个模型建立独立的环境，这个环境与当前部署的环境（Xinfrence启动）的环境不一致。当然也可以让它使用部署环境。

![[Pasted image 20260709170127.png]]

2、注意模型虚拟空间的cuda版本-torch-vllm的版本


3、注意副本，当前的副本为1，意思是启动一个服务实例。
![[Pasted image 20260828195718.png]]

## 技术细节

```*
1448168 (xinference-local 主进程)
   ├── 1448530 (resource_tracker)
   └── 1448531 (spawn 子进程)
```


spawn的子进程进行了主环境变量的切割和修剪，同时后端采用xocar架构，非常适合异构。

> Xoscar 是一个专为**异构计算**（CPU、GPU等）设计的 Python Actor 框架。它的技术优势主要体现在：为构建高性能、高可用的分布式AI系统（如Xinference）提供了一个坚实、灵活且高效的“地基”。

### 集成关系：Xinference 如何用 xoscar “包装” vLLM

Xinference 为了在自身框架内更好地管理和调度 vLLM，会使用 xoscar 对 vLLM 进行“包装”。具体体现在：

- **替换执行器**：在多 GPU 场景下，Xinference 会**替换 vLLM 默认的执行器（Executor）**，改用自己基于 xoscar 实现的 `XinferenceDistributedExecutorV1`。
    
- **注册 WorkerActor**：这个新执行器会通过 xoscar 为每一个 GPU 进程（rank）注册一个 `WorkerActor`，从而实现跨进程的分布式协调。这解释了为何你之前用 `pstree` 看到的进程树中，`VLLM::EngineCore` 是 `python (Xinference主进程)` 的子进程。

![[Pasted image 20260828195430.png]]

可跳过模型

https://github.com/xorbitsai/xllamacpp

xinference launch -n Qwen3-Embedding-4B --model-engine llama.cpp --n_ctx 20480 --n_gpu_layers -1 --model-type embedding  --disable-virtual-env (跳過模型虛擬環境)--download_hub modelscope -f ggufv2

## 命令测试

`root@dkh:~# xinference list
UID          Type    Name     Format      Size (in billions)  Quantization
-----------  ------  -------  --------  --------------------  --------------
qwen3.6-35B  LLM     qwen3.6  pytorch                     35  none
qwen3.8-27B  LLM     qwen3.8  fp8                         27  FP8

UID                   Type       Name                    Dimensions
--------------------  ---------  --------------------  ------------
Qwen3-Embedding-0.6B  embedding  Qwen3-Embedding-0.6B          1024
Qwen3-Embedding-4B    embedding  Qwen3-Embedding-4B            2560

UID                Type    Name
-----------------  ------  -----------------
Qwen3-Reranker-8B  rerank  Qwen3-Reranker-8B
`