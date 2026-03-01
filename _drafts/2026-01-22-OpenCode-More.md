整体架构，从核心流程出发，解决什么问题 -- 最后才能总结出来。

首先肯定是 tool registry、tool 执行、结果回灌。


agent 配置
ReAct 循环
会话管理
工具系统
LLM provider
事件总线

Interface 层（入口与协议） ：CLI/TUI/HTTP Server/ACP，把“用户输入/事件流”转成“Session 驱动”
Runtime 层（Project/Instance/State） ：工作目录、worktree、VCS、配置加载、生命周期状态
Conversation 层（Session/Message/Parts） ：消息与 part 的数据模型、存储、回放/流式更新、patch 记录
Brain 层（Prompting/Compaction/Summary/Retry）
    system prompt 准备
    tool 注册


Actuation 层（Tools + Policy） ：工具注册/执行、权限与审批、doom-loop 保护、输出截断
Extensibility 层（Plugins/MCP/Skills/Subagents）

oh-my-opencode

