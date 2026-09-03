https://langfuse.com/self-hosting/upgrade/upgrade-guides/upgrade-v2-to-v3#docker-compose

## 国内安装步骤


```shell

# step1
sudo docker pull docker.m.daocloud.io/langfuse/langfuse:4
sudo docker pull docker.m.daocloud.io/langfuse/langfuse-worker:4


# docker tag 用于给已有的镜像打标签（添加新的引用名称）。
# 【含义】：它不复制镜像，只是增加一个新的别名。
# 【格式】：docker tag <源镜像>:<标签> <目标仓库>/<目标镜像>:<标签>
#
# 【使用场景】：
# 1. 重命名本地镜像：让它看起来更符合命名规范或包含版本信息。
# 2. 推送至私有仓库：为将镜像推送到 Docker Hub 或私有仓库做准备，通常需要将标签指向仓库地址。
# 3. 版本管理：通过打不同的 tag（如 v1.0, v1.1, latest）来区分镜像的不同版本。
# 打回原 tag
# step2
sudo docker tag docker.m.daocloud.io/langfuse/langfuse:4 langfuse/langfuse:4
sudo docker tag docker.m.daocloud.io/langfuse/langfuse-worker:4 langfuse/langfuse-worker:4

# step3
# 1. 备份 cp docker-compose.yml docker-compose.yml.bak

# 2. 替换镜像地址 
sed -i 's|docker.langfuse.com/langfuse/langfuse-worker:4|langfuse/langfuse-worker:4|g' docker-compose.yml 
sed -i 's|docker.langfuse.com/langfuse/langfuse:4|langfuse/langfuse:4|g' docker-compose.yml

# 3. 确认是否改成功（必须执行） grep -E "image:.*langfuse" docker-compose.yml

# step4
sudo docker compose up -d
```

==Tips: 为了增加LLM链接==

```shell
这几个环境变量是 Langfuse 为**防止 SSRF（服务器端请求伪造）攻击**而设置的安全白名单。

简单来说，它们控制着 Langfuse 在创建或测试“LLM 连接”（即与外部大模型 API 的连接）时，**允许访问哪些内部网络地址**。

### 三个变量的具体含义

- **`LANGFUSE_LLM_CONNECTION_WHITELISTED_HOST`**
    
    - 用于**白名单特定的主机名**。
        
    - 例如，如果你想允许 Langfuse 连接到一个内网中名为 `internal-llm.local` 的服务，可以将其加入此变量的列表。
        
- **`LANGFUSE_LLM_CONNECTION_WHITELISTED_IPS`**
    
    - 用于**白名单特定的 IP 地址**。
        
    - 例如，如果你的内网 LLM 网关 IP 是 `192.168.1.100`，可以将其添加到这里。
        
- **`LANGFUSE_LLM_CONNECTION_WHITELISTED_IP_SEGMENTS`**
    
    - 用于**白名单一个 IP 网段（CIDR 格式）**。
        
    - 例如，如果你想允许访问整个 `10.0.0.0/8` 内网段，可以在这里配置
```

![[Pasted image 20260902190308.png]]





下面按 **Observability（可观测性）**、**Prompt Manager（提示词管理）**、**Evaluation（评估）** 三大模块，系统整理 Langfuse 的核心流程细节与高级应用场景。

---

## 1. Observability（可观测性）

### 核心定位
把 LLM 应用从「黑盒」变成可追踪、可分析、可归因的系统，是后续 Prompt 迭代和评估的数据基础。

### 标准流程细节

1. **接入与埋点**
   - 使用 SDK（Python / JS）或 OpenTelemetry 自动/手动记录。
   - 推荐层级：Trace（一次完整请求）→ Span / Generation（中间步骤）→ Observation（具体 LLM 调用、检索、工具调用）。

2. **关键记录内容**
   - Input / Output
   - 模型名称、参数、Token、成本、延迟
   - Metadata（用户 ID、会话 ID、版本、环境、业务标签）
   - 工具调用参数与结果、检索到的文档

3. **日常使用流程**
   - 实时查看 Tracing 列表与详情
   - 用 Filters 按状态、延迟、成本、分数、Prompt 版本、用户等筛选
   - 下钻到具体 Observation，查看完整上下文
   - 关联 Session（多轮对话）进行整体分析

4. **生产监控**
   - 配置 Dashboard（成本、延迟、错误率、质量分数趋势）
   - 设置 Alerts（分数下降、成本飙升、错误激增）
   - 支持 User Feedback（点赞/点踩）直接关联 Trace

### 高级应用场景

- **Agent 轨迹分析**：可视化多步骤 Agent 的工具选择、推理路径、失败节点，定位是检索问题还是规划问题。
- **成本归因与优化**：按 Prompt 版本、模型、用户群体、功能模块拆解 Token 与费用，做精准优化。
- **会话级质量监控**：对多轮对话做 Session 级评分，发现「越聊越偏」的问题。
- **A/B 测试生产流量**：用 `release` / `version` 字段区分不同版本，实时对比质量、成本、延迟。
- **合规与审计**：完整保留输入输出与中间过程，满足审计与问题复盘需求。
- **从生产到评估的闭环起点**：低分 / 用户差评 Trace 一键进入 Annotation Queue 或 Dataset。

---

## 2. Prompt Manager（提示词管理）

