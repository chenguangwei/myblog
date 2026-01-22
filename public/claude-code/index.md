# Claude Code 精通指南：实现 10 倍效率的高级模式


# Claude Code 精通指南：实现 10 倍效率的高级模式

**一份关于那些真正能带来实质提升的模式的综合指南**

*基于 2000+ 小时 LLM（大语言模型）辅助编程经验*

-----

## 目录

1.  [核心理念：Context（上下文）工程](#1-核心理念context上下文工程)
2.  [模式 1：错误日志系统](#2-模式-1错误日志系统)
3.  [模式 2：Slash 命令作为轻量级本地应用](#3-模式-2slash-命令作为轻量级本地应用)
4.  [模式 3：用于确定性安全的 Hooks](#4-模式-3用于确定性安全的-hooks)
5.  [模式 4：Context 卫生管理](#5-模式-4context-卫生管理)
6.  [模式 5：子 Agent（智能体）控制](#6-模式-5子-agent智能体控制)
7.  [模式 6：重提示（Reprompter）系统](#7-模式-6重提示reprompter系统)
8.  [子 Agent 监控仪表盘](#8-子-agent-监控仪表盘)
9.  [快速参考表](#9-快速参考表)

-----

## 1. 核心理念：Context（上下文）工程

### 根本性的思维转变

**LLM 生成代码中的任何问题，归根结底都是你的责任。**

这并不是指责，而是赋能。每一个错误都可以追溯到：

-   **Prompt（提示词）不当**：指令模糊、缺少约束条件、成功标准不明确。
-   **Context 工程不当**：加载了错误的文件、Context 过时、缺少架构知识。
-   **Context 腐烂 (Context rot)**：随着无关信息填满 Context 窗口，质量随之下降。
-   **中间迷失 (Lost in the middle)**：LLM 对 Context 中间部分的信息关注度较低，这是一个有据可查的现象。

### Context 质量曲线

```
输出质量
     │
 100%├────────╮
     │        │╲
  80%│        │ ╲
     │        │  ╲ ← "中间迷失" 开始出现
  60%│        │   ╲
     │        │    ╲____
  40%│        │         ╲___
     │        │              ╲___
  20%│        │                   ╲
     └────────┴───────────────────────► Context 使用率 %
           20%   40%   60%   80%  100%
```

**关键洞察**：质量下降是非线性的。最后 20% 的 Context 往往是“毒药”。

-----

## 2. 模式 1：错误日志系统

### 问题所在

Agentic（代理式）编程对你隐藏了输入-输出的循环。Claude 做出决定，执行工具，而你只看到最终结果。当事情出错时，你不知道*原因*。

### 解决方案：重构反馈循环

建立一个系统的失败日志，包含：

1.  **触发该问题的确切 Prompt**
2.  **Context 状态**（加载了哪些文件，使用百分比）
3.  **失败模式**（输出错误、幻觉、拒绝执行、死循环）
4.  **你的诊断**（你哪里做错了？）

### 实现

#### 错误日志模板 (`~/.claude/error-log.md`)

```markdown
# Claude Code 错误日志

## 记录格式
- **Date**: 
- **Prompt**: [确切的文本]
- **Context %**: 
- **Files Loaded**: 
- **Failure Mode**: [wrong-output | hallucination | refusal | infinite-loop | context-rot]
- **What Claude Did**: 
- **What I Expected**: 
- **Root Cause**: [prompting | context | model-limitation]
- **Fix Applied**: 
- **Pattern Identified**: 

---

## 2025-01-03 | Auth 重构中的 Context 腐烂

**Prompt**: "Now refactor the auth module to use the new pattern" (现在重构 auth 模块以使用新模式)

**Context %**: 78%

**Files Loaded**: 12 files from src/auth/, 3 config files, previous conversation

**Failure Mode**: wrong-output (输出错误)

**What Claude Did**: 使用了旧模式，尽管 30 条消息前已经展示了新模式

**What I Expected**: 应用第 4 条消息中的新 auth 模式

**Root Cause**: context - 新模式处于“中间迷失”区域

**Fix Applied**: /clear → 重新陈述模式 → 指向包含该模式的具体文件

**Pattern Identified**: 如果 Context 使用率 >50%，在要求实施之前，务必重新陈述关键模式

---
```

#### 用于快速记录的 Slash 命令

创建 `~/.claude/commands/log-error.md`：

```markdown
---
description: Log a Claude Code error for pattern analysis
argument-hint: [failure-mode] [brief-description]
---

我需要为我的学习系统记录一个错误。请帮我填写这个模板：

**Failure Mode**: $1
**Brief Description**: $2

请一次一个地问我以下问题：
1. 触发此问题的确切 Prompt 是什么？
2. 发生时 Context % 是多少？
3. 加载了哪些文件？
4. 你预期的是什么 vs 实际发生了什么？
5. 你认为根本原因是什么？

然后将其格式化为 Markdown 条目，以便我可以将其追加到我的错误日志中。
```

### 错误类别与常见修复

|失败模式 (Failure Mode)|常见原因 (Common Causes)|标准修复 (Standard Fixes)|
|---|---|---|
|**Wrong Output (输出错误)**|Prompt 模糊，缺少约束|添加明确的约束和示例|
|**Hallucination (幻觉)**|询问 Claude 没看过的代码|先 @-mention 具体文件|
|**Refusal (拒绝)**|触发安全机制，请求模棱两可|重新表述，提供更多背景|
|**Infinite Loop (死循环)**|成功标准不明确|明确定义退出条件|
|**Context Rot (Context 腐烂)**|Context >70%，信息过时|`/clear`，用新鲜的 Context 重启|
|**Regression (倒退/回归)**|修好了这个，搞坏了那个|实施前要求先写测试|

-----

## 3. 模式 2：Slash 命令作为轻量级本地应用

### 心智模型

把 Slash 命令（Slash Commands）想象成 **Claude即服务 (Claude-as-a-Service) 的工作流**。它们就像是你可以在 5 分钟内构建的 SaaS 产品：

-   预配置了内置最佳实践的 Prompt
-   用于动态行为的参数处理
-   用于安全的工具限制
-   用于成本/质量权衡的模型选择

### 命令架构

```
~/.claude/commands/          # 个人命令（随处可用）
├── quick/                   # 快速、简单的命令
│   ├── commit.md
│   ├── pr.md
│   └── review.md
├── research/                # 调查类命令
│   ├── deep-dive.md
│   ├── compare.md
│   └── audit.md
└── workflows/               # 多步骤流程
    ├── feature.md
    ├── debug.md
    └── refactor.md

.claude/commands/            # 项目命令（团队共享）
├── test.md
├── deploy.md
└── docs.md
```

### 必备 Slash 命令集

#### 1. 智能提交 (`~/.claude/commands/commit.md`)

```markdown
---
description: Create a semantic commit with conventional format
allowed-tools: Bash(git diff:*), Bash(git status:*), Bash(git add:*), Bash(git commit:*)
model: claude-3-5-haiku-20241022
---

分析我暂存（staged）的更改并创建一个提交：

1. 运行 `git diff --cached` 查看暂存的更改
2. 运行 `git status` 了解范围
3. 生成遵循 Conventional Commits（约定式提交）的提交信息：
   - feat: 新功能
   - fix: 修复 bug
   - refactor: 代码重构
   - docs: 文档
   - test: 添加测试
   - chore: 维护杂项

4. 格式：`type(scope): brief description`
   - 保持在 72 个字符以内
   - 使用祈使语气（用 "add" 而不是 "added"）

5. 请求我确认，然后提交

如果没有暂存的更改，先帮我暂存相关文件。
```

#### 2. PR 创建器 (`~/.claude/commands/pr.md`)

```markdown
---
description: Create a comprehensive pull request
allowed-tools: Bash(git:*), Bash(gh:*)
argument-hint: [base-branch]
---

针对 $ARGUMENTS (默认: main) 创建一个 Pull Request：

1. 运行 `git log main..HEAD --oneline` 查看提交
2. 运行 `git diff main...HEAD --stat` 查看变更摘要
3. 生成包含以下内容的 PR：
   - Title: 符合主要变更的常规格式
   - Description:
     ## Summary
     [2-3 句话说明做了什么以及为什么]

     ## Changes
     - [关键变更的要点列表]

     ## Testing
     - [这是如何测试的]

     ## Screenshots
     [如果有 UI 变更，注明应添加截图]

4. 使用 `gh pr create` 和生成的内容创建 PR
5. 完成后输出 PR 的 URL
```

#### 3. 深度调查 (`~/.claude/commands/investigate.md`)

```markdown
---
description: Deep dive into a bug or behavior
allowed-tools: Read, Grep, Glob, Bash(git log:*), Bash(git blame:*)
argument-hint: [issue-description]
---

调查：$ARGUMENTS

遵循这个系统化流程：

## 第一阶段：理解
- 预期行为是什么？
- 实际行为是什么？
- 什么时候开始的？（如果相关，检查 git log）

## 第二阶段：定位
- 使用 Grep 搜索相关代码
- 从入口点追踪代码路径
- 识别所有涉及的文件

## 第三阶段：分析
- 使用 git blame 了解历史
- 寻找可能导致此问题的近期更改
- 检查其他地方是否有相关问题/模式

## 第四阶段：报告
提供结构化报告：
1. **Root Cause (根本原因)**: [一句话]
2. **Affected Files (受影响文件)**: [列表]
3. **Recommended Fix (推荐修复)**: [方法]
4. **Risk Assessment (风险评估)**: [什么可能会坏]
5. **Test Plan (测试计划)**: [如何验证]

不要做任何更改。仅限调查。
```

#### 4. 测试生成器 (`~/.claude/commands/test.md`)

```markdown
---
description: Generate comprehensive tests for a file or function
allowed-tools: Read, Write, Bash(npm test:*), Bash(pytest:*)
argument-hint: [file-or-function]
---

为以下内容生成测试：$ARGUMENTS

## 流程：

1. **阅读目标代码** - 理解所有路径和边缘情况
2. **确定测试框架** - 检查现有测试、package.json、pytest.ini
3. **生成覆盖以下内容的测试**：
   - Happy path (正常使用)
   - Edge cases (空值、null、边界值)
   - Error cases (无效输入、异常)
   - Integration points (外部依赖的 mock)

4. **遵循现有模式** - 匹配现有测试的风格
5. **运行测试** - 确保它们通过
6. **报告覆盖率缺口** - 还有什么未测试？

输出格式：遵循项目惯例创建测试文件。
```

#### 5. 安全审计 (`~/.claude/commands/security-audit.md`)

```markdown
---
description: Security-focused code review
allowed-tools: Read, Grep, Glob
argument-hint: [file-or-directory]
---

对以下内容执行安全审计：$ARGUMENTS

## 检查：

### 输入验证
- [ ] SQL 注入漏洞
- [ ] XSS (跨站脚本)
- [ ] 命令注入
- [ ] 路径遍历

### 认证与授权
- [ ] 硬编码凭据
- [ ] 薄弱的会话处理
- [ ] 缺少权限检查
- [ ] 提权路径

### 数据处理
- [ ] 日志中的敏感数据
- [ ] 未加密的密钥
- [ ] PII (个人隐私信息) 暴露
- [ ] 不安全的反序列化

### 依赖项
- [ ] 已知的易受攻击包
- [ ] 过时的依赖项
- [ ] 未使用的依赖项

## 输出格式：
对于每个发现：
- **Severity (严重性)**: Critical / High / Medium / Low
- **Location (位置)**: file:line
- **Issue (问题)**: 描述
- **Recommendation (建议)**: 如何修复
- **Reference (参考)**: CWE 或 OWASP 链接（如果适用）
```

#### 6. 重构规划器 (`~/.claude/commands/refactor-plan.md`)

```markdown
---
description: Plan a refactoring without executing it
allowed-tools: Read, Grep, Glob
argument-hint: [what-to-refactor]
---

规划重构：$ARGUMENTS

## 不要进行任何更改 - 仅规划

### 分析阶段：
1. 阅读所有相关文件
2. 映射依赖关系（什么导入了这个？这个导入了什么？）
3. 确定测试覆盖率
4. 记录潜在的破坏性更改

### 输出重构计划：

```markdown
# Refactoring Plan: [title]

## Current State (当前状态)
[描述当前架构/代码]

## Desired State (目标状态)
[描述目标架构/代码]

## Steps (步骤 - 按顺序)
1. [ ] Step 1 - [描述] - 风险: Low/Med/High
2. [ ] Step 2 - [描述] - 风险: Low/Med/High
...

## Files to Modify (需修改的文件)
- `path/to/file.ts` - [什么变更]
- `path/to/other.ts` - [什么变更]

## Files to Create (需创建的文件)
- `path/to/new.ts` - [用途]

## Files to Delete (需删除的文件)
- `path/to/old.ts` - [为什么删除是安全的]

## Tests to Add/Update (需添加/更新的测试)
- [所需测试变更列表]

## Rollback Plan (回滚计划)
[如果出错如何撤销]

## Estimated Effort (预估工作量)
[时间预估]
```

当我批准后，我将运行单独的命令来执行每个步骤。
```

-----

## 4. 模式 3：用于确定性安全的 Hooks

### `--dangerously-skip-permissions` 的问题

`--dangerously-skip-permissions` 标志非常诱人。它消除了不断的“我可以运行 X 吗？”的提示，实现了真正的“心流”状态。但它之所以被称为“危险”，是因为 Claude 可以：
- 使用 `rm -rf` 删除文件
- 未经审查推送到 git
- 运行任意 shell 命令
- 修改系统文件

### 解决方案：通过 Hooks 建立护栏

Hooks（钩子）允许你在生命周期的各个点拦截 Claude 的动作并应用你自己的规则。模式如下：

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   –dangerously-skip-permissions    +    Safety Hooks       │
│                                                             │
│        (心流状态)                       (安全护栏)          │
│                                                             │
│                    = 安全的 YOLO 模式                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Hook 配置

位置：`~/.claude/settings.json` 或 `.claude/settings.json`

#### 完整的安全 Hook 配置

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/bash-safety.sh"
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/write-safety.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write(*.py)",
        "hooks": [
          {
            "type": "command",
            "command": "python -m black \"$CLAUDE_FILE_PATHS\""
          }
        ]
      },
      {
        "matcher": "Write(*.ts)",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_FILE_PATHS\""
          }
        ]
      },
      {
        "matcher": "Write(*.tsx)",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_FILE_PATHS\" && npx tsc --noEmit \"$CLAUDE_FILE_PATHS\" 2>&1 || echo '⚠️ TypeScript errors'"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/session-summary.sh"
          }
        ]
      }
    ]
  }
}
```

#### Bash 安全 Hook (`~/.claude/hooks/bash-safety.sh`)

```bash
#!/bin/bash

# 从 stdin 读取工具输入
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# 定义危险模式
DANGEROUS_PATTERNS=(
    "rm -rf /"
    "rm -rf ~"
    "rm -rf \$HOME"
    "rm -rf \*"
    "> /dev/sd"
    "mkfs"
    "dd if="
    ":(){:|:&};:"         # Fork炸弹
    "chmod -R 777 /"
    "chown -R"
    "curl.*| bash"
    "wget.*| bash"
    "curl.*| sh"
    "wget.*| sh"
    "git push.*--force"
    "git push.*-f"
    "DROP TABLE"
    "DROP DATABASE"
    "DELETE FROM.*WHERE 1"
    "npm publish"
    "pip upload"
)

# 检查每个模式
for pattern in "${DANGEROUS_PATTERNS[@]}"; do
    if echo "$COMMAND" | grep -qE "$pattern"; then
        echo '{"decision": "block", "reason": "Blocked dangerous command pattern: '"$pattern"'"}'
        exit 0
    fi
done

# 阻止项目目录之外的操作（可选）
PROJECT_DIR=$(pwd)
if echo "$COMMAND" | grep -qE "cd\s+[^.]|cd\s+/(?!home)" ; then
    echo '{"decision": "ask", "reason": "Command navigates outside project directory"}'
    exit 0
fi

# 允许命令继续
exit 0
```

#### 写入安全 Hook (`~/.claude/hooks/write-safety.sh`)

```bash
#!/bin/bash

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# 受保护的路径
PROTECTED_PATTERNS=(
    "^/etc/"
    "^/usr/"
    "^/bin/"
    "^/sbin/"
    "^\\.env$"
    "^\\.env\\."
    "^.*\\.pem$"
    "^.*\\.key$"
    "^.*_rsa$"
    "^package-lock\\.json$"
    "^yarn\\.lock$"
    "^pnpm-lock\\.yaml$"
)

for pattern in "${PROTECTED_PATTERNS[@]}"; do
    if echo "$FILE_PATH" | grep -qE "$pattern"; then
        echo '{"decision": "ask", "reason": "Protected file: '"$FILE_PATH"'"}'
        exit 0
    fi
done

exit 0
```

#### 会话摘要 Hook (`~/.claude/hooks/session-summary.sh`)

```bash
#!/bin/bash

# 记录会话结束和摘要
TIMESTAMP=$(date +%Y-%m-%d_%H:%M:%S)
LOG_FILE="$HOME/.claude/session-logs/$TIMESTAMP.log"

mkdir -p "$HOME/.claude/session-logs"

# 如果在 repo 中，获取 git diff
if git rev-parse --git-dir > /dev/null 2>&1; then
    echo "=== Session End: $TIMESTAMP ===" >> "$LOG_FILE"
    echo "" >> "$LOG_FILE"
    echo "=== Files Changed ===" >> "$LOG_FILE"
    git diff --name-only >> "$LOG_FILE"
    echo "" >> "$LOG_FILE"
    echo "=== Diff Summary ===" >> "$LOG_FILE"
    git diff --stat >> "$LOG_FILE"
fi

exit 0
```

### 使 Hooks 可执行

```bash
chmod +x ~/.claude/hooks/*.sh
```

### “安全 YOLO” 别名

添加到你的 shell 配置 (`~/.bashrc` 或 `~/.zshrc`)：

```bash
# 安全 YOLO 模式 - 跳过权限检查但 Hooks 会保护你
alias claude-yolo="claude --dangerously-skip-permissions"

# 额外安全 - 还在 Docker 中运行
alias claude-sandbox="docker run -it -v $(pwd):/workspace anthropic/claude-code --dangerously-skip-permissions"
```

-----

## 5. 模式 4：Context 卫生管理

### 理解 Context 衰减

Context 不仅仅是关于耗尽 token——质量在达到限制之前很久就开始下降：

|Context %|质量|建议|
|---|---|---|
|0-40%|极佳|注意力集中，回忆能力强|
|40-60%|良好|开始对新 Context 有所选择|
|60-80%|衰减|“中间迷失”效应开始出现|
|80-95%|差|频繁犯错，遗忘指令|
|95-100%|危急|触发自动压缩，Context 丢失|

### 禁用自动压缩 (Auto-Compact)

自动压缩很方便但不透明。你无法控制什么被总结，关键细节可能会丢失。

**建议**：手动管理压缩。

添加到你的 `CLAUDE.md`：

```markdown
## Context 管理规则

- NEVER auto-compact (绝对不要自动压缩)。我将手动管理 Context。
- 当 Context 超过 70% 时，立即警告我。
- 当超过 50% 时，在你的回复中包含 "Context: XX%" 的提示。
```

### `/context` 状态模式

创建 `~/.claude/commands/status.md`：

```markdown
---
description: Show current context status and recommendations
---

提供 Context 状态报告：

## Context Status (Context 状态)
- **Current Usage**: [X]% of context window
- **Estimated Tokens**: [X]k / 200k
- **Files in Context**: [主要文件列表]
- **Conversation Turns**: [X]

## Health Assessment (健康评估)
- [ ] Under 50%: ✅ 健康
- [ ] 50-70%: ⚠️ 考虑尽快清理
- [ ] Over 70%: 🔴 建议清理

## Recommendations (建议)
[如果超过 50%，建议可以清理或压缩什么]

## Key Context to Preserve (需保留的关键 Context)
[列出在 /clear 操作后应该保留的最重要事项]
```

### “双击 Esc 时间旅行”模式

这是 Claude Code 中利用率最低的功能：

1.  **按一次 Escape**：中断当前 Claude 操作
2.  **按两次 Escape**：打开 **Rewind (倒带)** 界面

Rewind 界面允许你：

- 查看包含 diff（差异）的完整对话历史
- 选择任意之前的点进行恢复
- 恢复代码和对话状态
- 探索替代方案而不丢失工作

### Context 管理工作流

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   1. Start Fresh (全新开始): /clear                         │
│      └── 以干净的 Context 开始                              │
│                                                             │
│   2. Work Phase (工作阶段): 写代码，迭代                    │
│      └── 监控: "我的 Context 使用率是多少？"                │
│                                                             │
│   3. Checkpoint (检查点 - 50%): 记录状态                    │
│      └── 将重要决定保存到 CLAUDE.md 或笔记中                │
│                                                             │
│   4. Continue or Clear (继续或清理 - 70%):                  │
│      ├── 选项 A: 带明确指令的 /compact                      │
│      └── 选项 B: /clear + 从笔记恢复                        │
│                                                             │
│   5. Emergency (紧急情况): 双击 Esc                         │
│      └── 倒带回任意之前的状态                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 智能压缩命令

创建 `~/.claude/commands/smart-compact.md`：

```markdown
---
description: Compact context with explicit preservation rules
---

执行智能压缩：

## MUST PRESERVE (必须保留 - 永不总结掉):
1. 当前任务/目标
2. 过去 10 条消息中提到的所有文件路径
3. 我陈述的任何明确决定或约束
4. 错误信息及其解决方案
5. 当前计划/清单（如果存在）

## CAN SUMMARIZE (可以总结):
1. 导致死胡同的探索
2. 命令的冗长输出（只保留结论）
3. 未被修改的文件内容
4. 导致决策的一般性讨论（只保留决策）

## FORMAT (格式):
压缩后，在你的下一条消息开头加上：
```
 
📦 Context compacted. Preserved:
 
- [关键项 1]
- [关键项 2]
- [当前目标]
 
```
现在带着这些规则执行 /compact。
```

-----

## 6. 模式 5：子 Agent（智能体）控制

### 默认问题

Claude Code 会为各种任务生成子 Agent，但默认情况下可能会使用更便宜/更快的模型（Sonnet, Haiku），即使你为了 Opus 付了费。这导致：

- “研究”任务的分析质量较低
- 主 Agent 和子 Agent 之间的代码质量不一致
- 当子 Agent 返回糟糕结果时浪费 Context

### 强制使用 Opus 子 Agent

添加到你的全局 `~/.claude/CLAUDE.md`：

```markdown
## Subagent Rules (子 Agent 规则)

- **ALWAYS use Opus for subagents (子 Agent 永远使用 Opus)** 除非我明确要求不这样做
- 子 Agent 应被用于：
  - 研究和调查任务
  - 多个文件的并行分析
  - 变更的代码审查
  - 测试生成
  - 文档编写
- 在生成子 Agent 之前，告诉我它将做什么
- 子 Agent 完成后，简洁地总结其发现
```

### 自定义子 Agent 集合

在 `~/.claude/agents/` 中创建专用子 Agent：

#### 代码审查员 (`~/.claude/agents/code-reviewer.md`)

```markdown
---
name: code-reviewer
description: Expert code review focusing on bugs, security, and maintainability
tools: Read, Grep, Glob
model: opus
---

你是一位资深代码审查员，专长于：
- 安全漏洞
- 性能问题
- 代码可维护性
- 测试缺口

在审查代码时：

1. **第一遍 - 关键问题**
   - 安全漏洞（注入、认证、数据暴露）
   - 可能导致 Bug 的逻辑错误
   - 竞态条件或并发问题

2. **第二遍 - 质量问题**
   - 代码重复
   - 需要重构的复杂函数
   - 缺少错误处理
   - 命名不清

3. **第三遍 - 建议**
   - 性能改进
   - 更好的模式/抽象
   - 文档需求

输出格式：
```

## Critical 🔴

- [file:line] 问题描述

## Important 🟡

- [file:line] 问题描述

## Suggestions 🟢

- [file:line] 建议

```
具体一点。包含行号。提出修复建议。
```

#### 测试编写者 (`~/.claude/agents/test-writer.md`)

```markdown
---
name: test-writer
description: Writes comprehensive tests with edge case coverage
tools: Read, Write, Bash
model: opus
---

你是一位测试专家。在编写测试时：

## 原则
1. **测试行为，而非实现**
2. **每个测试一个断言**（在可行时）
3. **描述性的测试名称**: "should_return_empty_array_when_input_is_null"
4. **Arrange-Act-Assert** 结构
5. **系统性地覆盖边缘情况**

## 边缘情况清单
- [ ] Null/undefined 输入
- [ ] 空字符串/数组/对象
- [ ] 边界值 (0, -1, MAX_INT)
- [ ] 无效类型
- [ ] Unicode/特殊字符
- [ ] 并发访问（如果适用）
- [ ] 错误条件

## 测试结构
```

describe(’[Unit Under Test]’, () => {
describe(’[Method/Function]’, () => {
describe(‘when [condition]’, () => {
it(‘should [expected behavior]’, () => {
// Arrange
// Act  
// Assert
});
});
});
});

```
匹配代码库中现有的测试模式。
```

#### 架构分析师 (`~/.claude/agents/architect.md`)

```markdown
---
name: architect
description: Analyzes codebase architecture and suggests improvements
tools: Read, Grep, Glob
model: opus
---

你是一位软件架构师。在分析代码时：

## 分析框架

### 1. 依赖分析
- 映射模块依赖
- 识别循环依赖
- 找出高耦合组件
- 定位上帝对象 (god objects)/模块

### 2. 模式识别
- 使用了什么架构模式？
- 它们的应用是否一致？
- 缺少什么模式会有帮助？

### 3. 可扩展性评估
- 瓶颈识别
- 水平扩展准备度
- 数据库/存储模式

### 4. 可维护性评分
1-10 分并给出理由：
- 代码组织
- 关注点分离
- 测试覆盖率
- 文档

## 输出格式
```markdown
# Architecture Analysis: [Component/System]

## Current State (当前状态)
[图表或描述]

## Strengths (优势)
- 

## Concerns (担忧)
-

## Recommendations (建议)
1. [优先级] 描述
2. [优先级] 描述

## Suggested Refactoring Roadmap (建议重构路线图)
Phase 1: [速赢项]
Phase 2: [中等工作量]
Phase 3: [重大重构]
```

```

### 编排器模式 (Orchestrator Pattern)

对于复杂任务，使用一个主 Agent + 专用子 Agent：

```
┌─────────────────────────────────────────────────────────────┐
│                     Main Agent (Opus)                       │
│                     - 任务规划                              │
│                     - 决策制定                              │
│                     - 结果综合                              │
└─────────────────────────────────────────────────────────────┘
│
┌───────────────────┼───────────────────┐
│                   │                   │
▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ code-reviewer│     │ test-writer │     │  architect  │
│   (Opus)    │     │   (Opus)    │     │   (Opus)    │
└─────────────┘     └─────────────┘     └─────────────┘
```

### 子 Agent 监控仪表盘

参见 [第 8 节](#8-子-agent-监控仪表盘) 查看完整实现。

---

## 7. 模式 6：重提示（Reprompter）系统

### 手打 Prompt 的问题

- 高质量 Prompt 需要时间输入
- 打字会打断心流状态
- 容易忘记重要的约束条件
- Prompt 质量不一致

### 解决方案：语音 → 结构化管道

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Voice     │────▶│  Clarifying  │────▶│  Structured  │
│  Dictation   │     │  Questions   │     │    Prompt    │
└──────────────┘     └──────────────┘     └──────────────┘
│                     │                     │
│                     │                     │
Raw idea            Refinement            XML-tagged
(杂乱的想法)          (对话澄清)            (最佳化结构)
```

### 实现

#### 1. 语音输入设置

使用你的操作系统听写功能或类似工具：
- macOS: 内置听写 (Fn Fn)
- Windows: Windows 语音识别
- 跨平台: Whisper API, Talon

#### 2. Reprompter 命令 (`~/.claude/commands/reprompt.md`)

```markdown
---
description: Convert rough voice input into a structured prompt
argument-hint: [rough-idea]
---

我会给你一个粗糙的、语音口述的想法。你的工作是：

## Step 1: Understand (理解)
问我澄清性问题以了解：
- 具体目标是什么？
- 约束条件是什么？
- 成功是什么样子的？
- 我需要提供什么 Context？

一次问一个问题。等待我的回答。

## Step 2: Structure (结构化)
一旦你理解了，使用这种格式创建一个结构化的 Prompt：

```xml
<task>
  <goal>[清晰、具体的目标]</goal>

  <context>
    <files>[需阅读的相关文件]</files>
    <background>[关键背景]</background>
  </context>

  <constraints>
    <must>[必须包含的要素]</must>
    <must_not>[禁止的要素]</must_not>
  </constraints>

  <output>
    <format>[预期的输出格式]</format>
    <success_criteria>[如何验证成功]</success_criteria>
  </output>
</task>
```

## Step 3: Confirm (确认)

向我展示结构化的 Prompt，并询问它是否正确捕捉了我的意图。

-----

我的粗糙想法：$ARGUMENTS
```

#### 3. 简单任务的快速重提示

创建 `~/.claude/commands/qr.md` (quick reprompt)：

```markdown
---
description: Quick reprompt - one clarifying question, then execute
argument-hint: [task]
---

任务：$ARGUMENTS

在执行之前，问我**一个**澄清性问题，以确保你理解这项任务最关键的方面。

在我回答后，继续执行。
```

### 示例会话

```
你: /reprompt 我需要重构那个 auth 的东西来使用我们讨论的新模式，你知道就是那个带中间件的

Claude: 我想确保我理解正确。当你说“带中间件的新模式”时，你指的是：
1. 我们在本此会话早些时候讨论的具体中间件模式？
2. 你们代码库中某处记录的模式？
3. 一个通用的中间件认证模式（比如 JWT 验证中间件）？

你: 第 2 个，在 docs/auth-v2.md 里

Claude: 明白了。再问一个问题：这次重构应该：
A) 完全替换现有的 auth 系统
B) 暂时与其并行运行（迁移路径）
C) 仅更新特定组件？

你: A，完全替换

Claude: 这是你的结构化 Prompt：

<task>
  <goal>重构认证系统以使用 docs/auth-v2.md 中定义的模式</goal>

  <context>
    <files>docs/auth-v2.md, src/auth/**, src/middleware/**</files>
    <background>现有 auth 的完全替换 - 无并行运行</background>
  </context>

  <constraints>
    <must>严格遵循 auth-v2.md 中记录的模式</must>
    <must>维护所有现有的受 auth 保护的路由</must>
    <must>包含现有用户会话的迁移</must>
    <must_not>保留任何已弃用的 auth 代码处于活动状态</must_not>
  </constraints>

  <output>
    <format>包含新 auth 实现的修改后的文件</format>
    <success_criteria>所有现有测试通过，添加了新 auth 测试，在 CHANGELOG 中记录</success_criteria>
  </output>
</task>

这是否捕捉了你的意图？准备好继续了吗？
```

-----

## 8. 子 Agent 监控仪表盘

一个基于 React 的简单仪表盘，用于监控子 Agent 活动。

### 安装

```bash
# 创建监控目录
mkdir -p ~/.claude/monitoring
cd ~/.claude/monitoring

# 创建 package.json
npm init -y
npm install react react-dom next tailwindcss
```

### 仪表盘代码 (`~/.claude/monitoring/pages/index.tsx`)

```tsx
import React, { useState, useEffect } from 'react';

interface SubagentLog {
  timestamp: string;
  agentName: string;
  task: string;
  model: string;
  tokensUsed: number;
  duration: number;
  status: 'running' | 'completed' | 'failed';
  result?: string;
}

interface SessionStats {
  totalTokens: number;
  totalDuration: number;
  agentBreakdown: Record<string, number>;
}

export default function Dashboard() {
  const [logs, setLogs] = useState<SubagentLog[]>([]);
  const [stats, setStats] = useState<SessionStats | null>(null);

  useEffect(() => {
    // 每 2 秒轮询一次更新
    const interval = setInterval(() => {
      fetchLogs();
    }, 2000);

    fetchLogs();
    return () => clearInterval(interval);
  }, []);

  const fetchLogs = async () => {
    try {
      const res = await fetch('/api/logs');
      const data = await res.json();
      setLogs(data.logs);
      setStats(data.stats);
    } catch (e) {
      console.error('Failed to fetch logs:', e);
    }
  };

  const getStatusColor = (status: string) => {
    switch (status) {
      case 'running': return 'bg-yellow-500';
      case 'completed': return 'bg-green-500';
      case 'failed': return 'bg-red-500';
      default: return 'bg-gray-500';
    }
  };

  return (
    <div className="min-h-screen bg-gray-900 text-white p-8">
      <h1 className="text-3xl font-bold mb-8">🤖 Subagent Monitor</h1>

      {/* 统计面板 */}
      {stats && (
        <div className="grid grid-cols-3 gap-4 mb-8">
          <div className="bg-gray-800 p-4 rounded-lg">
            <h3 className="text-gray-400 text-sm">Total Tokens</h3>
            <p className="text-2xl font-bold">{stats.totalTokens.toLocaleString()}</p>
          </div>
          <div className="bg-gray-800 p-4 rounded-lg">
            <h3 className="text-gray-400 text-sm">Total Duration</h3>
            <p className="text-2xl font-bold">{(stats.totalDuration / 1000).toFixed(1)}s</p>
          </div>
          <div className="bg-gray-800 p-4 rounded-lg">
            <h3 className="text-gray-400 text-sm">Active Agents</h3>
            <p className="text-2xl font-bold">
              {logs.filter(l => l.status === 'running').length}
            </p>
          </div>
        </div>
      )}

      {/* Agent 细分 */}
      {stats && (
        <div className="bg-gray-800 p-4 rounded-lg mb-8">
          <h3 className="text-lg font-semibold mb-4">Token Usage by Agent</h3>
          <div className="space-y-2">
            {Object.entries(stats.agentBreakdown).map(([agent, tokens]) => (
              <div key={agent} className="flex items-center">
                <span className="w-32 text-sm">{agent}</span>
                <div className="flex-1 bg-gray-700 rounded h-4">
                  <div 
                    className="bg-blue-500 h-4 rounded"
                    style={{ width: `${(tokens / stats.totalTokens) * 100}%` }}
                  />
                </div>
                <span className="w-24 text-right text-sm">
                  {tokens.toLocaleString()}
                </span>
              </div>
            ))}
          </div>
        </div>
      )}

      {/* 活动日志 */}
      <div className="bg-gray-800 rounded-lg overflow-hidden">
        <h3 className="text-lg font-semibold p-4 border-b border-gray-700">
          Activity Log
        </h3>
        <div className="divide-y divide-gray-700 max-h-96 overflow-y-auto">
          {logs.map((log, i) => (
            <div key={i} className="p-4 hover:bg-gray-750">
              <div className="flex items-center justify-between mb-2">
                <div className="flex items-center space-x-3">
                  <span className={`w-2 h-2 rounded-full ${getStatusColor(log.status)}`} />
                  <span className="font-medium">{log.agentName}</span>
                  <span className="text-gray-400 text-sm">({log.model})</span>
                </div>
                <span className="text-gray-400 text-sm">{log.timestamp}</span>
              </div>
              <p className="text-sm text-gray-300 mb-2">{log.task}</p>
              <div className="flex space-x-4 text-xs text-gray-500">
                <span>⏱ {log.duration}ms</span>
                <span>📊 {log.tokensUsed} tokens</span>
              </div>
              {log.result && log.status === 'completed' && (
                <div className="mt-2 p-2 bg-gray-900 rounded text-sm">
                  {log.result.substring(0, 200)}...
                </div>
              )}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

### 日志收集 Hook

添加到你的 hooks 配置以向仪表盘提供数据：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/monitoring/log-subagent.sh start"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/monitoring/log-subagent.sh complete"
          }
        ]
      }
    ]
  }
}
```

### 日志脚本 (`~/.claude/monitoring/log-subagent.sh`)

```bash
#!/bin/bash

