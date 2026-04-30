# Outline

## Agent

* Agent为什么要每次会话都生成一个新对话？

* Agent 对象什么时候销毁？

* 为什么要无状态？

### SubAgent

首次初始化发生在 initializeApp

五个内置核心子Agent

````JSON
[
    {
        "name": "general-purpose",
        "description": "General-purpose agent for researching complex questions, searching for code, and executing multi-step tasks. When you are searching for a keyword or file and are not confident that you will find the right match in the first few tries use this agent to perform the search for you.",
        "tools": []
    },
    {
        "name": "Explore",
        "description": "Fast agent specialized for exploring codebases. Use this when you need to quickly find files by patterns (eg. \"src/components/**/*.tsx\"), search code for keywords (eg. \"API endpoints\"), or answer questions about the codebase (eg. \"how do API endpoints work?\"). When calling this agent, specify the desired thoroughness level: \"quick\" for basic searches, \"medium\" for moderate exploration, or \"very thorough\" for comprehensive analysis across multiple locations and naming conventions.",
        "tools": [
            "Glob",
            "Grep",
            "Read",
            "WebFetch",
            "WebSearch"
        ],
        "systemPrompt": "# Explore Subagent\n\nYou are a specialized code exploration agent. Your job is to **directly execute searches** using the tools available to you.\n\n## CRITICAL RULES\n\n1. **YOU ARE THE EXECUTOR** - Do NOT delegate to other agents. Use your tools (Glob, Grep, Read) directly.\n2. **NO TASK TOOL** - You do NOT have access to the Task tool. Do not attempt to call it.\n3. **DISCOVER BEFORE READ** - Always use Glob first to find what files exist. NEVER guess file paths.\n4. **NO ASSUMPTIONS** - Don't assume file names exist. Search first!\n\n## Your Available Tools\n\n| Tool | Purpose | Example |\n|------|---------|---------|\n| **Glob** | Find files by pattern | `**/*.tsx`, `src/**/*.ts` |\n| **Grep** | Search code content | `useState`, `class.*Component` |\n| **Read** | Read file contents | After finding files with Glob |\n| **WebFetch** | Fetch URL content | Documentation URLs |\n| **WebSearch** | Search the web | External information |\n\n## Workflow\n\n1. Glob(\"*\") -> Discover root structure\n2. Glob(\"src/**/*\") -> Map source directory\n3. Grep(\"keyword\") -> Find relevant code\n4. Read(found_file) -> Examine details\n5. Return comprehensive summary\n\n## Thoroughness Levels\n\n- **quick**: 1-2 Glob searches, read key files only\n- **medium**: Multiple Glob patterns, Grep for keywords, read 5-10 files\n- **very thorough**: Exhaustive search, all patterns, read all relevant files\n\nRemember: Execute searches directly. Return ONE comprehensive message to the parent agent."
    },
    {
        "name": "Plan",
        "description": "Software architect agent for designing implementation plans. Use this when you need to plan the implementation strategy for a task. Returns step-by-step plans, identifies critical files, and considers architectural trade-offs.",
        "tools": [],
        "systemPrompt": "# Plan Subagent\n\nYou are a software architect specializing in implementation planning.\n\n## Your Role\n\n1. Analyze requirements thoroughly\n2. Explore the codebase to understand existing patterns\n3. Design step-by-step implementation plans\n4. Identify critical files and dependencies\n5. Consider architectural trade-offs\n\n## Output Format\n\n### Implementation Plan\n\n**Goal**: [Clear statement of what will be implemented]\n\n**Critical Files**:\n- `path/to/file.ts` - [Why it's important]\n\n**Steps**:\n1. [Step with specific actions]\n2. [Step with specific actions]\n...\n\n**Trade-offs Considered**:\n- Option A vs Option B: [Reasoning]\n\n**Risks**:\n- [Potential issue and mitigation]\n\nBe thorough but concise. Focus on actionable steps."
    },
    {
        "name": "statusline-setup",
        "description": "Use this agent to configure the user's Claude Code status line setting.",
        "tools": [
            "Read",
            "Edit"
        ]
    },
    {
        "name": "verification",
        "description": "Independent verification agent that validates implementation by running builds, tests, linters, and adversarial probes. Strictly read-only — cannot modify code. Use after completing implementation to get an independent quality assessment.",
        "tools": [
            "Read",
            "Glob",
            "Grep",
            "Bash"
        ],
        "systemPrompt": "# Verification Agent\n\nYou are an **independent verification engineer**. Your sole purpose is to find problems — not to praise or reassure. You are the last line of defense before code ships.\n\n## Constraints\n\n1. **READ-ONLY**: You have NO write tools (no Edit, Write, or NotebookEdit). You cannot modify files. If you discover issues, report them — do not attempt to fix them.\n2. **NO SUB-AGENTS**: You must not delegate to other agents or use the Task tool. Execute all verification steps yourself using your tools directly.\n3. **TOOL-BASED EVIDENCE ONLY**: Every claim must be backed by actual tool output. Never say \"looks correct\" or \"should work\" — run the command and prove it.\n4. **NO ASSUMPTIONS**: Do not assume tests pass. Do not assume types are correct. Run the checks.\n\n## Verification Workflow\n\nExecute these phases in order. Do NOT skip any phase.\n\n### Phase 1: Project Setup Detection\n\n1. Use Glob to find project config files: `package.json`, `tsconfig.json`, `biome.json`, `.eslintrc.*`, `vitest.config.*`, `jest.config.*`, `Makefile`, `Cargo.toml`, `go.mod`, etc.\n2. Use Read to examine them and determine:\n   - Package manager (bun/npm/pnpm/yarn)\n   - Available scripts (test, lint, type-check, build)\n   - Project language and framework\n3. Identify which checks are available for this project.\n\n### Phase 2: Automated Checks\n\nRun all applicable checks. Capture full output.\n\n| Check | Typical Command | Priority |\n|-------|----------------|----------|\n| **Type checking** | `bun run type-check` or `npx tsc --noEmit` | HIGH |\n| **Tests** | `bun run test:all` or `npm test` | HIGH |\n| **Linting** | `bun run lint` or `npx biome check` | HIGH |\n| **Build** | `bun run build` | MEDIUM |\n\n- If a command fails, record the exact error output.\n- If a command succeeds, record confirmation.\n- Set reasonable timeouts (use Bash timeout parameter).\n\n### Phase 3: Code Review of Changed Files\n\n1. Run `git diff --name-only HEAD~1` (or appropriate range) to identify changed files.\n2. Read each changed file and review for:\n   - **Logic errors**: off-by-one, null/undefined handling, race conditions\n   - **Type safety**: any casts, type assertions, missing null checks\n   - **Error handling**: uncaught exceptions, missing error paths\n   - **Edge cases**: empty arrays, empty strings, boundary values\n   - **Security**: injection risks, credential exposure, unsafe eval\n   - **Code style**: naming conventions, dead code, commented-out code\n\n### Phase 4: Adversarial Analysis\n\nThink like an attacker or a hostile user:\n\n1. **Input validation**: Are all inputs validated? What happens with malformed data?\n2. **Boundary conditions**: What happens at limits? (max length, zero, negative)\n3. **Concurrency**: Are there race conditions or shared mutable state issues?\n4. **Dependency risks**: Are new dependencies trustworthy? Pinned versions?\n5. **Regression potential**: Could these changes break existing functionality?\n\n## Output Format\n\nYou MUST end your response with a structured verification report:\n\n```\n## Verification Result: PASS | FAIL | PARTIAL\n\n### Automated Checks\n- [ ] Type check: PASS/FAIL — [details]\n- [ ] Tests: PASS/FAIL — [details, including test count]\n- [ ] Lint: PASS/FAIL — [details]\n- [ ] Build: PASS/FAIL — [details]\n\n### Code Review Findings\n- [Issue severity: HIGH/MEDIUM/LOW] [file:line] Description\n  Evidence: [exact code or output]\n\n### Adversarial Analysis\n- [Risk level: HIGH/MEDIUM/LOW] Description\n  Impact: [what could go wrong]\n\n### Summary\n[1-3 sentence overall assessment with specific evidence]\n```\n\n### Verdict Rules\n\n- **PASS**: All automated checks pass AND no HIGH severity issues found.\n- **FAIL**: Any automated check fails OR any HIGH severity issue found.\n- **PARTIAL**: All automated checks pass BUT MEDIUM severity issues exist.\n\nBe thorough. Be skeptical. Find the bugs.",
        "source": "builtin"
    }
]
````

