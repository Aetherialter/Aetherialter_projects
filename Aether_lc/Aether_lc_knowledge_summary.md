# Aether_lc 项目知识点汇总

整理日期：2026-06-15  
适用对象：刚开始系统学习工程项目、CLI 工具、HTTP 请求、Cookie、测试与发布流程的开发者。

这份笔记不是设计过程记录，而是把 Aether_lc 项目中出现过的技术问题浓缩成知识点。你可以把它当成“通过一个真实小项目学习工程能力”的索引。

## 1. Aether_lc 是什么

Aether_lc 是一个面向 LeetCode 中文站的本地命令行工具。

它的核心工作流是：

```powershell
uv run lc login
uv run lc show
uv run lc get 1
uv run lc solve 1
uv run lc test
uv run lc submit
```

它不是本地题库系统，也不是完整刷题平台。当前 GitHub 版定位是轻量 CLI：在线取题、生成一个根目录 `solution.py`、本地测试、远程提交。

这个项目虽然小，但包含很多真实工程知识：命令行入口、HTTP 请求、Cookie 登录态、文件生成与解析、错误分层、测试、文档和发布。

## 2. CLI 工具基础

CLI 是 Command Line Interface，也就是命令行工具。

用户输入：

```powershell
uv run lc show --limit 20 --skip 0
```

程序接收命令、解析参数、执行对应逻辑。

Aether_lc 使用 `Typer` 写 CLI。

简化例子：

```python
from typer import Typer

app = Typer()

@app.command()
def show(limit: int = 50, skip: int = 0) -> None:
    ...
```

`@app.command()` 会把 Python 函数注册成命令。函数名 `show` 就对应 `lc show`。

CLI 层最好保持薄一点，只做三件事：

- 接收用户参数。
- 调用 service 层。
- 把结果交给 UI 展示。

不要把 HTTP 请求、文件解析、业务流程全塞进 CLI 函数里。

## 3. 项目分层

Aether_lc 按职责拆了多个模块：

```text
cli.py        命令入口
service.py    业务流程编排
client.py     LeetCode HTTP 客户端
workspace.py  solution.py 生成、解析与运行
problem.py    题目数据结构、题号解析、题目标准化
ui.py         终端输出
auth.py       Cookie 读取与 session 保存
```

这样做的好处：

- `cli.py` 不会变成大杂烩。
- `client.py` 专心处理网络请求。
- `workspace.py` 专心处理本地文件。
- `service.py` 专心串流程。
- 测试更容易写。
- 出 bug 时更容易定位是哪一层的问题。

这是从脚本走向工程项目的重要一步。

## 4. Typer 的 Exit

命令行工具遇到用户输入错误时，不应该直接打印 Python traceback。

应该输出用户能看懂的错误，然后退出。

Typer 提供：

```python
from typer import Exit

raise Exit(1)
```

常见约定：

- `Exit(0)`：成功退出。
- `Exit(1)`：失败退出。

项目中的例子：

```python
if limit <= 0:
    error("limit 必须是正整数")
    raise Exit(1)
```

这比让用户看到 `AttributeError`、`AssertionError` 更友好。

## 5. HTTP 请求基础

Aether_lc 需要请求 LeetCode 中文站，所以用到了 HTTP client。

项目使用：

```python
import httpx
```

创建客户端：

```python
self.client = httpx.Client(
    base_url="https://leetcode.cn",
    follow_redirects=True,
    timeout=20.0,
)
```

常见请求方式：

- `GET`：获取数据。
- `POST`：提交数据或查询复杂数据。

项目里：

```python
self.client.post("/graphql/", json=payload)
self.client.get(f"/submissions/detail/{submission_id}/check/")
```

`timeout=10` 表示最多等 10 秒。网络请求不能无限卡住。

## 6. GraphQL 基础

LeetCode 的题目详情和题目列表很多通过 GraphQL 获取。

GraphQL 通常向同一个接口发请求：

```text
POST /graphql/
```

请求体里写明要什么数据。

项目里的 payload 类似：