### 核心定位
把 Prompt 从代码里的字符串变成可版本、可协作、可实验、可回滚的资产。

### 标准流程细节

1. **创建与组织**
   - 支持 Text 与 Chat 两种类型。
   - 使用 `{{variable}}` 定义动态变量。
   - 用 `/` 做文件夹层级管理（如 `hr/attendance`、`hr/salary`）。
   - 支持 Config（模型、温度、工具定义、JSON Schema 等与 Prompt 一起版本化）。

2. **版本与发布**
   - 每次修改自动生成新版本。
   - 用 Label（`production`、`staging`、`experiment-v3`）管理发布状态。
   - 支持 Prompt 引用（Composition），复用公共指令片段。

3. **使用方式**
   - SDK 拉取指定版本或 Label 的 Prompt，再 `.compile()` 填入变量。
   - 在 Playground 中快速测试。
   - 与 Tracing 自动关联（记录使用了哪个 Prompt 版本）。

4. **与评估联动**
   - 在 Prompt Experiment 中直接选择不同版本跑 Dataset。
   - 对比不同 Prompt 版本在相同测试集上的表现。

### 高级应用场景

- **Prompt 与配置一体化版本管理**：把 model、temperature、tools、response_format 全部放进 Config，实现「提示词 + 参数」原子化发布与回滚。
- **多环境发布策略**：`production` 标签稳定运行，`staging` 标签做灰度，实验标签快速验证。
- **团队协作与权限**：产品、算法、运营共同迭代 Prompt，保留完整变更历史。
- **动态变量 + 消息占位符**：支持 `{{question}}` 等变量，以及 Chat History 的 Message Placeholder，适配多轮与复杂上下文。
- **Prompt 性能归因**：通过 Tracing 关联，分析「哪个 Prompt 版本」在真实流量中的质量、成本、延迟表现最好。
- **与 Dataset 解耦的实验**：Prompt 负责指令，Dataset 负责测试用例，二者独立演进。

---

## 3. Evaluation（评估）

### 核心定位
建立可重复、可量化、可对比的质量衡量体系，支持离线回归测试与在线持续监控，并形成闭环。

### 标准流程细节

1. **Dataset 建设**
   - 从生产 Trace 一键添加（支持 Field Mapping）。
   - 或 CSV / SDK 批量导入。
   - Item 包含 `input`、`expected_output`、`metadata`。
   - 支持版本管理，保证实验可复现。

2. **定义评估器**
   - **Code Evaluator**：确定性检查（格式、关键词、引用、JSON 结构）。
   - **LLM-as-a-Judge**：语义评估（正确性、幻觉、有用性、语气等），支持模板与自定义 Rubric。
   - **Human Annotation**：通过 Annotation Queues 做黄金标注与校准。
   - 统一 Score Config（Numeric / Categorical / Boolean）。

3. **运行 Experiment**
   - UI 方式：选 Dataset + Prompt 版本，快速跑 Prompt Experiment。
   - SDK 方式：自定义完整应用逻辑（RAG / Agent），支持并发与自动打分。
   - 可锁定 Dataset 历史版本。

4. **结果分析**
   - 多 Run 并排对比，设置 Baseline，高亮变好/变差的 item。
   - Score Analytics 查看分布、趋势、评估器一致性。
   - 按 Metadata 切片分析不同场景表现。

5. **闭环回流**
   - 在线低分 / 用户差评 → Annotation Queue → 人工确认 → 加入 Dataset。
   - 新 Dataset 案例可立即用于后续 Experiment 与 CI 门禁。

### 高级应用场景

- **回归测试套件 + CI 门禁**：每次 Prompt / 模型 / 检索策略变更，自动跑核心 Dataset，分数不达标则阻断发布。
- **多维度评估体系**：Code Eval（硬规则）+ LLM Judge（语义）+ 人工校准，形成分层评估。
- **失败模式驱动迭代**：通过对比视图定位「哪些题变差了」，归类失败模式，针对性改 Prompt 或系统。
- **在线 + 离线双轨**：
  - 在线：采样生产流量持续打分，发现新问题。
  - 离线：固定 Dataset 做版本对比，防止回归。
- **评估器校准**：用人工标注与 LLM Judge 结果做一致性分析（Score Analytics），持续优化 Judge Prompt。
- **业务切片评估**：按 metadata（问题类型、难度、部门、语言）分析不同子集表现，发现系统性薄弱点。
- **Agent / RAG 专项评估**：对检索相关性、工具调用正确性、任务完成度、轨迹效率等做专项打分。
- **成本-质量权衡决策**：在 Experiment 对比中同时看分数、成本、延迟，选择最优平衡点。

---

### 三大模块的协作关系（高级视角）

```
Observability（生产真相）
        ↓ 低分 / 差评 / 边界案例
Evaluation（Dataset + Experiment + Score）
        ↓ 发现有效改进
Prompt Manager（版本化迭代）
        ↓ 发布新版本
Observability（验证真实效果）
        ↓ 继续发现新问题
形成持续飞轮
```

- **Observability** 负责「看到真实发生了什么」。
- **Prompt Manager** 负责「把改进变成可管理的资产」。
- **Evaluation** 负责「用数据证明改进是否有效，并防止回归」。

三者打通后，才能从「靠感觉调 Prompt」升级为「**数据驱动的 LLM 工程体系**」。