| Agent            | 定位    | 工具范围                                            | 是否可以修改代码  | 特点     |
| :--------------- | :---- | :---------------------------------------------- | :-------- | :----- |
| general-purpose  | 万能通用  | 全部                                              | ✅         | <br /> |
| Explore          | 代码搜索  | 'Glob', 'Grep', 'Read', 'WebFetch', 'WebSearch' | ❌         | <br /> |
| Plan             | 架构规划  | 全部                                              | ✅（但职责是规划） | <br /> |
| statusline-setup | 状态栏配置 | 'Read', 'Edit'                                  | ✅（仅配置文件）  | <br /> |
| verification     | 验证    | 'Read', 'Glob', 'Grep', 'Bash'                  | <br />    | <br /> |

### **1. general-purpose — 通用万能 Agent**

* **职责**：通用型 agent，用于**研究复杂问题、搜索代码、执行多步骤任务**

* **工具**：`tools: []`，即拥有**所有工具**的访问权限

* **使用场景**：当你搜索关键词/文件但不确定前几次能找到正确匹配时，交给它来执行搜索

* **特点**：没有专门的 systemPrompt，是最灵活的 agent

***

### **2. Explore — 代码探索专家**

* **职责**：**快速探索代码库**的专用 agent

* **工具**：仅限 `Glob`、`Grep`、`Read`、`WebFetch`、`WebSearch`（只读工具，不能修改代码）

* **使用场景**：

  * 按模式查找文件（如 `src/components/**/*.tsx`）

  * 按关键词搜索代码（如 "API endpoints"）

  * 回答关于代码库的问题（如 "API 端点是怎么工作的？"）

* **特点**：有详细的 `systemPrompt`，定义了三个**搜索深度等级**：

  * **quick**：1-2 次 Glob，只看关键文件

  * **medium**：多种 Glob 模式 + Grep，读 5-10 个文件

  * **very thorough**：穷举式搜索，读所有相关文件

***

### **3. Plan — 架构规划师**

* **职责**：**设计实现方案**的软件架构 agent

* **工具**：`tools: []`，拥有**所有工具**（因为规划时可能需要探索代码库）

* **使用场景**：需要为一个任务制定实现策略时

* **特点**：有专门的 `systemPrompt`，输出格式固定为：

  * **Goal**（目标）

  * **Critical Files**（关键文件）

  * **Steps**（分步实现计划）

  * **Trade-offs**（架构权衡）

  * **Risks**（风险与缓解措施）

***

### **4. statusline-setup — 状态栏配置 Agent**

* **职责**：专门用于**配置用户的 Claude Code 状态栏设置**

* **工具**：仅限 `Read` 和 `Edit`（读取和编辑配置文件）

* **使用场景**：用户需要配置状态栏显示时

* **特点**：最简单的 agent，没有自定义 systemPrompt，功能很单一

### **5. verification — 独立验证工程师 🔍**

* **职责**：在代码实现完成后，进行**独立的质量评估**，是代码上线前的"最后一道防线"

* **工具**：`Read`、`Glob`、`Grep`、`Bash`（**严格只读，不能修改代码**）

* **核心原则**：**只找问题，不修问题**。发现 bug 只报告，不尝试修复

#### **验证流程分为 4 个阶段：**

1. **Phase 1 — 项目配置检测**

   * 通过 Glob 找 `package.json`、`tsconfig.json`、`biome.json` 等配置文件

   * 识别包管理器（bun/npm/pnpm/yarn）、可用脚本、项目语言和框架
2. **Phase 2 — 自动化检查**（通过 Bash 执行）

   * **类型检查**（`tsc --noEmit`）— 高优先级

   * **测试**（`bun run test:all`）— 高优先级

   * **Lint**（`biome check`）— 高优先级

   * **构建**（`bun run build`）— 中优先级
3. **Phase 3 — 代码审查**

   * 通过 `git diff` 找到变更文件，逐一审查：

   * 逻辑错误、类型安全、错误处理、边界情况、安全问题、代码风格
4. **Phase 4 — 对抗性分析**（像攻击者一样思考）

   * 输入验证、边界条件、并发竞态、依赖风险、回归风险

***

## Bootstrap

> 首次启动时，初始化了哪些关键程序？扫描了哪些配置？

1. 首次启动时，先检查是否存在新版本，如果存在新版本则弹窗让用户选择是否更新，直接打断后续流程

   * 先读当前agent 的版本，从 package.json中获取，获取不到版本号直接不比对了

   * 接着查找远程版本，优先读缓存\~/.blade/version-cache.json，如果没有缓存则从npm获取最新版本并写入缓存

     ```JSON
     {
         "latestVersion": "0.3.5",
         "checkedAt": 1777361186455,
         "skipUntilVersion": false
     }
     ```

   * 如果获取不到远程npmjs的最新版本，同样也跳过版本检查，否则通过 semver.gt 比较版本，如果存在新版本并且缓存中 skipUntilVersion 的值不为true，则弹窗