```python
payload = {
    "operationName": "questionData",
    "query": QUESTION_DETAIL_QUERY,
    "variables": {
        "titleSlug": title_slug,
    },
}
```

GraphQL 和普通 REST 的区别可以简单理解为：

- REST：不同资源通常对应不同 URL。
- GraphQL：通常同一个 URL，请求体里说明要查什么字段。

## 7. Cookie 和登录态

Cookie 是浏览器保存的一些小数据，网站用它识别你是谁。

登录 LeetCode 后，浏览器里会有类似：

```text
LEETCODE_SESSION=xxx
csrftoken=yyy
```

Aether_lc 会读取浏览器 Cookie，并保存到：

```text
.aether_lc/session.json
```

注意：本地保存的 Cookie 只是浏览器 Cookie 的副本，不会让登录态永久有效。

真正判断 Cookie 是否有效的是 LeetCode 服务端。

## 8. Cookie 是怎么发送的

项目里有：

```python
self.client.cookies.update(cookies)
```

这会把本地 session 里的 cookies 放进 `httpx.Client`。

之后每次请求，`httpx` 会自动带上类似请求头：

```http
Cookie: LEETCODE_SESSION=xxx; csrftoken=yyy
```

LeetCode 服务端收到请求后，会检查：

- Cookie 是否存在。
- session 是否还有效。
- session 是否过期。
- session 是否被登出或吊销。
- 对提交请求，还会检查 CSRF token。

所以不是 CLI 自己判断“我过期了吗”，而是每次请求时让服务端判断。

## 9. Cookie 失效后的表现

本地 `.aether_lc/session.json` 还在，不代表 Cookie 仍然有效。

可能出现：

```text
lc show 正常
lc get 正常
lc status 失败
lc submit 失败
```

原因是 `show/get` 查询的是公开题目数据，可能不要求登录；而 `status/profile/submit` 需要有效登录态。

当前项目在登录态无效时会提示类似：

```text
登录态无效或已过期, 请重新执行 lc login
```

解决方式：

```powershell
uv run lc login
```

重新从浏览器读取或手动录入 Cookie。

## 10. CSRF Token

CSRF 是一种 Web 安全机制。

提交代码属于敏感 POST 请求，LeetCode 不只看 Cookie，还要求 CSRF token。

项目提交时会带：

```python
headers={
    "X-CSRFToken": csrftoken,
    "Referer": f"{BASE_URL}/problems/{title_slug}/",
}
```

可以简单理解为：

- Cookie 证明“你是谁”。
- CSRF token 帮助证明“这个请求来自合法上下文”。

如果缺少 `csrftoken`，提交应失败并提示重新登录。

## 11. JSON 和接口结构校验

HTTP 返回值通常会解析成 JSON：

```python
result = response.json()
```

但不能盲信返回结构。

风险包括：

- 返回的不是 JSON。
- JSON 里没有 `data`。
- `data` 是 `null`。
- 字段名变了。
- 返回登录页 HTML。

危险写法：

```python
question = result["data"]["question"]
```

更稳妥：

```python
data = result.get("data", {})
if not isinstance(data, dict):
    return ClientResult(error=ClientErrorKind.INVALID_RESPONSE)
```

这能避免 `None.get(...)` 之类的 traceback。

## 12. 异常处理

项目中常见异常：

```python
except httpx.RequestError:
    ...
except httpx.HTTPStatusError:
    ...
except ValueError:
    ...
```

含义：

- `RequestError`：网络层失败，比如断网、连接失败。
- `HTTPStatusError`：服务器返回错误状态码，比如 403、500。
- `ValueError`：JSON 解析失败。

不要随便写：

```python
except Exception:
    ...
```

那会把真正的程序 bug 也吞掉，后续更难排查。

## 13. Result 对象

早期函数可能返回：

```python
dict | None
```

但 `None` 只能表示失败，不能说明为什么失败。

失败可能是：

- 网络失败。
- HTTP 错误。
- JSON 解析失败。
- 接口结构变了。
- 登录态无效。
- 缺少 CSRF token。

所以项目引入：

```python
ClientResult
ClientErrorKind
```

例子：

