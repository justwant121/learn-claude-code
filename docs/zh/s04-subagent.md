# s04: Subagents (子智能体)

`s01 > s02 > s03 > [ s04 ] s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"大任务拆小, 每个小任务干净的上下文"* -- 子智能体用独立 messages[], 不污染主对话。

## 问题

智能体工作越久, messages 数组越胖。每次读文件、跑命令的输出都==永久留在上下文里==。"这个项目用什么测试框架?" 可能要读 5 个文件, 但父智能体只需要一个词: "pytest。"

## 解决方案

```
Parent agent                     Subagent
+------------------+             +------------------+
| messages=[...]   |             | messages=[]      | <-- fresh
|                  |  dispatch   |                  |
| tool: task       | ----------> | while tool_use:  |
|   prompt="..."   |             |   call tools     |
|                  |  summary    |   append results |
|   result = "..." | <---------- | return last text |
+------------------+             +------------------+

Parent context stays clean. Subagent context is discarded.
```

## 工作原理

1. 父智能体有一个 `task` 工具。子智能体拥有==除 `task` 外==的所有基础工具 (禁止递归生成)。

```python
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task",
     "description": "Spawn a subagent with fresh context.",
     "input_schema": {
         "type": "object",
         "properties": {"prompt": {"type": "string"}},
         "required": ["prompt"],
     }},
]
```

2. 子智能体以 `messages=[]` 启动, 运行自己的循环。只有最终文本返回给父智能体。

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]
    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM,
            messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant",
                             "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input)
                results.append({"type": "tool_result",
                    "tool_use_id": block.id,
                    "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(
        b.text for b in response.content if hasattr(b, "text")
    ) or "(no summary)"
```

子智能体可能跑了 30+ 次工具调用, 但整个消息历史直接丢弃。父智能体收到的只是一段摘要文本, 作为普通 `tool_result` 返回。

## 相对 s03 的变更

| 组件           | 之前 (s03)       | 之后 (s04)                    |
|----------------|------------------|-------------------------------|
| Tools          | 5                | 5 (基础) + task (仅父端)      |
| 上下文         | 单一共享         | 父 + 子隔离                   |
| Subagent       | 无               | `run_subagent()` 函数         |
| 返回值         | 不适用           | 仅摘要文本                    |

## 试一试

```sh
cd learn-claude-code
python agents/s04_subagent.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Use a subtask to find what testing framework this project uses`
2. `Delegate: read all .py files and summarize what each one does`
3. `Use a task to create a new module, then verify it from here`

---

爹能用的工具 = 儿子能用的工具 + 其他

其中就有task工具，需要两个属性，一个prompt提示词，一个是描述，应该是专门拿去给儿子的命令

我们是拿parent_tool去给LLM的，所以自然就会用task工具，然后检查LLM的返回，如果是task就会去使用sub-agent函数

子agnet只能用30轮，其中再去调用LLM，但是用调用的参数是sub-message，tools也没有task工具，就避免了无限外包

看下子agent最后是什么结果，只返回最后一轮的结果，中间的一切都被丢弃掉

子agent跟之前的工具是一个逻辑，至少代码上，==在同一个层级上==，不是调用子agnet就是调用工具

减少了爹的上下文长度，而且还能限制子agent的权限

> 是的，`response.content` 被分成不同的部分（如 `text`、`tool_use` 等），通常是 **大语言模型（LLM）** 自己根据上下文和任务的要求生成的。这个过程是由模型在与用户交互时，根据其预设的规则或设计进行的。具体来说，模型生成的响应可以被划分为不同的**类型**，这些类型可以帮助处理不同的任务，比如文本生成、工具调用等。
>
> ### 解释 `response.content` 中不同部分的原因：
>
> 1. **文本（text）部分：**
>    - 这是最常见的部分，包含了 LLM 根据上下文生成的纯文本内容。这些文本是直接回应用户输入或先前消息的生成结果，通常是自然语言。
>    - 例如，当用户提出一个问题时，模型可能会生成一个回答，这部分的内容就是文本类型。
> 2. **工具调用（tool_use）部分：**
>    - LLM 在某些场景下可能需要调用外部工具或执行一些操作（比如执行系统命令、调用外部 API 等）。这些操作的请求会以 `tool_use` 类型的消息出现在响应中。
>    - 例如，在多轮对话中，如果模型需要查找某个数据或执行特定任务，它可能会调用外部工具。响应中的 `tool_use` 部分会描述该工具的调用细节。
> 3. **多任务与分离的响应结构：**
>    - 在一些复杂的任务中，LLM 可能会根据任务的需求自动分割响应内容，以便更好地处理不同类型的输出。例如：
>      - 一部分是文本回答，直接响应用户查询；
>      - 另一部分是工具的调用，执行具体的操作；
>      - 还有可能有其他类型的结果，例如状态或元数据。
> 4. **生成与交互模式：**
>    - LLM 是根据模型的设计和任务的需要来决定响应结构的。如果模型的设计包括与外部工具或系统的集成，它可能会将响应分成多个不同的部分。
>    - **例如**，一个对话系统可能需要同时返回两个结果：
>      - 一个是文本内容（比如对问题的回答）；
>      - 另一个是工具调用（比如查询数据库、运行代码等）。
>
> ### LLM 自己如何决定响应分段：
>
> - **模型架构和任务需求**：模型根据其架构（例如 GPT-3、GPT-4）和任务的需求来决定如何组织响应。如果任务需要调用外部工具（例如代码执行、API 调用等），模型会在响应中生成 `tool_use` 部分。
> - **上下文**：模型会根据上下文信息生成适当的响应类型。如果它检测到需要执行某个操作来完成任务（比如执行系统命令），它就会返回 `tool_use` 类型的消息。
> - **定义的系统**：系统设计和接口决定了响应的格式。开发者会在系统中定义如何处理 `tool_use` 或其他类型的消息，以及如何在响应中包含不同类型的数据。
>
> ### 举个例子：
>
> 假设你正在与一个大语言模型进行对话，任务是查询某个数据并给出答案。响应可能被分成以下几个部分：
>
> 1. **`text` 部分**：模型直接回答问题。
>
>    ```json
>    {
>      "role": "assistant",
>      "content": "The answer is 42."
>    }
>    ```
>
> 2. **`tool_use` 部分**：模型调用外部工具进行计算或获取数据。
>
>    ```json
>    {
>      "role": "assistant",
>      "content": "I need to check the database.",
>      "type": "tool_use",
>      "tool_name": "database_query",
>      "input": {"query": "SELECT * FROM facts WHERE question='What is the answer to life?'"} 
>    }
>    ```
>
> 3. **`tool_result` 部分**：工具返回结果。
>
>    ```json
>    {
>      "role": "assistant",
>      "content": "The result from the database is 42.",
>      "type": "tool_result"
>    }
>    ```
>
> ### 总结：
>
> `response.content` 中的各个部分（如 `text`、`tool_use` 等）通常是由 **LLM** 根据任务需求和上下文动态生成的。这些分段帮助模型处理不同的输出类型，确保能够正确地应对复杂的任务（比如文本生成、工具调用等）。模型会根据输入的上下文、任务的复杂性以及是否需要外部工具来决定如何将响应划分为不同类型的消息。