2. 如果不存在新版本，或者用户完成更新，则在更新回调中初始化整个APP

   1. 初始化全局配置，这时候全局配置已经初始化完毕了，首次初始化在 yargs loadConfiguration 中间件中完成，这时候已经可以拿到完整的用户配置

      ```JSON
      {
          "currentModelId": "ooXJ71ZEiPC_ccXDjmEEh",
          "models": [
              {
                  "id": "ooXJ71ZEiPC_ccXDjmEEh",
                  "provider": "openai-compatible",
                  "name": "GLM-5.1",
                  "model": "glm-5.1",
                  "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4/",
                  "apiKey": "a55475ca1e8e47b284e7b6b4484ccf57.MpPtCrhoZx90rrSs",
                  "maxContextTokens": 200000,
                  "maxOutputTokens": 131072,
                  "providerId": "zhipuai-coding-plan"
              },
              {
                  "id": "i_C2xY5y15w0v8rD_IOVr",
                  "provider": "openai-compatible",
                  "name": "GLM-5",
                  "model": "glm-5",
                  "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4/",
                  "apiKey": "a55475ca1e8e47b284e7b6b4484ccf57.MpPtCrhoZx90rrSs",
                  "maxContextTokens": 200000,
                  "maxOutputTokens": 131072,
                  "providerId": "zhipuai-coding-plan"
              },
              {
                  "id": "PmPyRQPX7x5TW5zALo7yf",
                  "provider": "openai-compatible",
                  "name": "GLM-4.7",
                  "model": "glm-4.7",
                  "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4/",
                  "apiKey": "d711b990d9a249df82d64a07952df556.1qBdIZSWK7dTnr2X"
              }
          ],
          "temperature": 0,
          "maxContextTokens": 128000,
          "stream": true,
          "topP": 0.9,
          "topK": 50,
          "timeout": 180000,
          "theme": "kanagawa",
          "uiTheme": "light",
          "language": "zh-CN",
          "fontSize": 14,
          "autoSaveSessions": true,
          "notifyBuild": true,
          "notifyErrors": false,
          "notifySounds": true,
          "privacyTelemetry": false,
          "privacyCrash": true,
          "debug": true,
          "mcpEnabled": true,
          "mcpServers": {},
          "permissions": {
              "allow": [
                  "Bash(pwd)",
                  "Bash(which *)",
                  "Bash(whoami)",
                  "Bash(hostname)",
                  "Bash(uname *)",
                  "Bash(date)",
                  "Bash(echo *)",
                  "Bash(ls *)",
                  "Bash(tree *)",
                  "Bash(git status)",
                  "Bash(git status -*)",
                  "Bash(git log *)",
                  "Bash(git diff *)",
                  "Bash(git show *)",
                  "Bash(git branch -* *)",
                  "Bash(git branch)",
                  "Bash(git tag -l *)",
                  "Bash(git tag --list *)",
                  "Bash(git stash list *)",
                  "Bash(git stash show *)",
                  "Bash(git rev-parse *)",
                  "Bash(git describe *)",
                  "Bash(git blame *)",
                  "Bash(git ls-files *)",
                  "Bash(git config --get *)",
                  "Bash(git config --list *)",
                  "Bash(git shortlog *)",
                  "Bash(git merge-base *)",
                  "Bash(git cat-file *)",
                  "Bash(git for-each-ref *)",
                  "Bash(git grep *)",
                  "Bash(git worktree list *)",
                  "Bash(git reflog show *)",
                  "Bash(git reflog)",
                  "Bash(git rev-list *)",
                  "Bash(git ls-remote *)",
                  "Bash(git remote -v)",
                  "Bash(git remote --verbose)",
                  "Bash(git remote)",
                  "Bash(gh pr view *)",
                  "Bash(gh pr list *)",
                  "Bash(gh pr diff *)",
                  "Bash(gh pr checks *)",
                  "Bash(gh pr status *)",
                  "Bash(gh issue view *)",
                  "Bash(gh issue list *)",
                  "Bash(gh issue status *)",
                  "Bash(gh run list *)",
                  "Bash(gh run view *)",
                  "Bash(gh repo view *)",
                  "Bash(gh auth status)",
                  "Bash(npm list *)",
                  "Bash(bun pm ls *)",
                  "Bash(npm view *)",
                  "Bash(npm outdated *)",
                  "Bash(pnpm list *)",
                  "Bash(yarn list *)",
                  "Bash(pip list *)",
                  "Bash(pip show *)"
              ],
              "ask": [
                  "Bash(curl *)",
                  "Bash(wget *)",
                  "Bash(aria2c *)",
                  "Bash(axel *)",
                  "Bash(rm -rf *)",
                  "Bash(rm -r *)",
                  "Bash(rm --recursive *)",
                  "Bash(nc *)",
                  "Bash(netcat *)",
                  "Bash(telnet *)",
                  "Bash(ncat *)"
              ],
              "deny": [
                  "Read(./.env)",
                  "Read(./.env.*)",
                  "Bash(rm -rf /)",
                  "Bash(rm -rf /*)",
                  "Bash(sudo *)",
                  "Bash(chmod 777 *)",
                  "Bash(bash *)",
                  "Bash(sh *)",
                  "Bash(zsh *)",
                  "Bash(fish *)",
                  "Bash(dash *)",
                  "Bash(eval *)",
                  "Bash(source *)",
                  "Bash(mkfs *)",
                  "Bash(fdisk *)",
                  "Bash(dd *)",
                  "Bash(format *)",
                  "Bash(parted *)",
                  "Bash(open http*)",
                  "Bash(open https*)",
                  "Bash(xdg-open http*)",
                  "Bash(xdg-open https*)"
              ]
          },
          "permissionMode": "yolo",
          "hooks": {
              "enabled": false,
              "defaultTimeout": 60,
              "timeoutBehavior": "ignore",
              "failureBehavior": "ignore",
              "maxConcurrentHooks": 5,
              "PreToolUse": [],
              "PostToolUse": [],
              "PostToolUseFailure": [],
              "PermissionRequest": [],
              "UserPromptSubmit": [],
              "SessionStart": [],
              "SessionEnd": [],
              "Stop": [],
              "SubagentStop": [],
              "Notification": [],
              "Compaction": []
          },
          "env": {},
          "disableAllHooks": false,
          "maxTurns": -1
      }
      ```

   2. 将命令行透传进来的 props 和全局 config 合并，如果同一个配置同时在配置文件配置和在命令行中传递，则命令行传递的参数具有更高的优先级

   3. 更新合并了命令行参数后的配置到全局  store

   4. 如果用户通过 --session-id 指定了会话ID, 则覆盖 store 中的默认的随机 ID, 详见 sessionSlice中 restoreSession 方法，/resume 指令用的也是同一个方法，但是指令相比命令行参数多恢复了UI消息和完整的会话消息

   5. 加载主题：如果用户配置了主题，则set 主题，ThemeManager只初始化一次，初始化时通过 Map 保存全部主题和ColorScheme

   6. 加载五个内置的子agent，每个子agent可以指定model属性（目前未兼容， 只为了兼容 cladue），如果未指定默认继承主agent的模型。加载子用户自定义子agent 配置, 首次扫描用户级别以及项目级别 .claude/agents, .balde/agents 下的子agent，扫描目录下所有md文件，使用 yaml formatter 解析 md 内容，一个子agent 必须包含 name 和 description 属性，否则放弃加载，一个标注的解析后的子agent 大概如下

      ```JSON
        {
            "name": "customer-support",
            "description": "Handle support tickets, FAQ responses, and customer emails. Creates help docs, troubleshooting guides, and canned responses. Use PROACTIVELY for customer inquiries or support documentation.",
            "model": "haiku"
        }
      ```

      子agent markdown的主题内容作为系统提示词，一个完整的子agent 的配置包括

      ```TypeScript
      interface SubagentConfig {
        /** Subagent 唯一标识符 */
        name: string;

        /** 描述（给 LLM 看的能力说明） */
        description: string;

        /** 系统提示模板（可选，支持变量替换） */
        systemPrompt?: string;

        /** 允许的工具列表（空数组 = 所有工具） */
        tools?: string[];

        /** UI 背景颜色（可选，用于视觉区分） */
        color?: SubagentColor;

        /** 子agent文件的实际路径 */
        configPath?: string;

        /**
        * 模型别名（sonnet/opus/haiku）或 'inherit'
        * - inherit: 继承父 Agent 模型（默认）
        * - 注意：Blade 目前不支持多模型，此字段仅用于兼容 Claude Code 配置
        */
        model?: 'sonnet' | 'opus' | 'haiku' | 'inherit' | string;

        /** 权限模式（已映射为 Blade PermissionMode） */
        permissionMode?: PermissionMode;

        /** 自动加载的 skills 列表 */
        skills?: string[];

        /** 配置来源（用于调试和优先级） */
        source?:
          | 'builtin'
          | 'claude-code-user'
          | 'claude-code-project'
          | 'blade-user'
          | 'blade-project'
          | `plugin:${string}`;
      }
      ```

      最终返回全部子agent 的个数

   7. 初始化 HookManager，加载用户配置，注意hook 的开关除了用户配置的开关外，还有更细颗粒度会话级别的开关，由 slash 指令 /slash enable, /slash disable 单独控制，如果用户开启了hook，执行SessionStart hook，SessionStart不需要matcher, 默认执行，遍历所有 SessionStart hook根据 hook type（command， promt， function， http） 分类执行。

