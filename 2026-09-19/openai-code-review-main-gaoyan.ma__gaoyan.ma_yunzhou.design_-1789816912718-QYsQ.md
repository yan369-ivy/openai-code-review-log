# 代码评审报告

## 一、变更概述

本次变更修改了 GitHub Actions 工作流文件 `.github/workflows/main-remote-jar.yml` 中的环境变量配置，将原本用于 **ChatGLM（智谱AI）** 的配置替换为 **DeepSeek** 的配置：

| 变更项 | 变更前 | 变更后 |
|--------|--------|--------|
| API 主机地址 | `CHATGLM_APIHOST` | `DEEPSEEK_API_HOST` |
| API 密钥 | `CHATGLM_APIKEYSECRET` | `DEEPSEEK_API_KEY` |

---

## 二、存在的问题与风险

### 🔴 1. 注释未同步更新（严重）

```yaml
# OpenAi - ChatGLM 配置「https://open.bigmodel.cn/api/paas/v4/chat/completions」、「https://open.bigmodel.cn/usercenter/apikeys」
DEEPSEEK_API_HOST: ${{ secrets.DEEPSEEK_API_HOST }}
DEEPSEEK_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
```

**问题**：注释仍然指向 ChatGLM 的文档地址，而实际变量已切换为 DeepSeek。这会对后续维护者造成严重误导。

**建议修改**：
```yaml
# DeepSeek 配置「https://api.deepseek.com」、「https://platform.deepseek.com/api_keys」
DEEPSEEK_API_HOST: ${{ secrets.DEEPSEEK_API_HOST }}
DEEPSEEK_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
```

---

### 🔴 2. 变量名称变更但应用代码未同步的风险（高危）

从命名看，应用代码中很可能存在读取 `CHATGLM_APIHOST` / `CHATGLM_APIKEYSECRET` 的逻辑。如果只改了 workflow，未同步修改：

- **Java 应用配置类** / **`application.yml`** 中 `@Value` 或 `Environment.getProperty` 的读取键
- **Spring Boot 环境变量映射**（`CHATGLM_APIHOST` → `chatglm.apiHost`）

则会出现**启动时变量为 null** 或 **运行时调错接口** 的问题。

**建议**：确认仓库中是否有类似下图的引用，一并提交修改：
```java
@Value("${CHATGLM_APIHOST}")
private String apiHost;

@Value("${CHATGLM_APIKEYSECRET}")
private String apiKeySecret;
```

---

### 🟡 3. 命名规范不一致

原变量 `CHATGLM_APIHOST` 与 `CHATGLM_APIKEYSECRET` 是 `大写无分隔` 风格，新变量 `DEEPSEEK_API_HOST` 改用了 `下划线分隔` 风格。

**风险点**：
- 如果应用中通过 Spring Boot 的 `relaxed binding`（宽松绑定）读取（如 `chatglm.apihost`），环境变量命名风格变化可能导致无法匹配。
- 环境变量名应保持项目内命名一致性。

**建议**：确认应用侧读取方式与命名风格后再定稿。若项目统一使用下划线风格，建议将旧变量也重构；若保持原风格，则应命名为 `DEEPSEEK_APIHOST` / `DEEPSEEK_APIKEY`。

---

### 🟡 4. Secrets 配置需要在仓库端同步设置

变更引入了新 secret：
- `DEEPSEEK_API_HOST`
- `DEEPSEEK_API_KEY`

**必须确认**：GitHub 仓库的 Settings → Secrets and variables → Actions 中已创建这两个 secret，否则工作流会读到**空字符串**（GitHub 不会因 secret 不存在而报错），导致运行时静默失败。

**建议**：在 PR 描述中明确列出需要新增的 secrets 清单，或提供迁移说明文档。

---

### 🟢 5. 缺乏平滑迁移策略

本次为「**硬替换**」，一旦合并：
- 旧的 ChatGLM 逻辑立即失效
- 无法快速回滚到 ChatGLM（需再改回来）

**建议**：若需要保持灵活性，可考虑同时保留两组变量，让应用按优先级/开关选择，例如：

```yaml
# 大模型服务配置（二选一，通过 LLM_PROVIDER 控制）
LLM_PROVIDER: deepseek
DEEPSEEK_API_HOST: ${{ secrets.DEEPSEEK_API_HOST }}
DEEPSEEK_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
# 备用：ChatGLM
# CHATGLM_APIHOST: ${{ secrets.CHATGLM_APIHOST }}
# CHATGLM_APIKEYSECRET: ${{ secrets.CHATGLM_APIKEYSECRET }}
```

---

## 三、合并前 Checklist

- [ ] 更新代码注释为 DeepSeek 官方地址
- [ ] 确认 Java 代码中读取的环境变量名已同步修改（含 `application.yml`、`@Value`）
- [ ] 在 GitHub 仓库中新增 `DEEPSEEK_API_HOST`、`DEEPSEEK_API_KEY` 两个 secrets
- [ ] 确认应用侧变量命名风格（下划线 vs 无分隔）与 Spring 宽松绑定规则一致
- [ ] 检查是否有其他 workflow / 文档 / README 引用了旧变量名
- [ ] 建议保留旧变量一段时间或写清回滚流程

---

## 四、总体评价

| 维度 | 评价 |
|------|------|
| 变更目的 | ✅ 明确合理（切换 LLM 供应商） |
| 变更最小化 | ✅ 仅 2 行改动，聚焦 |
| 一致性 | ❌ 注释未更新、命名风格不统一 |
| 完整性 | ⚠️ 需验证应用侧和 secrets 同步 |
| 可回滚性 | ⚠️ 硬替换，回滚成本略高 |

**结论**：变更本身方向正确，但**不能仅以这 2 行改动合并**。请务必完成应用代码同步、注释更新、secrets 创建与命名风格对齐，否则极易出现线上/CI 静默失败的问题。

**建议合并策略**：将 workflow 修改、应用代码修改、README/注释更新放到同一个 PR 中一起提交，做到"改名一致、链路闭环"。