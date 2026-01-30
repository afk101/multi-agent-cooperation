PolyMind 软件规格说明书 (Software Specification) v7.0
1. 项目概述 (Project Overview)
PolyMind 是一个基于 Node.js 的命令行界面 (CLI) 工具，旨在通过并行编排多个 AI Agent 来协同解决用户问题。系统支持挂载本地工作目录作为上下文，允许 Agent 使用工具（如文件读取、MCP 网络搜索）获取信息。所有 Agent 通过统一的 OpenAI 兼容 API 接口连接不同的底层模型（如 GPT-5.2, Claude 4.5 Opus 等），最终聚合多方视角输出结果。

2. 技术栈与依赖 (Tech Stack & Dependencies)
为了保证代码的健壮性、类型安全和最佳兼容性：

Runtime: Node.js (TypeScript)
Build: tsc (编译到 dist/，支持生产环境运行)
Test: vitest (单元测试)
CLI Framework: commander (命令解析)
LLM Client: openai (官方 Node.js SDK)
Validation: zod (运行时配置验证)
Config Parser: js-yaml & dotenv
UI/UX: ora (Spinner 动画), chalk (彩色输出)
MCP Integration:
@modelcontextprotocol/sdk: 官方 SDK，用于实现 MCP Client。
Node.js child_process: 用于启动 uvx 等 MCP 服务进程。
3. 系统架构 (System Architecture)
遵循 SOLID 原则，模块化设计。

3.1 目录结构

polymind/
├── config/
│   └── agents.yaml         # 默认配置模板
├── src/
│   ├── cli/                # CLI 入口
│   ├── config/             # 配置加载器
│   ├── core/               # 核心层
│   │   ├── Orchestrator.ts # 编排器
│   │   ├── Agent.ts        # Agent 类
│   │   └── LLMClient.ts    # OpenAI SDK 封装
│   ├── tools/              # 工具层
│   │   ├── base/           # 抽象基类
│   │   ├── mcp/            # MCP Client & Adapter
│   │   └── fs/             # 原生 FS 工具实现
│   ├── interfaces/         # TS 接口
│   └── utils/              # 通用工具
└── package.json
3.2 核心设计决策
A. 交互与输出 (Interaction & Output)
模式: 单次运行 CLI (polymind [args])。
输出策略: 非流式聚合，按配置顺序排序。
调试支持: 支持 --verbose 参数，输出详细日志。
B. 配置加载 (Configuration)
Override 策略: CWD > User Home > Default。高优先级文件完全替换低优先级文件。
环境变量: 标准化 Key 为 PolyMind_API_KEY 和 PolyMind_BASE_URL。
C. 工具实现策略 (Tool Strategy)
WebSearch: 使用 MCP。
启动 uvx mcp-server-fetch。
启动检查：若 uvx 不可用，禁用相关工具并警告。
FileSystem: 使用 原生 TypeScript 实现。
上下文注入: 如果传入 --dir，Orchestrator 会自动在所有 Agent 的 System Prompt 末尾追加：“当前已挂载工作目录：[path]，请使用文件工具查阅代码...”。
快速失败: 若 --dir 路径不存在，CLI 立即报错退出。
D. 容错与限制 (Robustness)
Max Turns: 限制 20 次循环。
工具容错: 工具报错返回 Error Message 给 LLM，允许自我修正。
Agent 独立性: 各个 Agent 之间内存不共享，完全独立运行。
4. 数据流 (Data Flow)
启动: polymind "如何重构?" --dir ./src --verbose
验证与环境:
加载 Config (agents.yaml) & Env (PolyMind_*)。
验证 --dir。
检查 uvx。
初始化:
Orchestrator 创建 Agent 实例。
注入 Context Prompt: “当前已挂载工作目录...”
注入工具: web_fetch (MCP), list_dir, read_file (FS)。
并行执行:
UI 显示: ⠋ 3 Agents are thinking...
并发运行 run()。
Agent Loop:
LLM <-> Tools (Loop until finish/limit)。
输出:
等待所有完成。
按配置顺序排序。
打印报告。
5. 关键接口 (Key Interfaces)

// 配置接口
interface IAgentProfile {
  name: string;
  model: string;            // e.g. "gpt-5.2"
  apiBaseUrl?: string;      // 可选，覆盖全局
  apiKeyEnvVar?: string;    // 可选，覆盖全局 PolyMind_API_KEY
  systemPrompt: string;
  enableMcp: boolean;
}

// 统一结果
interface IAgentResult {
  agentName: string;
  status: 'fulfilled' | 'rejected';
  content?: string;
  error?: string;
}
6. 配置文件示例 (agents.yaml)

agents:
  - name: "架构师"
    model: "claude-4.5-opus"
    # api_base 和 api_key_env 可省略，默认使用全局 PolyMind_* 环境变量
    enable_mcp: true
    system_prompt: "..."

  - name: "审查员"
    model: "gpt-5.2"
    enable_mcp: false
    system_prompt: "..."
7. 预期输出 (Console Output)

$ polymind "分析鉴权模块" --dir ./src

🚀 PolyMind v1.0.0
📂 Context: /Users/qihoo/project/src

⠋ Agents are thinking... (Architect, Reviewer)
  (Spinner...)

==================================================
🤖 Agent: 架构师 (Architect)
==================================================
鉴权模块的设计采用了策略模式...

==================================================
🔍 Agent: 审查员 (Reviewer)
==================================================
发现 2 个潜在的安全漏洞...