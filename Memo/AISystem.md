
# K8s + Ray

https://mp.weixin.qq.com/s/j1pq-Fus3Rr-9ATsIQYKwg k8s+ray 腾讯云技术




对话助手：

历史对话直接嵌入


1、多轮对话优化
     无论是否启用"多轮对话优化"（`refine_multiturn`），**完整的历史对话都会被加入到发给大模型的 `messages` 中**。这个开关只影响**检索用的查询问题**是否被改写，并不影响历史消息是否传给模型生成答案。

      历史消息本身会随着对话轮次不断累积增长，但发给大模型的最终 `msg`/`llmMessages` **不会无限增大**——系统会用 `message_fit_in`（Python）/ `messageFitIn`（Go）在生成前按 token 预算做裁剪，超出预算时会丢弃/裁切中间内容，只保留 system 消息 + 最后一条消息，必要时再对内容做 token 级截断。
   
2、元数据自动识别
3、关键词提取
3、关键词检索
4、答案生成