ACTION=$1
INPUT=$(cat)
TIMESTAMP=$(date -Iseconds)
LOG_FILE="$HOME/.claude/monitoring/logs/$(date +%Y-%m-%d).json"

mkdir -p "$HOME/.claude/monitoring/logs"

AGENT_NAME=$(echo "$INPUT" | jq -r '.tool_input.description // "unknown"')
MODEL=$(echo "$INPUT" | jq -r '.model // "unknown"')
TASK=$(echo "$INPUT" | jq -r '.tool_input.prompt // ""' | head -c 200)

if [ "$ACTION" = "start" ]; then
    echo "{\"timestamp\": \"$TIMESTAMP\", \"agentName\": \"$AGENT_NAME\", \"model\": \"$MODEL\", \"task\": \"$TASK\", \"status\": \"running\"}" >> "$LOG_FILE"
elif [ "$ACTION" = "complete" ]; then
    RESULT=$(echo "$INPUT" | jq -r '.result // ""' | head -c 500)
    TOKENS=$(echo "$INPUT" | jq -r '.tokens_used // 0')
    echo "{\"timestamp\": \"$TIMESTAMP\", \"agentName\": \"$AGENT_NAME\", \"model\": \"$MODEL\", \"status\": \"completed\", \"result\": \"$RESULT\", \"tokensUsed\": $TOKENS}" >> "$LOG_FILE"