## Configuration Management

## MCP

## Tools

## Plugin

## Skills

## Hooks

### HookManager 单例

首次初始化发生在 initializeApp

HookManager内置一份默认配置，将和用户配置合并，hook开关默认关闭，需要手动启用

loadConfig：加载用户配置 `hooks` ，注意 agent 内置了一个 PostToolUse 工具

<br />

```JSON
[
    {
      name: 'builtin:code-review-sensor',
      matcher: { tools: 'Edit|Write' },
      hooks: [
        {
          type: HookType.Prompt,
          prompt:
            '审查此代码变更，重点检查：' +
            '1) 安全漏洞（命令注入、XSS、SQL 注入、硬编码密钥/密码）' +
            '2) 明显的逻辑错误（无限循环、off-by-one、空引用）' +
            '3) 类型安全问题。' +
            '如果发现严重问题，将问题描述放在 hookSpecificOutput.additionalContext 中。' +
            '如果没有严重问题，返回 approve。',
          model: undefined,
          timeout: 15,
        },
      ],
    },
  ]
```

<br />

getMatchingHook: 根据 hook的 matcher属性和hook context 来判断当前hook 是否应该执行， macther有三种匹配模式，1、根据matcher.tools匹配，2根据matcher.path匹配，3、根据matcher.commands匹配

matcher的类型

<br />

```
interface MatcherConfig {
  /** 工具名匹配 (支持精确、管道分隔、正则、或数组) */
  tools?: string | string[];

  /** 文件路径匹配 (glob 模式) */
  paths?: string | string[];

  /** 命令匹配 (正则) */
  commands?: string | string[];
}
```

只有PreToolUse, PostToolUse,PermissionRequest,PostToolUseFailure 这四个钩子需要检查 matcher，其他的钩子默认允许执行

## Spec Mode

## Plan Mode

## SessionRuntime

## Slash Command

* `/init`

## Zustand/Vanilla Store

完整 Store

<br />

```JSON
{
    "session": {
        "sessionId": "OppyrH5ug5s5KTu7qMOub",
        "messages": [],
        "restoredContextMessages": null,
        "restoredVisibleMessageCount": 0,
        "isCompacting": false,
        "currentCommand": null,
        "error": null,
        "isActive": true,
        "tokenUsage": {
            "inputTokens": 0,
            "outputTokens": 0,
            "totalTokens": 0,
            "maxContextTokens": 200000
        },
        "currentThinkingContent": null,
        "thinkingExpanded": false,
        "clearCount": 0,
        "historyExpanded": false,
        "expandedMessageCount": 100,
        "currentStreamingMessageId": null,
        "currentStreamingChunks": [],
        "currentStreamingLines": [],
        "currentStreamingTail": "",
        "currentStreamingLineCount": 0,
        "currentStreamingVersion": 0,
        "finalizingStreamingMessageId": null,
        "actions": {}
    },
    "app": {
        "initializationStatus": "ready",
        "initializationError": null,
        "activeModal": "none",
        "modelEditorTarget": null,
        "todos": [],
        "awaitingSecondCtrlC": false,
        "thinkingModeEnabled": false,
        "subagentProgress": null,
        "actions": {}
    },
    "config": {
        "config": {
            "currentModelId": "ooXJ71ZEiPC_ccXDjmEEh",
            "models": [
                {
                    "id": "ooXJ71ZEiPC_ccXDjmEEh",
                    "provider": "openai-compatible",
                    "name": "GLM-5.1",
                    "model": "glm-5.1",
                    "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4/",
                    "apiKey": "a55475ca1e8e47b284e7b6b4484ccf57.MpPtCrhoZx90rrSs",
                    "maxContextTokens": 200000,
                    "maxOutputTokens": 131072,
                    "providerId": "zhipuai-coding-plan"
                },
                {
                    "id": "i_C2xY5y15w0v8rD_IOVr",
                    "provider": "openai-compatible",
                    "name": "GLM-5",
                    "model": "glm-5",
                    "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4/",
                    "apiKey": "a55475ca1e8e47b284e7b6b4484ccf57.MpPtCrhoZx90rrSs",
                    "maxContextTokens": 200000,
                    "maxOutputTokens": 131072,
                    "providerId": "zhipuai-coding-plan"
                },
                {
                    "id": "PmPyRQPX7x5TW5zALo7yf",
                    "provider": "openai-compatible",
                    "name": "GLM-4.7",
                    "model": "glm-4.7",
                    "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4/",
                    "apiKey": "d711b990d9a249df82d64a07952df556.1qBdIZSWK7dTnr2X"
                }
            ],
            "temperature": 0,
            "maxContextTokens": 128000,
            "stream": true,
            "topP": 0.9,
            "topK": 50,
            "timeout": 180000,
            "theme": "kanagawa",
            "uiTheme": "light",
            "language": "zh-CN",
            "fontSize": 14,
            "autoSaveSessions": true,
            "notifyBuild": true,
            "notifyErrors": false,
            "notifySounds": true,
            "privacyTelemetry": false,
            "privacyCrash": true,
            "debug": true,
            "mcpEnabled": true,
            "mcpServers": {},
            "permissions": {
                "allow": [
                    "Bash(pwd)",
                    "Bash(which *)",
                    "Bash(whoami)",
                    "Bash(hostname)",
                    "Bash(uname *)",
                    "Bash(date)",
                    "Bash(echo *)",
                    "Bash(ls *)",
                    "Bash(tree *)",
                    "Bash(git status)",
                    "Bash(git status -*)",
                    "Bash(git log *)",
                    "Bash(git diff *)",
                    "Bash(git show *)",
                    "Bash(git branch -* *)",
                    "Bash(git branch)",
                    "Bash(git tag -l *)",
                    "Bash(git tag --list *)",
                    "Bash(git stash list *)",
                    "Bash(git stash show *)",
                    "Bash(git rev-parse *)",
                    "Bash(git describe *)",
                    "Bash(git blame *)",
                    "Bash(git ls-files *)",
                    "Bash(git config --get *)",
                    "Bash(git config --list *)",
                    "Bash(git shortlog *)",
                    "Bash(git merge-base *)",
                    "Bash(git cat-file *)",
                    "Bash(git for-each-ref *)",
                    "Bash(git grep *)",
                    "Bash(git worktree list *)",
                    "Bash(git reflog show *)",
                    "Bash(git reflog)",
                    "Bash(git rev-list *)",
                    "Bash(git ls-remote *)",
                    "Bash(git remote -v)",
                    "Bash(git remote --verbose)",
                    "Bash(git remote)",
                    "Bash(gh pr view *)",
                    "Bash(gh pr list *)",
                    "Bash(gh pr diff *)",
                    "Bash(gh pr checks *)",
                    "Bash(gh pr status *)",
                    "Bash(gh issue view *)",
                    "Bash(gh issue list *)",
                    "Bash(gh issue status *)",
                    "Bash(gh run list *)",
                    "Bash(gh run view *)",
                    "Bash(gh repo view *)",
                    "Bash(gh auth status)",
                    "Bash(npm list *)",
                    "Bash(bun pm ls *)",
                    "Bash(npm view *)",
                    "Bash(npm outdated *)",
                    "Bash(pnpm list *)",
                    "Bash(yarn list *)",
                    "Bash(pip list *)",
                    "Bash(pip show *)"
                ],
                "ask": [
                    "Bash(curl *)",
                    "Bash(wget *)",
                    "Bash(aria2c *)",
                    "Bash(axel *)",
                    "Bash(rm -rf *)",
                    "Bash(rm -r *)",
                    "Bash(rm --recursive *)",
                    "Bash(nc *)",
                    "Bash(netcat *)",
                    "Bash(telnet *)",
                    "Bash(ncat *)"
                ],
                "deny": [
                    "Read(./.env)",
                    "Read(./.env.*)",
                    "Bash(rm -rf /)",
                    "Bash(rm -rf /*)",
                    "Bash(sudo *)",
                    "Bash(chmod 777 *)",
                    "Bash(bash *)",
                    "Bash(sh *)",
                    "Bash(zsh *)",
                    "Bash(fish *)",
                    "Bash(dash *)",
                    "Bash(eval *)",
                    "Bash(source *)",
                    "Bash(mkfs *)",
                    "Bash(fdisk *)",
                    "Bash(dd *)",
                    "Bash(format *)",
                    "Bash(parted *)",
                    "Bash(open http*)",
                    "Bash(open https*)",
                    "Bash(xdg-open http*)",
                    "Bash(xdg-open https*)"
                ]
            },
            "permissionMode": "yolo",
            "hooks": {
                "enabled": false,
                "defaultTimeout": 60,
                "timeoutBehavior": "ignore",
                "failureBehavior": "ignore",
                "maxConcurrentHooks": 5,
                "PreToolUse": [],
                "PostToolUse": [],
                "PostToolUseFailure": [],
                "PermissionRequest": [],
                "UserPromptSubmit": [],
                "SessionStart": [],
                "SessionEnd": [],
                "Stop": [],
                "SubagentStop": [],
                "Notification": [],
                "Compaction": []
            },
            "env": {},
            "disableAllHooks": false,
            "maxTurns": -1,
            "outputFormat": "text",
            "inputFormat": "text",
            "print": false
        },
        "actions": {}
    },
    "focus": {
        "currentFocus": "main-input",
        "previousFocus": null,
        "actions": {}
    },
    "command": {
        "isProcessing": false,
        "abortController": null,
        "pendingCommands": [],
        "actions": {}
    },
    "spec": {
        "currentSpec": null,
        "specPath": null,
        "isActive": false,
        "steeringContext": null,
        "recentSpecs": [],
        "isLoading": false,
        "error": null,
        "workspaceRoot": null,
        "actions": {}
    }
}
```

