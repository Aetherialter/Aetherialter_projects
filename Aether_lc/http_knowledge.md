# Aether_lc HTTP 知识点

整理日期：2026-06-19

本文档面向刚开始系统学习 HTTP、Cookie、Python HTTP 客户端和 CLI 工程分层的开发者。内容以 Aether_lc 当前代码为例，解释项目里用到的 HTTP 知识。

## 1. HTTP 在 Aether_lc 中负责什么

Aether_lc 是一个本地 CLI，但它的核心数据来自 LeetCode 中文站。

整体关系是：

```text
Aether_lc CLI -> leetcode.cn -> 返回题目、登录态、提交结果
```

在项目中，HTTP 主要用于：

- 获取当前登录状态。
- 获取题目列表。
- 获取题目详情。
- 提交代码。
- 查询判题结果。

核心代码集中在：

```text
src/aether_lc/client.py
```

当前项目使用的是 `httpx`：

```python
import httpx
```

## 2. 一次 HTTP 请求包含什么

一次 HTTP 请求通常包含：

- method：请求方法，例如 `GET`、`POST`。
- url：请求地址。
- headers：请求头。
- cookies：登录态等 Cookie 信息。
- body/json：请求体。

一次 HTTP 响应通常包含：

- status code：状态码。
- headers：响应头。
- body：响应体。

在 Aether_lc 中，获取题目详情大概是：

```python
response = self.client.post("/graphql/", json=payload, timeout=10)
```

实际请求地址是：

```text
https://leetcode.cn/graphql/
```

因为 client 初始化时设置了：

```python
base_url="https://leetcode.cn"
```

## 3. GET 和 POST

常见 HTTP 方法：

- `GET`：获取数据。
- `POST`：提交数据，或发送复杂查询。

项目里的 `GET` 示例：

```python
response = self.client.get(
    f"/submissions/detail/{submission_id}/check/",
    timeout=10,
)
```

它用于查询某次提交的判题状态。

项目里的 `POST` 示例：

```python
response = self.client.post("/graphql/", json=payload, timeout=10)
```

它用于请求 GraphQL 数据。

提交代码也是 `POST`：

```python
response = self.client.post(
    f"/problems/{title_slug}/submit/",
    json=payload,
    headers={...},
    timeout=10,
)
```

## 4. base_url

项目中：

```python
BASE_URL = "https://leetcode.cn"

self.client = httpx.Client(
    base_url=BASE_URL,
    ...
)
```

之后可以写相对路径：

```python
self.client.post("/graphql/")
```

实际等价于：

```text
https://leetcode.cn/graphql/
```

`base_url` 的价值是减少重复，并保证所有请求都指向同一个站点。

## 5. Headers 请求头

Headers 是请求附带的说明信息。

项目中的默认 headers：

```python
headers={
    "User-Agent": USER_AGENT,
    "Accept": "application/json, text/plain, */*",
    "Origin": BASE_URL,
    "Referer": f"{BASE_URL}/",
}
```

含义：

- `User-Agent`：说明客户端类型，项目中模拟浏览器。
- `Accept`：告诉服务端希望接收什么格式。
- `Origin`：请求来源站点。
- `Referer`：请求来自哪个页面上下文。

提交代码时还有：

```python
headers={
    "X-CSRFToken": csrftoken,
    "Referer": f"{BASE_URL}/problems/{title_slug}/",
}
```

这是 LeetCode 安全校验的一部分。

## 6. Cookie 和登录态

Cookie 是浏览器保存的小型身份数据。

登录 LeetCode 后，浏览器里通常会有：

```text
LEETCODE_SESSION=xxx
csrftoken=yyy
```

Aether_lc 会读取浏览器 Cookie，并保存到：

```text
.aether_lc/session.json
```

项目把本地 Cookie 放进 HTTP client：

```python
if cookies:
    self.client.cookies.update(cookies)
```

后续请求会自动带上：

```http
Cookie: LEETCODE_SESSION=xxx; csrftoken=yyy
```

