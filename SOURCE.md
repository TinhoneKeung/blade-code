# Outline

## Agent

### SubAgent

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

## Configuration Management

## MCP

## Tools

## Plugin

## Skills

## Hooks

## Spec Mode

## Plan Mode

## SessionRuntime

## Slash Command

* `/init`

## Zustand/Vanilla Store

### Slice

#### State

#### Action

### Selector

###

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

&#x20; &#x20;

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
