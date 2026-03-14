# s02: Tool Use (工具使用)

`s01 > [ s02 ] s03 > s04 > s05 > s06 | s07 > s08 > s09 > s10 > s11 > s12`

> *"加一个工具, 只加一个 handler"* -- 循环不用动, 新工具注册进 dispatch map 就行。

## 问题

只有 `bash` 时, 所有操作都走 shell。`cat` 截断不可预测, `sed` 遇到特殊字符就崩, 每次 bash 调用都是==不受约束==的安全面。专用工具 (`read_file`, `write_file`) 可以在工具层面做路径沙箱。

之前是我们相信LLM模型可以返回完整的可靠的bash命令，但是我们在agnet-loop里面全让run-bash函数干活了，不安全，并且没办法增加别的工具了，现在我们不希望他去读取到我们别的文件夹下的文件

关键洞察: 加工具不需要改循环。

## 解决方案

```
+--------+      +-------+      +------------------+
|  User  | ---> |  LLM  | ---> | Tool Dispatch    |
| prompt |      |       |      | {                |
+--------+      +---+---+      |   bash: run_bash |
                    ^           |   read: run_read |
                    |           |   write: run_wr  |
                    +-----------+   edit: run_edit |
                    tool_result | }                |
                                +------------------+

The dispatch map is a dict: {tool_name: handler_function}.
One lookup replaces any if/elif chain.
```

## 工作原理

1. 每个工具有一个处理函数。路径沙箱==防止逃逸工作区==。防止攻击注入 

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path

def run_read(path: str, limit: int = None) -> str:
    text = safe_path(path).read_text()
    lines = text.splitlines()
    if limit and limit < len(lines):
        lines = lines[:limit]
    return "\n".join(lines)[:50000]
```

2. dispatch map 将==工具名映射到处理函数==。

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                        kw["new_text"]),
}
```

3. 循环中按名称查找处理函数。循环体本身与 s01 完全一致。

```python
for block in response.content:
    if block.type == "tool_use":
        handler = TOOL_HANDLERS.get(block.name)
        output = handler(**block.input) if handler \
            else f"Unknown tool: {block.name}"
        results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": output,
        })
```

加工具 = 加 handler + 加 schema。循环永远不变。

## 相对 s01 的变更

| 组件           | 之前 (s01)         | 之后 (s02)                     |
|----------------|--------------------|--------------------------------|
| Tools          | 1 (仅 bash)        | 4 (bash, read, write, edit)    |
| Dispatch       | 硬编码 bash 调用   | `TOOL_HANDLERS` 字典           |
| 路径安全       | 无                 | `safe_path()` 沙箱             |
| Agent loop     | 不变               | 不变                           |

## 试一试

```sh
cd learn-claude-code
python agents/s02_tool_use.py
```

试试这些 prompt (英文 prompt 对 LLM 效果更好, 也可以用中文):

1. `Read the file requirements.txt`
2. `Create a file called greet.py with a greet(name) function`
3. `Edit greet.py to add a docstring to the function`
4. `Read greet.py to verify the edit worked`

---

原来==只有run_bash这么一个工具==，现在使用的是TOOL_HANDLERS.get这么一个方法

TOOL_HANDLERS本身是一个 字典，它的作用是根据传入的命令类型调用不同的处理函数。

每个键代表一个操作命令，值是一个 lambda 函数，用于执行相应的操作。

> `lambda` 函数是 Python 中的一种简洁的匿名函数（即没有名称的函数）。它通常用于需要快速定义一个简单函数的场景，而不需要正式定义一个完整的函数。
>
> ### 语法：
>
> ```python
> lambda 参数1, 参数2, ..., 参数n : 表达式
> ```
>
> - `lambda` 关键字用于定义一个匿名函数。
> - 参数部分列出函数的输入。
> - `:` 后面跟着一个表达式，该表达式的结果会作为函数的返回值。
>
> ### 特点：
>
> 1. **简短**：`lambda` 函数通常用于非常简单的操作，像是单行的运算。
> 2. **匿名**：`lambda` 函数没有名字，它不需要像普通函数那样使用 `def` 来命名。
> 3. **返回值**：`lambda` 函数会自动返回表达式的结果，无需使用 `return` 语句。
>
> ### 示例：
>
> #### 1. 计算两个数的和：
>
> ```python
> sum = lambda x, y: x + y
> print(sum(3, 5))  # 输出 8
> ```
>
> 这里 `lambda x, y: x + y` 定义了一个匿名函数，接受两个参数 `x` 和 `y`，并返回它们的和。
>
> #### 2. 判断一个数是否为偶数：
>
> ```python
> is_even = lambda x: x % 2 == 0
> print(is_even(4))  # 输出 True
> print(is_even(7))  # 输出 False
> ```
>
> 这个 `lambda` 函数接收一个参数 `x`，并返回 `x` 是否是偶数的判断结果。
>
> #### 3. 在 `sort()` 中使用 `lambda`：
>
> ```python
> list_of_tuples = [(1, 'one'), (3, 'three'), (2, 'two')]
> sorted_list = sorted(list_of_tuples, key=lambda x: x[0])
> print(sorted_list)  # 输出 [(1, 'one'), (2, 'two'), (3, 'three')]
> ```
>
> 这里的 `lambda x: x[0]` 表示排序时根据元组的第一个元素来排序。
>
> ### 总结：
>
> `lambda` 函数让我们能够在一行代码中快速定义简单的函数，常用于一些短小的、临时的函数需求场景，比如在 `map()`、`filter()`、`sorted()` 等函数中传递作为参数。

> `lambda **kw: run_bash(kw["command"])` 是一个使用 `lambda` 表达式定义的匿名函数，它接受一个**可变数量的关键字参数（`\**kw`）**，并将 `kw` 字典中的 `command` 对应的值传递给 `run_bash` 函数。
>
> ### 详细解析：
>
> - `lambda **kw`: 这是一个 **lambda** 表达式，定义了一个匿名函数。`**kw` 表示接收一个 **关键字参数字典**（也就是你可以传递多个键值对作为参数），这些参数会作为字典 `kw` 传递给该函数。
> - `kw["command"]`: 在 `lambda` 函数体内，==`kw` 是一个字典==。`kw["command"]` 是从 `kw` 字典中获取键为 `"command"` 的值。
> - `run_bash(kw["command"])`: 将 `kw["command"]` 的值传递给 `run_bash` 函数。也就是说，`run_bash` 函数会接收 `kw["command"]` 的值并执行。
>
> ### 示例：
>
> 假设我们有一个 `run_bash` 函数，它接受一个命令字符串并执行它：
>
> ```python
> def run_bash(command):
>     print(f"执行命令: {command}")
> ```
>
> 然后你可以通过 `lambda` 表达式来调用 `run_bash`：
>
> ```python
> lambda **kw: run_bash(kw["command"])
> ```
>
> 比如，如果你调用这个 `lambda` 函数：
>
> ```python
> lambda_func = lambda **kw: run_bash(kw["command"])
> lambda_func(command="ls -l")
> ```
>
> 这会调用 `run_bash` 函数，并传递 `"ls -l"` 作为 `command` 参数，最终输出：
>
> ```
> 执行命令: ls -l
> ```
>
> ### 总结：
>
> `lambda **kw: run_bash(kw["command"])` 定义了一个匿名函数，它接收一个字典（包含多个键值对作为参数），然后从字典中获取 `"command"` 键对应的值，传递给 `run_bash` 函数。