LeetCode 服务端收到 Cookie 后判断：

- session 是否存在。
- session 是否过期。
- session 是否被吊销。
- 当前用户是否有权限。

本地保存 Cookie 不会延长 LeetCode 登录态有效期。

## 7. Cookie 失效后的表现

本地 `.aether_lc/session.json` 还在，不代表 Cookie 仍然有效。

可能出现：

```text
lc show 正常
lc get 正常
lc status 失败
lc submit 失败
```

原因是：

- `lc show` 和 `lc get` 访问公开题目数据，可能不要求登录。
- `lc status`、`lc profile`、`lc submit` 需要有效登录态。

Cookie 失效后，项目会提示类似：

```text
登录态无效或已过期, 请重新执行 lc login
```

重新同步方式：

```powershell
uv run lc login
```

## 8. CSRF Token

CSRF 是一种 Web 安全机制。

提交代码属于敏感 POST 请求，所以 LeetCode 不只看 Cookie，还会检查 CSRF token。

项目中：

```python
csrftoken = self.client.cookies.get("csrftoken")
```

如果没有：

```python
return ClientResult(error=ClientErrorKind.MISSING_CSRF)
```

提交时发送：

```python
headers={
    "X-CSRFToken": csrftoken,
    "Referer": f"{BASE_URL}/problems/{title_slug}/",
}
```

可以简单理解为：

```text
Cookie      证明你是谁
CSRF token 证明这个 POST 请求更像合法页面上下文发出的
```

## 9. JSON 请求体

项目大量使用：

```python
json=payload
```

例如：

```python
payload = {
    "operationName": "questionData",
    "query": QUESTION_DETAIL_QUERY,
    "variables": {
        "titleSlug": title_slug,
    },
}

response = self.client.post("/graphql/", json=payload, timeout=10)
```

`json=payload` 会把 Python dict 转成 JSON 请求体。

## 10. JSON 响应体

收到响应后，项目会调用：

```python
result = response.json()
```

这会把响应体解析成 Python 对象。

但不能直接相信远端结构。

危险写法：

```python
question = result["data"]["question"]
```

如果 `data` 是 `None`，就会报错。

更稳妥：

```python
data = result.get("data", {})
if not isinstance(data, dict):
    return ClientResult(error=ClientErrorKind.INVALID_RESPONSE)
```

这能防止 `None.get(...)` 这类 traceback。

## 11. HTTP 状态码和 raise_for_status

常见状态码：

- `200`：成功。
- `301/302`：重定向。
- `400`：请求错误。
- `401`：未认证。
- `403`：禁止访问。
- `404`：不存在。
- `500`：服务端错误。

项目中：

```python
response.raise_for_status()
```

含义：

```text
如果响应状态码表示错误，就抛出 HTTPStatusError。
```

所以项目捕获：

```python
except httpx.HTTPStatusError:
    return ClientResult(error=ClientErrorKind.HTTP)
```

## 12. timeout

项目里每个请求基本都有：

```python
timeout=10
```

或者：

```python
timeout=20
```

timeout 的意义是：网络不能无限等。

如果 LeetCode 响应卡住，程序应该在指定时间后失败并返回错误，而不是一直挂起。

## 13. follow_redirects

项目初始化：

```python
httpx.Client(
    follow_redirects=True,
    ...
)
```

重定向是：

```text
请求 A
服务端说去 B
客户端再请求 B
```

例如未登录时，服务端可能把请求重定向到登录页。

但要注意：自动重定向后返回的内容不一定还是 JSON，所以仍然需要 JSON 解析和结构校验。

## 14. 为什么使用 httpx.Client

项目使用：

```python
self.client = httpx.Client(...)
```

而不是每次直接：

```python
httpx.post(...)
```

`Client` 可以复用：

- base_url。
- headers。
- cookies。
- timeout。
- follow_redirects。
- 连接池。

你的项目会连续请求同一个站点，所以用 `httpx.Client` 合理。

## 15. 异常分层

项目中典型异常处理：

