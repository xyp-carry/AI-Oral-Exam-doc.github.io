---
title: Agents
weight: 40
bookToc: true
---

# Agents

本页说明项目当前如何构建 Agent、注册工具、管理 prompt，以及控制长对话的输入长度。主要实现位于 `AIOralExamSystem/Agent/base_Agent.py` 和 `AIOralExamSystem/Agent/base_prompt.py`。

## Agent 的构建流程

`BaseAgent` 是公共基类。子类调用 `super().__init__(name, model_settings, ...)` 后，基类依次完成：

1. 调用子类的 `get_tools()`，取得工具列表。
2. 从 `model_settings` 读取必需的 `model_name`、`model_url`、`model_api_key`，创建 `ChatOpenAI`。
3. 组装长文本处理、模型调用次数限制等中间件。
4. 调用 `create_agent(model, tools=..., middleware=...)` 创建 LangChain Agent。
5. 按需绑定工具日志和工具事件回调。

`run()` 负责启动与停止心跳、调用子类的 `execute()`，并将运行异常整理为错误结果。具体任务输入、消息构造和结果处理由子类的 `execute()` 完成。

## 配置工具

子类覆盖 `get_tools()`，返回允许该 Agent 使用的工具。工具可以通过 LangChain 的 `@tool` 定义；`args_schema` 规定模型可传的参数，`description` 解释工具用途。基类只负责注册返回的工具，不会自动扫描工具目录。

`BaseAgent.__init__()` 会立即调用 `get_tools()`，因此工具需要的范围、路径或身份信息必须先于 `super().__init__()` 绑定：

```python
class MyAgent(BaseAgent):
    def __init__(self, model_settings: dict, document_scope: str):
        self.document_scope = document_scope
        super().__init__("MyAgent", model_settings)

    def get_tools(self):
        return [my_search_tool, my_read_tool]
```

项目中的 `QuestionSetterAgent.get_tools()` 返回 `docInfoSearch` 和 `docReadFile`。这两个工具在描述中说明文档范围，并在实现中把服务端绑定的 `document_scope` 传给搜索和读取工具。prompt 可以指导模型何时使用工具；实际可访问的范围仍由工具代码校验。没有工具需求的 Agent 可返回空列表，例如 `GeneralAgent`。

## 管理 prompt

`BasePrompt` 把内容分成系统提示词层和本次任务数据：

| 内容 | 设置方式 | 发送位置 |
| --- | --- | --- |
| 稳定规则 | `set_layer("stable.xxx", 内容)` | system 消息 |
| Agent 与项目上下文 | `set_layer("context.xxx", 内容)` | system 消息 |
| 当前运行信息 | `set_layer("volatile.xxx", 内容)` | system 消息 |
| 运行数据 | `update_context(section, values)` | user 消息中的“运行上下文” |
| 当前任务 | `build_messages(task)` 的 `task` | user 消息中的“当前任务” |

`build_system_prompt()` 按 `stable → context → volatile` 的顺序拼接各层。`set_layer()` 对同一路径再次赋值会替换原内容，传入空内容会移除该层。运行数据以字典形式保存在 Agent 实例内：`update_context()` 默认递归合并，`replace=True` 可覆盖整个分区，`clear_context()` 可清理不用的数据。`build_messages(task)` 最终返回 system 和 user 消息；如果没有系统层，就只返回 user 消息。

```python
prompt = BasePrompt()
prompt.set_layer("stable.evidence", "先取得证据再回答", title="证据原则")
prompt.set_layer("context.role", "你是项目助理", title="身份")
prompt.update_context("repository", {"active_branch": "main"})
messages = prompt.build_messages("解释这个模块")
```

`UniversalAgent` 使用 `UniversalPromptBuilder.create()` 配置固定规则，并在每次 `execute()` 前更新当前分支、构造本次任务，再调用 `build_prompt_messages()` 发送消息。其他一些子类仍在自己的 `execute()` 中手工构造 system/user 消息；`BasePrompt` 目前不是所有 Agent 的统一 prompt 入口。

工具运行中调用 `update_prompt_context()`，更新的是实例中保存的上下文。若本次 `ainvoke()` 的初始消息已经构建，这项更新会在下一次构建消息时进入 prompt；本次运行仍可直接读取工具返回值。重复使用同一个 Agent 实例时，应及时替换或清理不再适用的运行数据。

## 压缩长对话

长文本处理由 `BasePrompt.build_compression_middlewares()` 根据 `model_settings` 配置：

```python
model_settings = {
    "model_name": "...",
    "model_url": "...",
    "model_api_key": "...",
    "max_input_tokens": 12000,
    "summarization": {
        "enabled": True,
        "trigger": ("tokens", 8000),
        "keep": ("messages", 8),
    },
}
```

### 摘要压缩

仅当 `summarization.enabled=True` 时，才加入 LangChain 的 `SummarizationMiddleware`。默认触发条件为 `("tokens", 8000)`，默认保留最近 8 条消息。还可以配置摘要模型 `model`、`summary_prompt`、`trim_tokens_to_summarize` 和 `token_counter`。这项机制用于压缩历史对话；默认不启用。

### 输入预算兜底

`TokenBudgetMiddleware` 始终启用，默认 `max_input_tokens=12000`。每次模型调用前，它计算消息和工具定义的预计 token 数；有自定义 `token_counter` 时优先使用，否则尝试模型计数方法，最后使用字符估算。超限时会移除较早的工具结果，只保留最近一次工具结果及其匹配的工具调用。如果处理后仍超限，就抛出错误。它只裁剪工具历史，不会生成摘要，也不会自动缩短过长的当前任务或单条工具结果。

### 模型调用次数

`BaseAgent` 另外配置 `ModelCallLimitMiddleware` 和 `FinalizationBudgetMiddleware`：`max_agent_model_calls` 默认 40 次，`finish_warning_ratio` 默认 0.8。接近上限时，后者会临时追加收尾指令。调用次数限制与输入 token 预算是两件事；`UniversalAgent` 将自己的调用次数上限设为 100。
