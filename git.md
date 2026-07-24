针对**团队协作**场景，给出两个**通用且专业**的工作流程。你可以根据团队对提交历史的偏好（线性 vs. 真实分叉）选择其一。

## 核心原则（所有流程都遵守）
1. **永远不要直接推送到 `main` / `master`**（除非是 hotfix 且经过紧急审批）。
2. **使用 Pull Request / Merge Request** 进行代码评审和自动检查。
3. **保持本地 `main` 始终与远程 `main` 同步**（每次开始前 `git pull`）。
4. **功能分支的命名清晰**（如 `feat/xxx`, `fix/xxx`）。

---

## 方案 A：线性历史工作流（推荐，更专业）
**特点**：最终 `main` 分支是一条干净的直线，无多余 merge commit。  
**适合**：注重可读性、推崇“每个提交都有意义”的团队（如开源项目、大型项目）。

### 完整步骤

```bash
# 1. 开始新功能
git checkout main
git pull origin main
git checkout -b feat/user-login

# ... 日常开发，多次 commit ...

# 2. 准备合并前，同步上游 main 并 rebase
git fetch origin
git rebase origin/main
# 如果有冲突，解决后：git add . && git rebase --continue

# 3. 强制推送功能分支（因为 rebase 改写了历史）
git push --force-with-lease origin feat/user-login

# 4. 在 GitLab/GitHub 上创建 Merge Request，目标分支 main
#    - 标题和描述清晰
#    - 关联 issue（如有）
#    - 等待 CI 通过 + 至少一人 Approve

# 5. 合并时选择 "Rebase and merge" 或 "Squash and merge"
#    - Rebase and merge：保留所有原始 commit（线性）
#    - Squash and merge：将所有 commit 合并成一个（推荐，保持主分支极简）
```

### 何时使用 `--force-with-lease` 而非 `--force`
- `--force-with-lease` 更安全：它会检查远程分支是否被其他人更新过，避免覆盖别人的工作。

---

## 方案 B：真实历史工作流（简单，适合多数团队）
**特点**：保留所有合并提交，真实反映分支何时合并。  
**适合**：团队不习惯 rebase，希望看到“何时把哪条分支合入”的明确记录。

### 完整步骤

```bash
# 1. 开始新功能（同方案 A）
git checkout main && git pull && git checkout -b feat/user-login

# 2. 日常开发，多次 commit

# 3. 准备合并前，同步上游 main 并 merge（不是 rebase）
git checkout feat/user-login
git merge main   # 或 git pull origin main
# 解决冲突后：git add . && git commit -m "Merge main into feat/user-login"

# 4. 推送功能分支（普通推送，不需要 --force）
git push origin feat/user-login

# 5. 创建 Merge Request，目标 main

# 6. 合并时选择 "Create a merge commit"（普通 merge）
#    这样会在 main 上生成一个 "Merge branch 'feat/user-login'" 提交
```

### 这个流程的优点
- 无需 `--force`，不会改写历史，对新手友好。
- 冲突解决记录清晰（merge commit 能看到整合点）。
- 如果功能分支已经推送到远程且多人协作，不会因为 rebase 导致混乱。
    git log --oneline --graph --all
---

## 进阶：处理已推送的功能分支且不想 rebase

如果你的功能分支已经推送到远程，并且**其他人基于它做了提交**，就不能 rebase。这时同步上游的正确做法是：

```bash
git checkout feat/user-login
git merge main        # 而不是 rebase
git push origin feat/user-login
```

然后在 MR 中正常合并。虽然会产生一个额外的 merge commit，但这是最安全的协作方式。

---

## 总结对比表

| 方面 | 方案 A（rebase + 线性历史） | 方案 B（merge + 真实历史） |
| :--- | :--- | :--- |
| 主分支历史形状 | 直线 | 有分叉和合并点 |
| 是否需要强制推送 | ✅ 需要 `--force-with-lease` | ❌ 不需要 |
| 解决冲突时机 | rebase 时（可能多次解决） | merge 时（一次解决） |
| 多人协作同一个功能分支 | ⚠️ 困难（需协调强制推送） | ✅ 容易（普通 push） |
| 代码评审工具推荐合并方式 | **Rebase and merge** 或 **Squash and merge** | **Create a merge commit** |
| 典型适用场景 | 成熟团队、开源项目、追求极简历史 | 内部团队、初学者、重视操作简便 |

## 团队统一规范建议

1. **在项目根目录添加 `.gitattributes` 或 `.gitmessage` 规范提交格式。**
2. **在 CI 中加入 `git diff --check` 等检查。**
3. **约定一个标准流程写入 `CONTRIBUTING.md`**，例如：
   > - 所有功能从 `main` 分支切出
   > - 使用 `git rebase origin/main` 同步上游（功能分支未公开时）或 `git merge main`（已公开时）
   > - 通过 MR 合并，至少一人 Approve，CI 为绿色
   > - 合并时选择 **Squash and merge**（保持主分支整洁）

选择适合你团队文化和工作习惯的流程，并确保**所有人遵循同一套规范**。