fi

exit 0
```

-----

## 9. 快速参考表

### 键盘快捷键

|快捷键|操作|
|---|---|
|`Escape`|中断当前操作|
|`Escape Escape`|打开 Rewind (时间旅行)|
|`Shift + Tab`|切换 Plan Mode / 自动接受|
|`Ctrl + C`|退出 Claude Code|
|`↑` / `↓`|浏览 Prompt 历史|

### 必备命令

|命令|用途|何时使用|
|---|---|---|
|`/clear`|清理对话|开始新任务，Context 臃肿|
|`/compact`|总结 Context|接近 70% 使用率|
|`/context`|显示 Context 使用率|定期监控|
|`/rewind`|时间旅行界面|犯了错，想撤销|
|`/help`|列出所有命令|忘记命令名称|
|`/model`|切换模型|成本优化，能力需求|
|`/agents`|管理子 Agent|配置自定义 Agent|
|`/permissions`|查看/编辑权限|排查工具访问问题|
|`/hooks`|管理 Hooks|审查/更新安全规则|

### Context 管理阈值

|Context %|状态|行动|
|---|---|---|
|0-40%|🟢 健康|自由工作|
|40-60%|🟡 观察|对新文件保持选择性|
|60-80%|🟠 警戒|考虑压缩|
|80-95%|🔴 危急|`/clear` 或有针对性的 `/compact`|
|95-100%|⛔ 危险|自动压缩触发（不可控）|

### 模型选择指南

|模型|最适合|成本|速度|
|---|---|---|---|
|Opus 4.5|架构、复杂推理、关键代码|$$$|慢|
|Sonnet 4.5|大多数编码任务、平衡型|$$|中|
|Haiku 4.5|快速查询、简单任务、探索|$|快|

### 文件位置

|路径|用途|
|---|---|
|`~/.claude/CLAUDE.md`|全局指令|
|`~/.claude/settings.json`|全局设置，hooks|
|`~/.claude/commands/`|个人 Slash 命令|
|`~/.claude/agents/`|自定义子 Agent|
|`.claude/CLAUDE.md`|项目指令（团队）|
|`.claude/settings.json`|项目设置（团队）|
|`.claude/settings.local.json`|本地项目设置 (gitignored)|
|`.claude/commands/`|项目 Slash 命令|
|`.claude/agents/`|项目子 Agent|

### 安全 Hook 检查清单

|Hook 类型|推荐用途|
|---|---|
|`PreToolUse:Bash`|阻止危险命令|
|`PreToolUse:Write`|保护敏感文件|
|`PostToolUse:Write(*.py)`|自动格式化 Python|
|`PostToolUse:Write(*.ts)`|自动格式化 + 类型检查|
|`PostToolUse:Edit`|运行 Linters|
|`Stop`|会话摘要记录|

### Prompt 质量检查清单

在发送 Prompt 之前，请验证：

- [ ] **目标具体**：到底应该发生什么？
- [ ] **提供 Context**：Claude 需要哪些文件/信息？
- [ ] **约束明确**：必须/必须不发生什么？
- [ ] **成功标准定义**：我如何知道它完成了？
- [ ] **给出示例**：对于复杂模式，展示它而不是讲述它

### 错误恢复流程图

```
Issue Detected (检测到问题)
      │
      ▼
  Small/Local? (小的/局部的?) ────Yes────▶ Escape Escape (Rewind)
      │                                       │
      No                                选取恢复点
      │                                       │
      ▼                                       ▼
  Code broken? (代码坏了?) ────Yes────▶ git checkout / git stash
      │
      No
      │
      ▼
  Context rot? (Context 腐烂?) ────Yes────▶ /clear + 带笔记重启
      │
      No
      │
      ▼
  Log to error journal for pattern analysis (记录到错误日志进行模式分析)