```python
ClientResult(data=stats)
ClientResult(error=ClientErrorKind.NETWORK)
```

调用方：

```python
if not result.ok:
    error(...)
    raise Exit(1)
```

这样错误类型更清楚，也不容易出现 `None.get(...)`。

## 14. Enum 枚举

错误类型使用 Enum：

```python
class ClientErrorKind(str, Enum):
    NETWORK = "network"
    HTTP = "http"
    INVALID_JSON = "invalid_json"
    INVALID_RESPONSE = "invalid_response"
    UNAUTHORIZED = "unauthorized"
    MISSING_CSRF = "missing_csrf"
```

好处：

- 错误值固定。
- IDE 能提示。
- 不容易拼错字符串。

比到处写字符串更安全。

## 15. dataclass

`ClientResult` 适合用 dataclass：

```python
@dataclass(frozen=True)
class ClientResult:
    data: Any = None
    error: ClientErrorKind | None = None
    message: str = ""
```

`dataclass` 会自动生成初始化方法。

`frozen=True` 表示对象创建后不建议再修改，让它更像一个稳定的数据结果。

## 16. @property

项目里有：

```python
@property
def ok(self) -> bool:
    return self.error is None
```

调用时：

```python
if result.ok:
    ...
```

不是：

```python
if result.ok():
    ...
```

`@property` 的作用是让一个方法像属性一样被访问。

## 17. service 层

service 层是业务流程编排层。

例如 `lc submit` 背后的流程：

```text
读取 solution.py
解析元信息和提交代码
读取本地 cookies
提交代码到 LeetCode
得到 submission_id
轮询判题结果
返回结果给 UI
```

这些流程不应该全写在 `cli.py`，否则 CLI 会越来越乱。

service 层让业务流程更清晰，也更容易测试。

## 18. workspace 层和 solution.py

Aether_lc 的核心工作文件是根目录：

```text
solution.py
```

`workspace.py` 负责：

- 生成 `solution.py`。
- 写入题目元信息。
- 写入提交区域 marker。
- 解析提交代码。
- 运行本地测试。

读写文件时使用：

```python
encoding="utf-8"
```

这样中文标题不容易乱码。

## 19. marker 标记

项目用 marker 标记提交区域：

```python
# @lc submit_begin
class Solution:
    ...
# @lc submit_end
```

`lc submit` 只提交这两个 marker 中间的代码。

为什么需要 marker？

因为 `solution.py` 里还有：

- 元信息。
- 自动导入。
- 本地测试函数 `run_cases()`。
- `if __name__ == "__main__"`。

这些不一定要提交给 LeetCode。

marker 让提交范围明确。

## 20. 题目元信息

生成的 `solution.py` 里会有：

```python
# @lc problem_id: 1
# @lc submit_question_id: 1
# @lc title: Two Sum
# @lc title_slug: two-sum
```

含义：

- `problem_id`：用户看到的题号。
- `submit_question_id`：LeetCode 提交接口需要的内部 ID。
- `title`：题目标题。
- `title_slug`：题目 URL slug。

为什么写进文件？

因为 CLI 每次执行都是短生命周期进程。

`lc solve 1` 执行完，程序就结束了。

下一次 `lc submit` 是新的进程，不能依赖内存记住上次题目。

所以要把提交目标写进 `solution.py`。

## 21. 展示题号和内部 ID

LeetCode 有两个容易混淆的 ID：

- 展示题号：用户在页面上看到的题号。
- 内部 questionId：提交接口需要的 ID。

有些题这两个不一样。

如果提交时用错 ID，就可能提交到错误题目。

所以项目分开保存：

```python
problem_id
submit_question_id
```

这是接口集成里非常重要的细节。

## 22. submission_id

提交代码后，LeetCode 不会立刻返回最终判题结果。

它会先返回：

```python
submission_id
```

然后项目用这个 ID 去查询判题状态。

流程：

```text
提交代码
  -> 得到 submission_id
  -> 查询判题状态
  -> PENDING / STARTED 继续等
  -> Accepted / Wrong Answer / Runtime Error 返回结果
```

所以提交函数不能只返回 bool。

## 23. 轮询

