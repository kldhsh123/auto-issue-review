# auto-issue-review

AI-powered GitHub issue template review for repositories that want cleaner issues without forcing users to write like professional developers.

[中文文档](docs/README.zh-CN.md)

Friendship links: [linux.do](https://linux.do/)

## What This Project Is

`auto-issue-review` is a GitHub Action that reviews newly opened or reopened issues with an OpenAI-compatible model.

It reads your repository's issue templates, picks the right template by the labels added by GitHub issue forms, and asks AI whether the issue is understandable enough for a normal user report.

It can:

- review multiple issue templates
- comment with AI-generated feedback
- add labels for valid, invalid, max-check, unmatched-template, and no-AI cases
- optionally close invalid issues
- check again when a user reopens an issue
- stop after a configurable number of checks per issue
- let users opt out of AI review by replying `manual review requested`
- use localized built-in bot messages

The review standard is intentionally user-friendly. The issue does not need to be a perfect engineering report. It only needs to clearly explain the problem, request, or idea.

## How It Works

1. A user opens or reopens an issue.
2. GitHub issue forms add labels such as `bug` or `enhancement`.
3. `auto-issue-review` matches the issue label to a configured template.
4. The action reads the matching issue template file.
5. The issue title, body, template, and review rules are sent to an OpenAI-compatible chat completions API.
6. The model returns JSON:

   ```json
   {
     "valid": false,
     "reason": "The title is too vague.",
     "missing": ["Title"],
     "suggestedComment": "Please make the title describe the actual problem."
   }
   ```

7. The action applies the configured behavior:
   - valid: optionally add a success label and comment
   - invalid: add a label, comment, and optionally close
   - too many checks: stop processing the issue
   - manual review requested: add `no-ai-template` and skip future AI checks

The workflow listens to:

```yaml
issues:
  types: [opened, reopened]
issue_comment:
  types: [created]
```

Editing an issue does not trigger a new review. If an invalid issue was closed, users should edit it and reopen it.

## Getting Started

> You can use the example files in the repository as templates and modify them instead of rewriting everything from scratch—that is exactly what they are intended for.

### 1. Create Labels

Create these labels in the target repository:

- `bug`
- `enhancement`
- `needs-template-fix`
- `template-ok`
- `template-check-limit`
- `needs-template-label`
- `no-ai-template`

`bug` and `enhancement` are GitHub default labels in many repositories. If your repository uses different labels, update both the issue templates and `.github/issue-ai-checker.json`.

### 2. Add Issue Templates

Example bug template:

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

Example feature template:

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

Optional issue template config:

```yaml
blank_issues_enabled: false
```

### 3. Add Configuration

Create `.github/issue-ai-checker.json`:

```json
{
  "maxChecksPerIssue": 3,
  "invalidAction": "label",
  "responseLanguage": "en",
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

### 4. Configure Repository Variables and Secrets

Add repository variables:

| Name | Required | Example |
| --- | --- | --- |
| `ISSUE_AI_BASE_URL` | Yes | `https://api.openai.com/v1` |
| `ISSUE_AI_MODEL` | Yes | `gpt-4o-mini` |
| `ISSUE_AI_TEMPERATURE` | No | `0` |
| `ISSUE_AI_MAX_TOKENS` | No | `700` |
| `ISSUE_AI_TIMEOUT_MS` | No | `30000` |

Add repository secrets:

| Name | Required | Description |
| --- | --- | --- |
| `ISSUE_AI_API_KEY` | Yes | API key for the OpenAI-compatible provider |
| `ISSUE_AI_EXTRA_HEADERS_JSON` | No | JSON object for extra HTTP headers |

### 5. Add Workflow

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
    if: ${{ github.event.issue.pull_request == null }}
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Check issue template compliance
        uses: ./
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

Replace `OWNER/auto-issue-review@v1` with the actual Marketplace action owner and release tag.

## Configuration Reference

### `maxChecksPerIssue`

Maximum number of AI checks for one issue. The count is stored in a hidden marker appended to bot comments.

### `invalidAction`

Controls what happens when the model rejects an issue.

- `label`: add the invalid label and comment, but leave the issue open
- `close`: add the invalid label, comment, and close the issue as `not_planned`

### `responseLanguage`

Controls built-in bot text and tells the model what language to use.

Currently bundled languages:

- `en`
- `zh` / `zh-CN` / `zh-TW`

To contribute a new language, copy `.github/review-issue/locales/en.json` to a new file, for example `.github/review-issue/locales/ja.json`, translate the values, and set:

```json
{
  "responseLanguage": "ja"
}
```

If a locale file is missing, built-in messages fall back to English.

### `templates`

Maps issue labels to template files.

```json
{
  "label": "bug",
  "path": ".github/ISSUE_TEMPLATE/bug_report.yml",
  "name": "Bug report"
}
```

The `label` must match a label added by the issue form.

### `labels`

Defines labels the action applies:

- `invalid`: issue needs template fixes
- `valid`: optional success label
- `maxChecksReached`: check limit reached
- `unmatchedTemplate`: no configured template label matched
- `manualReview`: manual override label; defaults to `no-ai-template`

### `manualReview`

When enabled, users can comment:

```text
manual review requested
```

The action adds the manual-review label and stops processing that issue. If the issue is closed, users should reopen it after commenting.

### `comments`

Turns bot comments on or off for specific outcomes.

### `addValidLabel`

When `true`, valid issues receive `labels.valid`.

### `removeInvalidLabelWhenValid`

When `true`, the invalid label is removed after an issue passes a later review.

### `model`

The `model` section names environment variables. It does not contain actual credentials or provider values.

This keeps model configuration out of the repository and lets users configure values through GitHub Actions variables and secrets.