### Slice

#### State

#### Action

### Selector

### Registry

SubagentRegistry

## Memory

## Cli

### Commands

#### Args

支持的参数有

```TypeScript
export const globalOptions = {
  debug: {
    alias: 'd',
    type: 'string',
    describe:
      'Enable debug mode with optional category filtering (e.g., "agent,ui" or "!chat,!loop")',
    group: 'Debug Options:',
    // 过滤逻辑已实现：
    // 正向过滤：--debug "agent,ui" 只显示 Agent 和 UI 类别
    // 负向过滤：--debug "!chat,!loop" 排除 Chat 和 Loop 类别
    // 支持的分类：agent, ui, tool, service, config, context, execution, loop, chat, general
  },
  print: {
    alias: 'p',
    type: 'boolean',
    describe: 'Print response and exit (useful for pipes)',
    group: 'Output Options:',
  },
  headless: {
    type: 'boolean',
    describe: 'Run full agent loop without Ink UI and print all events to the terminal',
    group: 'Output Options:',
  },
  'output-format': {
    alias: ['outputFormat'],
    type: 'string',
    choices: ['text', 'json', 'stream-json', 'jsonl'],
    default: 'text',
    describe:
      'Output format: --print supports text|json|stream-json; --headless supports text|jsonl',
    group: 'Output Options:',
  },
  'include-partial-messages': {
    alias: ['includePartialMessages'],
    type: 'boolean',
    describe: 'Include partial message chunks as they arrive',
    group: 'Output Options:',
  },
  'input-format': {
    alias: ['inputFormat'],
    type: 'string',
    choices: ['text', 'stream-json'],
    default: 'text',
    describe: 'Input format',
    group: 'Input Options:',
  },
  'replay-user-messages': {
    alias: ['replayUserMessages'],
    type: 'boolean',
    describe: 'Re-emit user messages from stdin',
    group: 'Input Options:',
  },
  'allowed-tools': {
    alias: ['allowedTools'],
    type: 'array',
    string: true,
    describe: 'Comma or space-separated list of tool names to allow',
    group: 'Security Options:',
  },
  'disallowed-tools': {
    alias: ['disallowedTools'],
    type: 'array',
    string: true,
    describe: 'Comma or space-separated list of tool names to deny',
    group: 'Security Options:',
  },
  'mcp-config': {
    alias: ['mcpConfig'],
    type: 'array',
    string: true,
    describe: 'Load MCP servers from JSON files or strings',
    group: 'MCP Options:',
  },
  'system-prompt': {
    alias: ['systemPrompt'],
    type: 'string',
    describe: 'System prompt to use for the session (replaces default)',
    group: 'AI Options:',
  },
  'append-system-prompt': {
    alias: ['appendSystemPrompt'],
    type: 'string',
    describe: 'Append a system prompt to the default system prompt',
    group: 'AI Options:',
  },
  'max-turns': {
    alias: ['maxTurns'],
    type: 'number',
    describe:
      'Maximum conversation turns (-1: unlimited, 0: disable chat, N>0: limit to N turns)',
    group: 'AI Options:',
    default: undefined, // 不设置默认值，优先使用配置文件
  },
  'permission-mode': {
    alias: ['permissionMode'],
    type: 'string',
    choices: ['default', 'autoEdit', 'yolo', 'plan'],
    describe:
      'Permission mode (default: ask for non-read tools, autoEdit: auto-approve edits, yolo: auto-approve all, plan: reserved)',
    group: 'Security Options:',
  },
  yolo: {
    type: 'boolean',
    describe: 'Auto-approve all tools (shortcut for --permission-mode=yolo)',
    group: 'Security Options:',
  },
  continue: {
    alias: 'c',
    type: 'boolean',
    describe: 'Continue the most recent conversation',
    group: 'Session Options:',
  },
  resume: {
    alias: 'r',
    describe:
      'Resume a conversation - provide a session ID or interactively select a conversation to resume',
    group: 'Session Options:',
    // 移除 type 声明，让 yargs 自动推断
    coerce: (value: string | boolean | undefined) => {
      // 如果没有提供值（value === undefined 或 true），返回 'true' 表示交互式选择
      if (value === undefined || value === true || value === '') {
        return 'true';
      }
      // 如果提供了值，返回 sessionId
      return String(value);
    },
  },
  'fork-session': {
    alias: ['forkSession'],
    type: 'boolean',
    describe: 'Create a new session ID when resuming',
    group: 'Session Options:',
  },

  // TODO: 未实现 - 需要在 UI/Agent 中读取并使用
  settings: {
    type: 'string',
    describe: 'Path to a settings JSON file or JSON string',
    group: 'Configuration:',
  },
  'add-dir': {
    alias: ['addDir'],
    type: 'array',
    string: true,
    describe: 'Additional directories to allow tool access to',
    group: 'Security Options:',
  },
  // TODO: 未实现 - 需要在 UI 启动时自动连接 IDE
  ide: {
    type: 'boolean',
    describe: 'Automatically connect to IDE on startup',
    group: 'Integration:',
  },
  acp: {
    type: 'boolean',
    describe: 'Run in ACP (Agent Client Protocol) mode for IDE integration',
    group: 'Integration:',
  },
  'strict-mcp-config': {
    alias: ['strictMcpConfig'],
    type: 'boolean',
    describe: 'Only use MCP servers from --mcp-config',
    group: 'MCP Options:',
  },
  'session-id': {
    alias: ['sessionId'],
    type: 'string',
    describe: 'Use a specific session ID for the conversation',
    group: 'Session Options:',
  },
  // TODO: 未实现 - 需要解析 JSON 并配置自定义 Agent
  agents: {
    type: 'string',
    describe: 'JSON object defining custom agents',
    group: 'AI Options:',
  },
  'setting-sources': {
    alias: ['settingSources'],
    type: 'string',
    describe: 'Comma-separated list of setting sources to load',
    group: 'Configuration:',
  },
  'plugin-dir': {
    alias: ['pluginDir'],
    type: 'array',
    string: true,
    describe: 'Load plugins from specified directories',
    group: 'Plugin Options:',
  },
} satisfies Record<string, Options>;
```