```

-----

## 附录：CLAUDE.md 模板

将其复制到你的项目根目录并根据需要自定义：

```markdown
# Project: [NAME]

## Overview
[2-3 句话描述项目]

## Tech Stack
- Language: [e.g., TypeScript 5.3]
- Framework: [e.g., Next.js 14]
- Database: [e.g., PostgreSQL + Prisma]
- Testing: [e.g., Jest + React Testing Library]

## Key Commands
- `npm run dev` - 启动开发服务器
- `npm run build` - 生产构建
- `npm run test` - 运行测试
- `npm run lint` - 运行 linter

## Project Structure
```

src/
├── app/          # Next.js app router
├── components/   # React components
├── lib/          # Utilities and helpers
├── hooks/        # Custom React hooks
└── types/        # TypeScript types

```
## Coding Standards (编码规范)
- 使用带 hooks 的函数式组件
- 优先使用具名导出 (named exports) 而非默认导出
- 每个文件一个组件
- 测试放在 `__tests__/` 目录中

## Important Patterns (重要模式)
- State management: [描述模式]
- API calls: [描述模式]
- Error handling: [描述模式]

## DO NOT (不要)
- 修改 `node_modules/` 中的文件
- 直接提交到 main 分支
- 跳过新功能的测试

## Subagent Rules (子 Agent 规则)
- 子 Agent 始终使用 Opus
- 生成子 Agent 前先通告
- 简洁总结子 Agent 的结果

## Context Management (Context 管理)
- 当 Context 超过 60% 时警告
- 当 >50% 时，在回复中包含 "Context: X%"
- 未经询问绝不自动压缩 (auto-compact)
```

-----

*指南版本 1.0 | 最后更新：2026年1月*
