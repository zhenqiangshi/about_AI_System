https://docs.litellm.ai/docs/

LiteLLM 就是一个**企业级 AI 流量中枢**：

- 业务方只需要对接一个 endpoint + 一个虚拟 Key
- 后台统一管理所有模型、成本、权限、安全、日志
- 模型切换、故障切换、成本优化对上层应用透明
- 负载均衡：同一模型多部署（多 Azure/多区域）之间分流

---
## 1. 缓存设计 Cache

Redis缓存设计在LLM response返回进行缓存命中设计，同时在路由端保证多个worker可以共享状态。

## 2. 护栏设计 Guardrail

- 进入护栏：客户端只调 Proxy；Proxy 根据 guardrails 名字，在 pre_call 时自动把输入送进 Presidio/Lakera。
- 护栏内部：Presidio 本地扫 PII 并打码/拦截；Lakera 则是 Proxy 再发一次 HTTP 给远程接口。
- 怎么返回：护栏把「改写后的文本」或「拦截」交回 Proxy；Proxy 再决定调不调 LLM，以及最终给客户端什么。
- 谁调用、什么方式：对外只有客户端 → Proxy 这一次 HTTP；护栏是 Proxy 内部自动调用 的，客户端不会直接进 Presidio/Lakera。


## 3. 向量数据库

**LiteLLM 的 Vector Store = 把各家向量库/知识库收成统一接口的适配层，用来做权限管控和 RAG 检索。**

## 4. Auto 路由


## 5. 负载均衡


### 场景1举例

```markdown

你有：
- **多个系统**
- 其中某个系统下有 **4 个模块**（其他系统也可能有模块）
- 目标：**按系统统计用量**，同时 **按模块统计用量**
- 使用 LiteLLM 开源版 Proxy（Virtual Key / Team / Tag）

本质需求是两层归属：
1. 系统级总量（预算、权限、汇总）
2. 模块级明细（谁在花、花多少）

---

### 通解方案（推荐固定结构）

| 层级 | 用什么 | 对应业务 | 作用 |
|------|--------|----------|------|
| **系统** | **Team** | System-A、System-B… | 系统总预算、模型权限、系统级用量 |
| **模块** | **Service Account Key** | SystemA-Module1… | 模块身份、模块级用量；不绑定个人，人员变动不影响 |
| **业务维度（可选）** | **Tag** | `system:A`、`module:1`、`env:prod`… | 跨 Key 汇总、多维度分析、Tag 级预算 |

#### 具体怎么配

1. **每个系统建一个 Team**  
   - 例如：`System-A`、`System-B`  
   - 在 Team 上设总预算、可用模型、限流

2. **每个模块发一个 Service Account Key**  
   - 只选对应 Team，**不选 User**  
   - Key Alias 写清楚，如 `SystemA-Module1`  
   - 各模块用自己的 Key 调用

3. **Tag 按需加（可后期补）**  
   - 固定维度建议：`system:xxx`、`module:xxx`  
   - 需要时再加：`env:prod`、`cost-center:xxx`  
   - 创建时没打也没关系，之后在 UI / `/key/update` 里补即可  
   - **注意**：Tag 只对更新后的新请求生效，历史日志一般不回溯

#### 统计怎么看

| 问题 | 看哪里 |
|------|--------|
| 某系统一共花了多少？ | **Team** |
| 某模块花了多少？ | **Key** |
| 所有系统的「模块 1」合计？ | **Tag** `module:1` |
| 生产环境合计？ | **Tag** `env:prod` |

---

### 为什么这样最合适

- **Team** 解决系统隔离和系统预算  
- **Service Account Key** 解决模块稳定归属（不跟人走）  
- **Tag** 解决横切汇总（跨系统、跨模块、按环境/成本中心）  
- Key 已是 1 模块 1 Key 时，Tag 不是必须，但是低成本增强；可随时后补  

---

### 一句话落地口诀

> **Team 管系统，Key 管模块，Tag 管维度。**  
> 先建 Team + 模块 Key，Tag 有需要再补。


备注：
Organization = 事业部 / 区域 / 子公司这一层的硬隔离 + 分权管理。

系统、模块级统计与管控，用 Team + Key（+ Tag）即可；只有组织变大、要跨 BU 隔离和二级管理员时，Organization 才真正有价值。

```



**使用接口可以解锁一些UI上不被允许的功能**