## UI

* User Interaction

  * onPaste

    * 文字

    * 图片

***

## Q\&A

1. 在输入框中粘贴图片时，怎么对图片进行处理
2. 非多模态的大模型能否识别图像内容，例如图片中包含部分文本说明或者框选某个区域，大模型如何识别
3. Agent首次初始化，以及为什么要多次实例化，何时被清理
4. Plan, Tools Hooks, MCP 是如何实现的
5. 上下文管理是怎么实现的

restoreSession


```Typescript
#!/usr/bin/env node

/**
 * mobile-debug.mjs — 一键启动移动端调试环境
 *
 * 全自动完成以下流程：
 *   1. 启动 Chrome（带 --remote-debugging-port 和 --auto-open-devtools-for-tabs）
 *   2. 等待 CDP 端口就绪
 *   3. 用 playwright-cli attach 连接到 Chrome
 *   4. 导航到目标 URL（默认 https://www.baidu.com）
 *   5. 等待 docked DevTools 页面加载完成（由 --auto-open-devtools-for-tabs 自动打开）
 *   6. 连接 docked DevTools 页面的 WebSocket，调用内部 API 切换 Device Toolbar
 *
 * 使用方式：
 *   node scripts/mobile-debug.mjs [url] [options]
 *
 * 参数：
 *   url              目标页面 URL（默认 https://www.baidu.com）
 *   --port=<port>    Chrome 调试端口（默认 9222）
 *   --user-data-dir  Chrome 用户数据目录（默认 ./chrome-debug-profile）
 *   --no-device-toolbar  跳过 Device Toolbar 切换（仅打开页面）
 *   --help           显示帮助信息
 *
 * 示例：
 *   node scripts/mobile-debug.mjs
 *   node scripts/mobile-debug.mjs https://www.google.com
 *   node scripts/mobile-debug.mjs https://m.taobao.com --port=9333
 *
 * 前置依赖：
 *   - Google Chrome（macOS）
 *   - Node.js >= 18
 *   - npm install ws（在 scripts/ 目录下）
 *   - npx @playwright/cli（全局可用）
 */

import http from 'node:http';
import { execSync, spawn } from 'node:child_process';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const SCRIPT_DIR = path.dirname(fileURLToPath(import.meta.url));
const CHROME_PATH = '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome';
const DEFAULT_URL = 'https://www.baidu.com';
const DEFAULT_PORT = 9222;
const MAX_WAIT_RETRIES = 30;
const WAIT_INTERVAL_MS = 1000;

// ─── 参数解析 ───────────────────────────────────────────────────────────

function parseArgs() {
  const args = process.argv.slice(2);
  const config = {
    url: DEFAULT_URL,
    port: DEFAULT_PORT,
    userDataDir: path.resolve(SCRIPT_DIR, '..', 'chrome-debug-profile'),
    enableDeviceToolbar: true,
    device: '',
    help: false,
  };

  for (const arg of args) {
    if (arg === '--help' || arg === '-h') {
      config.help = true;
    } else if (arg.startsWith('--port=')) {
      config.port = parseInt(arg.split('=')[1], 10);
    } else if (arg.startsWith('--user-data-dir=')) {
      config.userDataDir = path.resolve(arg.split('=')[1]);
    } else if (arg === '--no-device-toolbar') {
      config.enableDeviceToolbar = false;
    } else if (arg.startsWith('--device=')) {
      config.device = arg.split('=').slice(1).join('=');
    } else if (!arg.startsWith('--')) {
      config.url = arg;
    }
  }

  return config;
}

function printHelp() {
  console.log(`
mobile-debug.mjs — 一键启动移动端调试环境

Usage:
  node scripts/mobile-debug.mjs [url] [options]

Arguments:
  url                    目标页面 URL（默认 ${DEFAULT_URL}）

Options:
  --port=<port>          Chrome 调试端口（默认 ${DEFAULT_PORT}）
  --device=<name>        模拟设备名称（默认选择第一个可用设备，如 "iPhone 14 Pro Max"）
  --user-data-dir=<dir>  Chrome 用户数据目录（默认 ./chrome-debug-profile）
  --no-device-toolbar    跳过 Device Toolbar 切换
  --help, -h             显示帮助信息

Examples:
  node scripts/mobile-debug.mjs
  node scripts/mobile-debug.mjs https://www.google.com
  node scripts/mobile-debug.mjs https://m.taobao.com --port=9333
  node scripts/mobile-debug.mjs https://www.baidu.com --device="iPhone 14 Pro Max"
`);
}

// ─── 工具函数 ───────────────────────────────────────────────────────────

function log(emoji, message) {
  console.log(`${emoji}  ${message}`);
}

function fetchJson(url) {
  return new Promise((resolve, reject) => {
    http.get(url, (res) => {
      let body = '';
      res.on('data', (chunk) => body += chunk);
      res.on('end', () => {
        try { resolve(JSON.parse(body)); }
        catch { reject(new Error(`Invalid JSON from ${url}`)); }
      });
    }).on('error', reject);
  });
}

function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function runPlaywrightCli(args) {
  const command = `npx playwright-cli ${args}`;
  try {
    return execSync(command, { encoding: 'utf-8', timeout: 30000 });
  } catch (error) {
    throw new Error(`playwright-cli failed: ${error.stderr || error.message}`);
  }
}

async function loadWebSocket() {
  try {
    const ws = await import('ws');
    return ws.default || ws.WebSocket;
  } catch {
    console.error('Error: "ws" module not found. Run: cd scripts && npm install ws');
    process.exit(1);
  }
}

function evaluateInDevTools(WebSocket, webSocketUrl, expression) {
  return new Promise((resolve, reject) => {
    const ws = new WebSocket(webSocketUrl);
    const timeout = setTimeout(() => { ws.close(); reject(new Error('Evaluate timed out')); }, 15000);

    ws.on('open', () => {
      ws.send(JSON.stringify({
        id: 1,
        method: 'Runtime.evaluate',
        params: { expression, returnByValue: true }
      }));
    });

    ws.on('message', (data) => {
      const message = JSON.parse(data.toString());
      if (message.id === 1) {
        clearTimeout(timeout);
        if (message.result?.result?.value) {
          try { resolve(JSON.parse(message.result.result.value)); }
          catch { resolve(message.result.result.value); }
        } else {
          reject(new Error('Unexpected evaluate response: ' + JSON.stringify(message)));
        }
        ws.close();
      }
    });

    ws.on('error', (error) => { clearTimeout(timeout); reject(error); });
  });
}

// ─── 步骤实现 ───────────────────────────────────────────────────────────

function checkExistingChrome(port) {
  return fetchJson(`http://localhost:${port}/json/version`)
    .then(() => true)
    .catch(() => false);
}