轮询就是隔一段时间重复查询，直到结果出现。

简化逻辑：

```python
for _ in range(MAX_ATTEMPTS):
    result = client.get_submission_result(submission_id)
    if state not in {"PENDING", "STARTED"}:
        return result
    sleep(0.5)
```

注意点：

- 不能无限轮询。
- 要设置最大次数。
- 每次之间要等待。

## 24. 参数校验

`lc show` 支持：

```powershell
uv run lc show --limit 20 --skip 0
```

参数含义：

- `limit`：一次显示多少题。
- `skip`：跳过多少题。

当前规则：

- `limit` 必须是正整数。
- `limit` 最大为 100。
- `skip` 必须是非负整数。

错误输入应该在本地拦截：

```powershell
uv run lc show --limit -1 --skip 0
```

输出：

```text
limit 必须是正整数
```

而不是请求 LeetCode 后显示接口异常。

## 25. 分页

题目列表不能假设一次取完。

分页常见参数：

```text
limit=100, skip=0
limit=100, skip=100
limit=100, skip=200
```

`show` 是用户分页查看。

`get/solve` 按题号查找高题号时，也需要分页扫描题目索引。

这解决了高题号如 `2196` 找不到的问题。

## 26. 本地测试

`lc test` 本质上是运行：

```text
solution.py
```

生成的文件中有：

```python
def run_cases() -> None:
    pass

if __name__ == "__main__":
    run_cases()
```

用户可以自己写断言：

```python
def run_cases() -> None:
    assert Solution().twoSum([2, 7, 11, 15], 9) == [0, 1]
```

当前轻量设计里，如果 `run_cases()` 没有断言，程序正常运行就会显示“本地测试通过”。

这不是严格单元测试，只是本地运行入口。

## 27. 模板和编辑器提示

生成 `solution.py` 时会导入常用类型：

```python
from typing import Any, Dict, List, Optional, Set, Tuple
```

这些导入刚开始可能没被使用，编辑器可能提示未使用。

项目用文件级配置降低干扰：

```python
# pyright: reportUnusedImport=false, reportUnusedVariable=false
# ruff: noqa: F401, F841
```

这样刷题工作区更安静。

## 28. TreeNode 和 ListNode

LeetCode 的树、链表题模板里经常有注释掉的定义：

```python
# class TreeNode:
#     ...
```

当前项目不自动取消注释。

如果用户要本地构造树或链表测试，需要自己取消注释。

原因是当前 GitHub 版保持轻量，不做复杂模板改写器。

## 29. Ruff

Ruff 用于格式化和静态检查。

常用命令：

```powershell
uv run ruff format src tests
uv run ruff check src pyproject.toml tests
```

为什么不总是：

```powershell
uv run ruff check .
```

因为根目录 `solution.py` 是用户刷题工作区，可能有未完成代码。

所以项目源码检查一般限定在：

```text
src
pyproject.toml
tests
```

## 30. pytest

pytest 用来写自动化测试。

运行：

```powershell
uv run pytest
```

项目测试覆盖：

- 题目数据标准化。
- `solution.py` 生成和解析。
- service 层错误边界。
- `lc show` 参数校验。

### monkeypatch

测试里常用 `monkeypatch` 替换真实依赖。

例如：

```python
monkeypatch.setattr(service, "load_session", lambda: {"cookies": {"k": "v"}})
```

这样测试不会真的读取本地 session。

也可以用 fake client 避免真实网络请求。

## 31. fake client

测试 service 层时，不应该依赖真实 LeetCode 网络。

可以写假的 client：

```python
class FakeClient:
    def problem_list(self, limit: int = 50, skip: int = 0) -> ClientResult:
        return ClientResult(data=None)
```

这样能稳定测试错误路径。

好处：

- 测试快。
- 测试稳定。
- 不依赖真实账号。
- 不怕 LeetCode 临时变慢。

## 32. pyproject.toml

`pyproject.toml` 是 Python 项目的核心配置文件。

里面有项目名和版本：

```toml
[project]
name = "aether-lc"
version = "0.5.6"
```

还有命令入口：

