# 代码评审报告

## 变更概述
本次变更将 GitHub Actions 工作流 `main-maven-jar.yml` 中的触发分支从 `master` 改为 `main`。

```diff
-      - master
+      - main
```

## 评审意见

### ✅ 变更合理性
1. **符合行业趋势**：GitHub 官方在 2020 年已将默认分支名从 `master` 改为 `main`，此变更与主流规范保持一致，属于合理的现代化调整。
2. **语义清晰**：`main` 作为主分支命名更加中性、明确，避免了 `master` 带来的历史语义争议。
3. **变更范围最小化**：仅修改了 push 和 pull_request 两个触发条件的 branch 名称，未引入额外副作用。

### ⚠️ 需要确认的关键点

**1. 仓库实际默认分支是否已重命名？**
这是本次变更能否生效的**核心前提**。请确认：
- 仓库 Settings → Branches → Default branch 是否已切换为 `main`？
- 若仓库实际分支仍叫 `master`，此变更会导致 CI **完全不会被触发**（既不会在推送时运行，也不会在 PR 时运行），属于高风险变更。

**2. 是否存在其他引用 `master` 的位置？**
建议全局搜索确认一致性：
```bash
grep -rn "master" .github/ 
grep -rn "refs/heads/master" .
```
典型遗漏点：
- 其他 workflow 文件（如 `release.yml`、`deploy.yml`）
- README 徽章（badge）中的分支引用
- `dependabot.yml` 中的 `target-branch`
- 项目文档、脚本中的分支引用
- IDE 配置文件（`.idea/`、`.vscode/`）

**3. 历史 PR 与自动化任务**
- 已存在的、基于 `master` 的 PR 是否需要 rebase？
- 若有基于 `master` 触发的 cron 任务或其他调度，需同步调整。

### 💡 优化建议

**建议使用通配或双分支过渡（可选）**
若需平滑迁移，可短期同时保留两个分支以降低风险：
```yaml
on:
  push:
    branches:
      - main
      - master
  pull_request:
    branches:
      - main
      - master
```
待确认 `main` 分支 CI 稳定运行后，再移除 `master`。

**建议使用变量或复用**
若文件数量较多，可考虑提取分支名为组织级变量（GitHub Variables）或 `env`，减少后续维护成本：
```yaml
env:
  DEFAULT_BRANCH: main

on:
  push:
    branches:
      - ${{ vars.DEFAULT_BRANCH }}  # 需在仓库/组织设置 Variables
```
（注意：`on` 段是否支持表达式在不同场景下有限制，请按需评估。）

### 🔒 风险评估
| 项 | 等级 | 说明 |
|---|---|---|
| 分支名不匹配导致 CI 失效 | 🔴 高 | 若仓库默认分支未同步重命名 |
| 遗漏其他文件的 `master` 引用 | 🟡 中 | 可能造成部分 workflow 行为不一致 |
| 变更本身回滚成本 | 🟢 低 | 单文件单行，易回滚 |

## 结论
✅ **变更方向正确，但合并前必须确认仓库默认分支已同步为 `main`，并全局搜索清理残留的 `master` 引用。** 建议合并前完成一次 dry-run 或临时保留双分支以验证 CI 触发正常。

---
*评审人：架构评审 · 关注点：CI 有效性与跨文件一致性*