function launchChrome(port, userDataDir) {
  log('🚀', `Launching Chrome on port ${port}...`);

  const chromeProcess = spawn(CHROME_PATH, [
    `--remote-debugging-port=${port}`,
    '--auto-open-devtools-for-tabs',
    `--user-data-dir=${userDataDir}`,
  ], {
    detached: true,
    stdio: 'ignore',
  });

  chromeProcess.unref();
  log('✅', `Chrome launched (PID: ${chromeProcess.pid})`);
  return chromeProcess.pid;
}

async function waitForCdpReady(port) {
  log('⏳', 'Waiting for CDP endpoint to be ready...');

  for (let attempt = 0; attempt < MAX_WAIT_RETRIES; attempt++) {
    try {
      const version = await fetchJson(`http://localhost:${port}/json/version`);
      log('✅', `CDP ready — Chrome ${version.Browser}`);
      return version;
    } catch {
      await sleep(WAIT_INTERVAL_MS);
    }
  }

  throw new Error(`CDP endpoint on port ${port} did not become ready within ${MAX_WAIT_RETRIES}s`);
}

function attachPlaywrightCli(port) {
  log('🔗', `Attaching playwright-cli to http://localhost:${port}...`);
  const output = runPlaywrightCli(`attach --cdp=http://localhost:${port}`);
  log('✅', 'Attached to Chrome');
  return output;
}

function navigateToUrl(url) {
  log('🌐', `Navigating to ${url}...`);

  // 先获取 tab 列表，找到非 chrome:// 的页面
  const tabListOutput = runPlaywrightCli('tab-list');
  const tabLines = tabListOutput.split('\n').filter(line => line.includes('- '));

  // 找到第一个普通页面 tab（非 chrome:// 和 devtools://），或者用第一个 tab
  let targetTabIndex = -1;
  for (const line of tabLines) {
    const match = line.match(/^- (\d+):/);
    if (match && !line.includes('chrome://omnibox') && !line.includes('devtools://')) {
      targetTabIndex = parseInt(match[1], 10);
      break;
    }
  }

  if (targetTabIndex === -1) {
    // 所有 tab 都是 chrome:// 或 devtools://，选最后一个
    const lastMatch = tabLines[tabLines.length - 1]?.match(/^- (\d+):/);
    targetTabIndex = lastMatch ? parseInt(lastMatch[1], 10) : 0;
  }

  runPlaywrightCli(`tab-select ${targetTabIndex}`);
  const output = runPlaywrightCli(`goto ${url}`);
  log('✅', `Page loaded: ${url}`);
  return output;
}

async function waitForDevToolsPage(port) {
  log('⏳', 'Waiting for DevTools page to be ready...');

  for (let attempt = 0; attempt < MAX_WAIT_RETRIES; attempt++) {
    const pages = await fetchJson(`http://localhost:${port}/json/list`);
    const devtoolsPage = pages.find(page =>
      page.url.startsWith('devtools://') && page.url.includes('devtools_app.html')
    );

    if (devtoolsPage) {
      log('✅', 'DevTools page is ready');
      return devtoolsPage;
    }

    await sleep(WAIT_INTERVAL_MS);
  }

  throw new Error('DevTools page did not appear. Make sure Chrome was started with --auto-open-devtools-for-tabs');
}

async function findDevToolsWebSocket(port) {
  const pages = await fetchJson(`http://localhost:${port}/json/list`);
  const devtoolsPage = pages.find(page => page.url.startsWith('devtools://'));
  if (!devtoolsPage) {
    throw new Error('DevTools page not found in target list');
  }
  return devtoolsPage.webSocketDebuggerUrl;
}

async function toggleDeviceToolbar(port) {
  log('📱', 'Toggling Device Toolbar...');

  const WebSocket = await loadWebSocket();
  const devToolsWsUrl = await findDevToolsWebSocket(port);

  const result = await evaluateInDevTools(
    WebSocket,
    devToolsWsUrl,
    `(function() {
      var app = Emulation.AdvancedApp.instance();
      var deviceModeView = app.deviceModeView;
      var wasOn = deviceModeView.isDeviceModeOn();
      if (!wasOn) {
        deviceModeView.toggleDeviceMode();
      }
      var isNowOn = deviceModeView.isDeviceModeOn();
      return JSON.stringify({ wasOn: wasOn, isNowOn: isNowOn });
    })()`
  );

  if (result.isNowOn) {
    log('✅', 'Device Toolbar is ON');
  } else {
    log('⚠️', `Device Toolbar toggle result: was=${result.wasOn}, now=${result.isNowOn}`);
  }

  return result;
}

async function selectMobileDevice(port, deviceName) {
  log('📲', `Selecting device: ${deviceName || 'first available'}...`);

  const WebSocket = await loadWebSocket();
  const devToolsWsUrl = await findDevToolsWebSocket(port);

  const result = await evaluateInDevTools(
    WebSocket,
    devToolsWsUrl,
    `(function() {
      var app = Emulation.AdvancedApp.instance();
      var toolbar = app.deviceModeView.deviceModeView.toolbar;
      var devices = toolbar.standardDevices();
      var targetName = ${JSON.stringify(deviceName || '')};

      var device;
      if (targetName) {
        device = devices.find(function(d) { return d.title === targetName; });
        if (!device) {
          return JSON.stringify({
            error: 'Device not found: ' + targetName,
            availableDevices: devices.map(function(d) { return d.title; })
          });
        }
      } else {
        device = devices[0];
      }

      toolbar.emulateDevice(device);

      var model = app.deviceModeView.deviceModeView.model;
      return JSON.stringify({
        selectedDevice: device.title,
        currentDevice: model.device() ? model.device().title : 'none',
        currentType: model.type(),
        availableDevices: devices.map(function(d) { return d.title; })
      });
    })()`
  );

  if (result.error) {
    log('⚠️', result.error);
    log('ℹ️', `Available devices: ${result.availableDevices.join(', ')}`);
  } else {
    log('✅', `Device selected: ${result.selectedDevice}`);
  }

  return result;
}

// ─── 主流程 ─────────────────────────────────────────────────────────────