```python
except httpx.RequestError:
    return ClientResult(error=ClientErrorKind.NETWORK)
except httpx.HTTPStatusError:
    return ClientResult(error=ClientErrorKind.HTTP)
except ValueError:
    return ClientResult(error=ClientErrorKind.INVALID_JSON)
```

含义：

- `RequestError`：请求发不出去，通常是网络问题。
- `HTTPStatusError`：服务端返回错误状态码。
- `ValueError`：响应不是合法 JSON。

这比统一返回 `None` 更清楚。

## 16. ClientResult

早期可能写：

```python
def user_status() -> dict | None:
    ...
```

但 `None` 不能表达失败原因。

现在使用：

```python
ClientResult(data=...)
ClientResult(error=ClientErrorKind.NETWORK)
```

调用方判断：

```python
if not result.ok:
    error(...)
    raise Exit(1)
```

这样可以把错误分层：

```text
client.py 识别错误类型
service.py 转成用户提示
cli.py 负责命令入口
```

## 17. Aether_lc 中的主要 HTTP 接口

### user_status

作用：验证当前 Cookie 是否登录。

```python
response = self.client.post("/graphql/", json=USER_STATUS_QUERY, timeout=10)
```

返回字段包括：

- `isSignedIn`
- `username`
- `realName`
- `avatar`
- `isPremium`

### problem_list

作用：分页获取题目索引。

变量：

```python
"limit": limit
"skip": skip
```

### problem_detail

作用：根据 `titleSlug` 获取题目详情、题面和代码模板。

### submit_solution

作用：提交代码。

```python
response = self.client.post(
    f"/problems/{title_slug}/submit/",
    json=payload,
    headers={...},
    timeout=10,
)
```

payload：

```python
{
    "lang": "python3",
    "question_id": question_id,
    "typed_code": code,
}
```

### get_submission_result

作用：查询判题结果。

```python
response = self.client.get(
    f"/submissions/detail/{submission_id}/check/",
    timeout=10,
)
```

## 18. submission_id 和轮询

提交代码后，LeetCode 不会马上返回最终结果。

它会先返回：

```python
submission_id
```

然后项目用这个 ID 查询判题状态。

流程：

```text
提交代码
  -> 得到 submission_id
  -> 查询判题状态
  -> PENDING / STARTED 继续等待
  -> Accepted / Wrong Answer / Runtime Error 返回结果
```

轮询示意：

```python
for _ in range(MAX_ATTEMPTS):
    result = client.get_submission_result(submission_id)
    if state not in {"PENDING", "STARTED"}:
        return result
    sleep(0.5)
```

不能无限轮询，所以要设置最大次数。

## 19. requests 是什么

`requests` 是 Python 中非常经典的第三方 HTTP 库。

简单用法：

```python
import requests

r = requests.get("https://example.com")
r = requests.post("https://example.com", json={"a": 1})
```

如果用 requests 写 Aether_lc，大概是：

```python
import requests

session = requests.Session()
session.headers.update(headers)
session.cookies.update(cookies)

response = session.post(
    "https://leetcode.cn/graphql/",
    json=payload,
    timeout=10,
)
response.raise_for_status()
result = response.json()
```

requests 易用、成熟，适合同步脚本和普通 HTTP 请求。

## 20. httpx 是什么

`httpx` 是更现代的第三方 HTTP 客户端。

它的同步 API 和 requests 很像，但还支持：

- `httpx.Client`
- `httpx.AsyncClient`
- 异步请求
- HTTP/2
- 清晰的异常类型

Aether_lc 当前使用同步的 `httpx.Client`，已经足够。

未来如果要批量请求、并发缓存题目详情，`httpx.AsyncClient` 会更自然。

## 21. Python 标准库 HTTP 模块

Python 标准库中也有 HTTP 相关模块：

```text
urllib.request
http.client
```

### urllib.request

可以直接打开 URL：

```python
from urllib.request import urlopen

with urlopen("https://example.com") as response:
    body = response.read()
```

优点：标准库自带，不用安装。

