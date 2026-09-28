https://docs.litellm.ai/docs/

LiteLLM 就是一个**企业级 AI 流量中枢**：

- 业务方只需要对接一个 endpoint + 一个虚拟 Key
- 后台统一管理所有模型、成本、权限、安全、日志
- 模型切换、故障切换、成本优化对上层应用透明
- 负载均衡：同一模型多部署（多 Azure/多区域）之间分流

---
## 缓存设计 Cache

Redis缓存设计在LLM response返回进行缓存命中设计，同时在路由端保证多个worker可以共享状态。

## 护栏设计 Guardrail

- 进入护栏：客户端只调 Proxy；Proxy 根据 guardrails 名字，在 pre_call 时自动把输入送进 Presidio/Lakera。
- 护栏内部：Presidio 本地扫 PII 并打码/拦截；Lakera 则是 Proxy 再发一次 HTTP 给远程接口。
- 怎么返回：护栏把「改写后的文本」或「拦截」交回 Proxy；Proxy 再决定调不调 LLM，以及最终给客户端什么。
- 谁调用、什么方式：对外只有客户端 → Proxy 这一次 HTTP；护栏是 Proxy 内部自动调用 的，客户端不会直接进 Presidio/Lakera。