```toml
[project.scripts]
lc = "aether_lc.cli:app"
```

这表示安装项目后，`lc` 命令会指向 `aether_lc.cli:app`。

## 33. uv

项目用 uv 管理依赖和运行命令。

常用命令：

```powershell
uv sync
uv run lc --help
uv run pytest
uv lock
```

`uv.lock` 用来锁定依赖版本。

发布新版本时，需要同步：

```powershell
uv lock
```

## 34. README 和 ROADMAP

README 面向用户。

应该回答：

- 项目是什么。
- 怎么安装。
- 怎么使用。
- 有哪些命令。
- 有哪些限制。
- 怎么验证。

ROADMAP 面向维护者和未来的自己。

应该记录：

- 已实现版本。
- 后续计划。
- 当前不做什么。
- 重要取舍。

文档不是装饰，是工程质量的一部分。

## 35. Git 发布流程

常见发布流程：

```powershell
git status
git add README.md ROADMAP.md pyproject.toml uv.lock src tests
git commit -m "fix: validate show pagination options"
git tag v0.5.6
git push origin main
git push origin v0.5.6
```

发布前必须确认：

- `solution.py` 为空。
- README 版本号正确。
- `pyproject.toml` 版本号正确。
- `uv.lock` 版本号正确。
- pytest 通过。
- ruff 通过。

## 36. 版本号

例如：

```text
v0.5.6
```

可以简单理解为：

- `0`：还没到正式稳定 1.0。
- `5`：当前功能阶段。
- `6`：patch 修复版本。

`v0.5.x` 主要是远程提交上线后的 bugfix 和体验收束。

## 37. 当前技术债

当前已知后续可以做：

- 新增 `lc doctor`。
- 更详细的错误诊断。
- service 层解包函数 `_unwrap_client_result()`。
- client 层请求封装 `_request_json()`。
- 提交前展示当前提交目标。
- 提交区域基础校验。
- 轻量缓存。
- GitHub Actions。

技术债不是坏事，关键是知道它在哪里、什么时候处理。

## 38. v0.6 的重点：doctor

`lc doctor` 应该帮助用户定位环境问题。

可以检查：

- session 文件是否存在。
- session JSON 是否可解析。
- cookies 字段是否存在。
- cookies 是否仍能通过 `userStatus` 验证。
- LeetCode 是否能连接。
- `solution.py` 是否为空。
- `solution.py` 是否包含元信息。
- `solution.py` 是否包含提交区域。
- 提交区域是否包含 `class Solution`。

它的目标不是扩展功能，而是让问题更容易诊断。

## 39. 学这个项目应该掌握什么

通过 Aether_lc，你应该掌握：

- CLI 工具如何设计。
- Python 项目如何分层。
- HTTP 请求如何发送。
- Cookie 登录态如何复用。
- CSRF token 为什么重要。
- GraphQL 返回值为什么要校验。
- 为什么不要只返回 `None`。
- Result 对象怎么表达错误。
- 文件生成和解析怎么做。
- marker 如何界定代码区域。
- 提交任务为什么需要轮询。
- pytest 如何 mock 外部依赖。
- README 和 ROADMAP 为什么重要。
- 发布前为什么必须跑验证命令。

## 40. 推荐阅读顺序

如果你是小白，建议按这个顺序看代码：

1. `cli.py`：理解命令怎么进来。
2. `service.py`：理解一个命令背后的业务流程。
3. `client.py`：理解 HTTP、Cookie、错误处理。
4. `workspace.py`：理解文件生成和解析。
5. `problem.py`：理解题号解析和题目数据结构。
6. `tests/`：理解如何测试不依赖真实网络。
7. `README.md` 和 `ROADMAP.md`：理解项目如何面向用户和未来维护。

## 41. 一句话总结

Aether_lc 表面上是一个 LeetCode CLI，实际上浓缩了很多工程基础：

```text
命令行入口
业务分层
HTTP 客户端
Cookie 登录态
错误建模
本地文件工作区
远程提交轮询
自动化测试
文档与发布
```

这些能力比单纯刷一道算法题更接近真实软件工程。
