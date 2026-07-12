# auto-issue-review

用 AI 自动审查 GitHub issue 模板的 GitHub Action。它的目标不是要求用户写出专业开发者式报告，而是帮助仓库维护者过滤明显缺失、占位或无法理解的 issue。

[英文文档](README.md)

## 这是什么项目

`auto-issue-review` 是一个 GitHub Action，用 OpenAI-compatible 模型检查新建或重新打开的 issue 是否符合对应模板。

它会读取仓库里的 issue template，通过 GitHub issue form 自动添加的 label 判断应该使用哪个模板，然后让 AI 判断用户是否已经把问题、需求或想法说清楚。

它支持：

- 多个 issue 模板
- AI 生成通过或拒绝评论
- 给有效、无效、达到检查上限、未匹配模板、跳过 AI 的 issue 添加标签
- 可选关闭不合格 issue
- 用户重新打开 issue 时再次检查
- 限制同一个 issue 的 AI 检查次数
- 用户回复 `manual review requested` 后停止 AI 处理
- 内置多语言机器人文案

审查标准按普通用户标准执行：只要别人能理解问题或想法，就不应该因为不够“专业”而拒绝。

## 如何工作

1. 用户创建或重新打开 issue。
2. GitHub issue form 自动添加 `bug`、`enhancement` 等标签。
3. `auto-issue-review` 根据标签匹配配置里的模板。
4. Action 读取对应的 issue template 文件。
5. 将 issue 标题、正文、模板和审查规则发送到 OpenAI-compatible `/chat/completions` 接口。
6. 模型返回 JSON：

   ```json
   {
     "valid": false,
     "reason": "标题太模糊。",
     "missing": ["标题"],
     "suggestedComment": "请让标题描述实际问题。"
   }
   ```

7. Action 根据配置执行：
   - 通过：评论，可选添加通过标签
   - 拒绝：添加标签、评论，可选关闭 issue
   - 达到检查次数上限：停止处理
   - 请求人工核查：添加 `no-ai-template`，以后跳过 AI

监听事件：

```yaml
issues:
  types: [opened, reopened]
issue_comment:
  types: [created]
```

编辑 issue 不会触发重新审查。如果 issue 被关闭，用户需要修改后重新打开。

## 如何开始使用

> 您可以使用仓库的示例文件作为模板修改，它们的作用就是这样

### 1. 创建标签

在目标仓库创建这些 label：

- `bug`
- `enhancement`
- `needs-template-fix`
- `template-ok`
- `template-check-limit`
- `needs-template-label`
- `no-ai-template`

`bug` 和 `enhancement` 通常是 GitHub 默认标签。如果你的仓库使用其他标签，需要同步修改 issue template 和 `.github/issue-ai-checker.json`。

### 2. 编写 issue 模板

Bug 模板示例：

```yaml
name: Bug report
description: Report a reproducible problem.
title: "[Bug]: "
labels: ["bug"]
body:
  - type: textarea
    id: description
    attributes:
      label: Problem description
    validations:
      required: true
  - type: textarea
    id: reproduce
    attributes:
      label: Steps to reproduce
    validations:
      required: true
  - type: textarea
    id: expected
    attributes:
      label: Expected behavior
    validations:
      required: true
  - type: textarea
    id: environment
    attributes:
      label: Environment
    validations:
      required: true
```

Feature 模板示例：

```yaml
name: Feature request
description: Suggest a new capability or improvement.
title: "[Feature]: "
labels: ["enhancement"]
body:
  - type: textarea
    id: problem
    attributes:
      label: Problem or use case
    validations:
      required: true
  - type: textarea
    id: proposal
    attributes:
      label: Proposed solution
    validations:
      required: true
  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives considered
    validations:
      required: false
```

可以禁用 blank issue：

```yaml
blank_issues_enabled: false
```

### 3. 添加配置文件

创建 `.github/issue-ai-checker.json`：

```json
{
  "maxChecksPerIssue": 3,
  "invalidAction": "label",
  "responseLanguage": "zh-CN",
  "templates": [
    {
      "label": "bug",
      "path": ".github/ISSUE_TEMPLATE/bug_report.yml",
      "name": "Bug report"
    },
    {
      "label": "enhancement",
      "path": ".github/ISSUE_TEMPLATE/feature_request.yml",
      "name": "Feature request"
    }
  ],
  "labels": {
    "invalid": "needs-template-fix",
    "valid": "template-ok",
    "maxChecksReached": "template-check-limit",
    "unmatchedTemplate": "needs-template-label",
    "manualReview": "no-ai-template"
  },
  "manualReview": {
    "enabled": true
  },
  "comments": {
    "valid": true,
    "invalid": true,
    "maxChecksReached": true,
    "unmatchedTemplate": true
  },
  "addValidLabel": false,
  "removeInvalidLabelWhenValid": true,
  "model": {
    "baseUrlEnv": "ISSUE_AI_BASE_URL",
    "apiKeyEnv": "ISSUE_AI_API_KEY",
    "nameEnv": "ISSUE_AI_MODEL",
    "temperatureEnv": "ISSUE_AI_TEMPERATURE",
    "maxTokensEnv": "ISSUE_AI_MAX_TOKENS",
    "timeoutMsEnv": "ISSUE_AI_TIMEOUT_MS",
    "extraHeadersEnv": "ISSUE_AI_EXTRA_HEADERS_JSON"
  }
}
```

