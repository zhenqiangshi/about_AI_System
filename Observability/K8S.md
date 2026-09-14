
概念梳理

|工具|定位|你主要用它做什么|需要会什么|
|---|---|---|---|
|**K8s**|底层平台|真正跑容器的地方|理解概念即可|
|**kubectl**|命令行工具|精确操作、排查、脚本|命令 + kubeconfig|
|**Rancher**|图形化管理平台|点点鼠标管理集群和应用|会网页操作|
|**Helm**|应用包管理器|安装/升级一整套应用|Chart + values|

### 1. Kubernetes（K8s）

**本质**：容器编排平台（集群的“操作系统”）

**核心能力**：

- 调度容器（Pod）到哪台机器跑
- 自动重启、扩缩容、滚动更新
- 服务发现与负载均衡（Service）
- 存储挂载、配置管理（ConfigMap/Secret）
- 声明式管理：你写 YAML 描述期望状态，它负责达成

**你不直接“用”它**，而是通过工具（kubectl、Rancher、Helm）去操作它。

---

### 2. kubectl

**本质**：K8s 的命令行客户端（官方 CLI）

**核心能力**：

- 查看资源：get、describe、logs
- 创建/删除/修改：apply、delete、edit
- 调试：exec 进容器、port-forward
- 管理配置：切换集群、上下文、命名空间

**特点**：

- 最底层、最精确的控制方式
- 必须有有效的 kubeconfig（证书或 Token）才能用
- 适合运维、脚本、自动化

---

### 3. Rancher

**本质**：K8s 的图形化管理平台 + 多集群管理工具

**核心能力**：

- 网页界面管理集群（你现在用的那个 UI）
- 一键创建/导入 K8s 集群
- 多集群统一管理、权限（RBAC）、项目隔离
- 应用商店、监控、日志、备份等扩展
- 给用户下发 kubeconfig、管理成员权限
- 自带网页版 kubectl / Shell

**和 kubectl 的关系**：

- Rancher UI 底层还是在调 K8s API
- 它帮你解决了认证、权限、多集群切换的麻烦
- 适合不熟命令行的人，或需要多人协作、权限管控的场景

---

### 4. Helm

**本质**：K8s 的包管理器（类似 apt/yum，但针对 K8s 应用）

**核心能力**：

- 把一组 YAML（Deployment、Service、ConfigMap…）打包成 **Chart**
- 一键安装、升级、回滚复杂应用
- 用 values.yaml 做参数化配置
- 管理应用版本和依赖

**典型用途**：

- 安装 Cube Studio、Prometheus、MySQL、Nginx Ingress 等
- 升级时只改配置，不用手动改一堆 YAML

**和另外三个的关系**：

Helm  → 生成/管理 YAML
       ↓
kubectl / Rancher  → 把 YAML 交给 K8s
       ↓
K8s  → 真正运行容器