async function main() {
  const config = parseArgs();

  if (config.help) {
    printHelp();
    process.exit(0);
  }

  console.log('');
  console.log('╔═══════════════════════════════════════════════╗');
  console.log('║       📱 Mobile Debug Environment Setup       ║');
  console.log('╚═══════════════════════════════════════════════╝');
  console.log('');

  // Step 1: 检查是否已有 Chrome 实例
  const alreadyRunning = await checkExistingChrome(config.port);
  if (alreadyRunning) {
    log('ℹ️', `Chrome already running on port ${config.port}, reusing...`);
  } else {
    // Step 2: 启动 Chrome
    launchChrome(config.port, config.userDataDir);
    // Step 3: 等待 CDP 就绪
    await waitForCdpReady(config.port);
  }

  // Step 4: Attach playwright-cli
  attachPlaywrightCli(config.port);

  // Step 5: 导航到目标 URL
  navigateToUrl(config.url);

  // Step 6: 等待 DevTools 加载，然后切换 Device Toolbar 并选择设备
  let selectedDeviceName = 'Desktop';
  if (config.enableDeviceToolbar) {
    await waitForDevToolsPage(config.port);
    await toggleDeviceToolbar(config.port);
    const deviceResult = await selectMobileDevice(config.port, config.device);
    selectedDeviceName = deviceResult.selectedDevice || deviceResult.currentDevice || 'Unknown';

    // Step 7: 刷新页面，让移动端样式正确渲染
    log('🔄', 'Reloading page for mobile rendering...');
    runPlaywrightCli('reload');
    log('✅', 'Page reloaded');
  }

  console.log('');
  console.log('╔═══════════════════════════════════════════════╗');
  console.log('║              🎉 All Done!                     ║');
  console.log('╠═══════════════════════════════════════════════╣');
  console.log(`║  URL:    ${config.url.padEnd(37)}║`);
  console.log(`║  Port:   ${String(config.port).padEnd(37)}║`);
  console.log(`║  Device: ${selectedDeviceName.padEnd(37)}║`);
  console.log('╚═══════════════════════════════════════════════╝');
  console.log('');
  console.log('Use playwright-cli to interact:');
  console.log('  npx playwright-cli snapshot     # 获取页面快照');
  console.log('  npx playwright-cli screenshot   # 截图');
  console.log('  npx playwright-cli click <ref>  # 点击元素');
  console.log('  npx playwright-cli goto <url>   # 导航');
  console.log('  npx playwright-cli close        # 关闭浏览器');
  console.log('');
}

main().catch((error) => {
  console.error('');
  console.error('❌ Error:', error.message);
  process.exit(1);
});

#!/usr/bin/env node

/**
 * Toggle Chrome DevTools Device Toolbar via DevTools Internal API
 *
 * 原理：
 *   普通 CDP 命令（如 Emulation.setDeviceMetricsOverride）只能控制页面的运行时行为，
 *   无法控制 DevTools UI 本身。但 DevTools 面板本身也是一个网页（devtools:// 协议），
 *   拥有独立的 WebSocket 调试地址。通过连接到 DevTools 页面的 WebSocket，
 *   我们可以用 Runtime.evaluate 在 DevTools 页面的 JavaScript 上下文中执行代码，
 *   调用 DevTools 的内部 API 来切换 Device Toolbar。
 *
 * 关键 API：
 *   - Emulation.AdvancedApp.instance().deviceModeView.toggleDeviceMode()
 *   - Emulation.AdvancedApp.instance().deviceModeView.isDeviceModeOn()
 *
 * 使用方式：
 *   1. 确保 Chrome 以 --remote-debugging-port=<port> --auto-open-devtools-for-tabs 启动
 *   2. node scripts/toggle-device-toolbar.mjs [port]  (默认 port=9222)
 *
 * 前置依赖：
 *   npm install ws  (或在 /tmp 目录下已安装)
 */

import http from 'node:http';

const CDP_PORT = parseInt(process.argv[2] || '9222', 10);

function fetchJson(url) {
  return new Promise((resolve, reject) => {
    http.get(url, (res) => {
      let body = '';
      res.on('data', (chunk) => body += chunk);
      res.on('end', () => {
        try { resolve(JSON.parse(body)); }
        catch (error) { reject(new Error(`Failed to parse JSON from ${url}: ${body}`)); }
      });
    }).on('error', reject);
  });
}

function findDevToolsWebSocket(pages) {
  const devtoolsPage = pages.find(page => page.url.startsWith('devtools://'));
  if (!devtoolsPage) {
    throw new Error(
      'No DevTools page found. Make sure Chrome is started with --auto-open-devtools-for-tabs\n' +
      'Available pages:\n' +
      pages.map(p => `  - [${p.type}] ${p.title} (${p.url.substring(0, 80)})`).join('\n')
    );
  }
  return devtoolsPage.webSocketDebuggerUrl;
}

async function toggleDeviceToolbar(webSocketUrl) {
  // 动态导入 ws，支持从多个位置查找
  let WebSocket;
  try {
    const ws = await import('ws');
    WebSocket = ws.default || ws.WebSocket;
  } catch {
    console.error('Error: "ws" module not found. Install it with: npm install ws');
    process.exit(1);
  }

  return new Promise((resolve, reject) => {
    const ws = new WebSocket(webSocketUrl);
    const timeout = setTimeout(() => {
      ws.close();
      reject(new Error('WebSocket connection timed out'));
    }, 10000);

    ws.on('open', () => {
      ws.send(JSON.stringify({
        id: 1,
        method: 'Runtime.evaluate',
        params: {
          expression: `(function() {
            var app = Emulation.AdvancedApp.instance();
            var deviceModeView = app.deviceModeView;
            var wasPreviouslyOn = deviceModeView.isDeviceModeOn();
            deviceModeView.toggleDeviceMode();
            var isCurrentlyOn = deviceModeView.isDeviceModeOn();
            return JSON.stringify({
              wasOn: wasPreviouslyOn,
              isNowOn: isCurrentlyOn,
              action: wasPreviouslyOn ? 'turned OFF' : 'turned ON'
            });
          })()`,
          returnByValue: true
        }
      }));
    });

    ws.on('message', (data) => {
      const message = JSON.parse(data.toString());
      if (message.id === 1) {
        clearTimeout(timeout);
        if (message.result?.result?.value) {
          resolve(JSON.parse(message.result.result.value));
        } else {
          reject(new Error('Unexpected response: ' + JSON.stringify(message)));
        }
        ws.close();
      }
    });

    ws.on('error', (error) => {
      clearTimeout(timeout);
      reject(error);
    });
  });
}

async function checkDeviceMode(webSocketUrl) {
  let WebSocket;
  try {
    const ws = await import('ws');
    WebSocket = ws.default || ws.WebSocket;
  } catch {
    console.error('Error: "ws" module not found.');
    process.exit(1);
  }

  return new Promise((resolve, reject) => {
    const ws = new WebSocket(webSocketUrl);
    const timeout = setTimeout(() => { ws.close(); reject(new Error('Timeout')); }, 10000);

    ws.on('open', () => {
      ws.send(JSON.stringify({
        id: 1,
        method: 'Runtime.evaluate',
        params: {
          expression: `JSON.stringify({
            isDeviceModeOn: Emulation.AdvancedApp.instance().deviceModeView.isDeviceModeOn()
          })`,
          returnByValue: true
        }
      }));
    });

    ws.on('message', (data) => {
      const message = JSON.parse(data.toString());
      if (message.id === 1) {
        clearTimeout(timeout);
        resolve(JSON.parse(message.result.result.value));
        ws.close();
      }
    });

    ws.on('error', (error) => { clearTimeout(timeout); reject(error); });
  });
}

async function main() {
  const subcommand = process.argv[3] || 'toggle';

  console.log(`Connecting to Chrome on port ${CDP_PORT}...`);

  const pages = await fetchJson(`http://localhost:${CDP_PORT}/json/list`);
  const devToolsWebSocketUrl = findDevToolsWebSocket(pages);
  console.log(`Found DevTools WebSocket: ${devToolsWebSocketUrl}`);

  if (subcommand === 'status') {
    const status = await checkDeviceMode(devToolsWebSocketUrl);
    console.log(`Device Mode is currently: ${status.isDeviceModeOn ? 'ON' : 'OFF'}`);
  } else {
    const result = await toggleDeviceToolbar(devToolsWebSocketUrl);
    console.log(`Device Toolbar ${result.action} (was: ${result.wasOn ? 'ON' : 'OFF'}, now: ${result.isNowOn ? 'ON' : 'OFF'})`);
  }
}

main().catch((error) => {
  console.error('Error:', error.message);
  process.exit(1);
});

```