### 4. 配置变量和密钥

Repository Variables：

| 名称 | 必填 | 示例 |
| --- | --- | --- |
| `ISSUE_AI_BASE_URL` | 是 | `https://api.openai.com/v1` |
| `ISSUE_AI_MODEL` | 是 | `gpt-4o-mini` |
| `ISSUE_AI_TEMPERATURE` | 否 | `0` |
| `ISSUE_AI_MAX_TOKENS` | 否 | `700` |
| `ISSUE_AI_TIMEOUT_MS` | 否 | `30000` |

Repository Secrets：

| 名称 | 必填 | 说明 |
| --- | --- | --- |
| `ISSUE_AI_API_KEY` | 是 | OpenAI-compatible 服务商 API key |
| `ISSUE_AI_EXTRA_HEADERS_JSON` | 否 | 额外 HTTP 请求头 JSON |

### 5. 添加 workflow

```yaml
name: "[review issue] Issue AI Checker"
run-name: "[review issue] #${{ github.event.issue.number }} ${{ github.event.issue.title }}"

on:
  issues:
    types: [opened, reopened]
  issue_comment:
    types: [created]

permissions:
  contents: read
  issues: write

concurrency:
  group: review-issue-${{ github.event.issue.number }}
  cancel-in-progress: false

jobs:
  check-issue:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Review issue
        uses: OWNER/auto-issue-review@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          config-path: .github/issue-ai-checker.json
          model-base-url: ${{ vars.ISSUE_AI_BASE_URL }}
          model: ${{ vars.ISSUE_AI_MODEL }}
          model-temperature: ${{ vars.ISSUE_AI_TEMPERATURE }}
          model-max-tokens: ${{ vars.ISSUE_AI_MAX_TOKENS }}
          model-timeout-ms: ${{ vars.ISSUE_AI_TIMEOUT_MS }}
          model-api-key: ${{ secrets.ISSUE_AI_API_KEY }}
          model-extra-headers-json: ${{ secrets.ISSUE_AI_EXTRA_HEADERS_JSON }}
```

将 `OWNER/auto-issue-review@v1` 替换成实际发布后的 action 地址和版本。

## 配置说明

### `maxChecksPerIssue`

同一个 issue 最多调用 AI 检查多少次。次数存储在机器人评论末尾的隐藏 HTML 标记里。

### `invalidAction`

未通过时怎么处理：

- `label`：添加标签并评论，不关闭 issue
- `close`：添加标签、评论，并以 `not_planned` 关闭 issue

### `responseLanguage`

控制内置机器人文案，也会告诉模型用什么语言回复。

当前内置语言：

- `en`
- `zh` / `zh-CN` / `zh-TW`

贡献新语言时，复制 `.github/review-issue/locales/en.json` 为新文件，例如 `.github/review-issue/locales/ja.json`，翻译其中的值，然后设置：

```json
{
  "responseLanguage": "ja"
}
```

如果找不到语言文件，会回退英文。

### `templates`

将 issue label 映射到模板文件：

```json
{
  "label": "bug",
  "path": ".github/ISSUE_TEMPLATE/bug_report.yml",
  "name": "Bug report"
}
```

`label` 必须和 issue form 自动添加的 label 一致。

### `labels`

Action 会使用这些标签：

- `invalid`：issue 需要补充模板信息
- `valid`：可选的通过标签
- `maxChecksReached`：达到检查次数上限
- `unmatchedTemplate`：没有匹配到模板标签
- `manualReview`：人工核查/跳过 AI 标签，默认 `no-ai-template`

### `manualReview`

启用后，用户可以评论：

```text
manual review requested
```

Action 会添加人工核查标签，并停止处理这个 issue。如果 issue 已关闭，用户还需要重新打开。

### `comments`

控制不同结果是否评论。

### `addValidLabel`

为 `true` 时，通过的 issue 会添加 `labels.valid`。

### `removeInvalidLabelWhenValid`

为 `true` 时，后续通过检查会移除无效标签。

### `model`

这里只保存环境变量名，不保存真实模型配置和密钥。

真实值应通过 GitHub Actions Variables 和 Secrets 配置。