缺点：写复杂请求、Cookie、headers、JSON 会比较麻烦。

### http.client

`http.client` 更底层，通常不直接用于日常项目。

它适合理解协议细节，但不适合作为 Aether_lc 这种项目的主 HTTP 客户端。

## 22. requests、httpx、标准库对比

| 对比项            | requests             | httpx            | urllib.request / http.client |
| -------------- | -------------------- | ---------------- | ---------------------------- |
| 来源             | 第三方库                 | 第三方库             | Python 标准库                   |
| 易用性            | 很高                   | 很高               | 较低                           |
| 同步请求           | 支持                   | 支持               | 支持                           |
| 异步请求           | 不原生支持                | 支持 `AsyncClient` | 不适合                          |
| Client/Session | `requests.Session()` | `httpx.Client()` | 可做但麻烦                        |
| Cookie 管理      | 方便                   | 方便               | 较麻烦                          |
| timeout        | 支持                   | 支持               | 支持但更底层                       |
| 适合 Aether_lc   | 可以                   | 当前更适合            | 不推荐                          |

## 23. 为什么 Aether_lc 用 httpx 合理

Aether_lc 需要：

- 统一 `base_url`。
- 统一 headers。
- 统一 cookies。
- 统一 timeout。
- 处理 JSON。
- 处理 HTTP 状态码。
- 未来可能支持异步或批量请求。

所以 `httpx.Client` 是合理选择。

如果只是写一次性同步脚本，requests 也很好。

如果不想装第三方库，可以用标准库，但代码会更复杂。

## 24. Aether_lc 的 HTTP 数据流

### lc get 2196

```text
用户输入 lc get 2196
  -> cli.py 接收 question_id
  -> service.py 解析题号
  -> service.py 调 client.problem_list 分页查找
  -> client.py POST /graphql/
  -> LeetCode 返回题目索引 JSON
  -> service.py 找到 title_slug
  -> client.py POST /graphql/ 查询题目详情
  -> problem.py 标准化数据
  -> ui.py 展示题目
```

### lc submit

```text
用户输入 lc submit
  -> workspace.py 解析 solution.py 元信息和代码
  -> auth.py 读取 session cookies
  -> client.py POST /problems/{title_slug}/submit/
  -> LeetCode 返回 submission_id
  -> client.py GET /submissions/detail/{submission_id}/check/
  -> service.py 轮询
  -> ui.py 展示判题结果
```

## 25. 最重要的 HTTP 心智模型

学习这个项目时，重点不是死记库 API，而是理解：

```text
HTTP 请求 = method + url + headers + cookies + body
HTTP 响应 = status_code + headers + body
```

还要记住：

- Cookie 是登录态副本，服务端决定是否有效。
- `json=payload` 用于发送 JSON 请求体。
- `response.json()` 可能失败。
- `raise_for_status()` 只处理 HTTP 状态码，不代表业务数据一定正确。
- 网络错误、HTTP 错误、JSON 错误、接口结构错误是不同层次。
- `ClientResult` 是把这些错误变成项目可控结果的方式。

## 26. 后续可能的 HTTP 重构

以后可以考虑抽象：

```python
def _request_json(self, method: str, url: str, **kwargs) -> ClientResult:
    ...
```

它可以统一处理：

- 请求发送。
- `raise_for_status()`。
- `response.json()`。
- `RequestError`。
- `HTTPStatusError`。
- `ValueError`。

但当前不急着做，因为更重要的是：

- 稳定业务字段解析。
- 完善错误提示。
- 实现 `lc doctor`。
- 增加测试覆盖。

## 27. 参考资料

- [HTTPX QuickStart](https://www.python-httpx.org/quickstart/)
- [HTTPX Clients](https://www.python-httpx.org/advanced/clients/)
- [Requests Quickstart](https://requests.readthedocs.io/en/latest/user/quickstart/)
- [Python urllib.request 官方文档](https://docs.python.org/3/library/urllib.request.html)
- [Python http.client 官方文档](https://docs.python.org/3/library/http.client.html)
