# Day 1：微服务架构 · 详细教程

> 今日目标：理解微服务"为什么拆、怎么拆、拆完怎么协作"，并用 FastAPI 亲手搭出
> 「网关 + 用户服务 + 订单服务 + 注册中心」的完整调用链路。
> 时间：约 7.5 小时（理论 2.5h + 实战 3h + 项目 2h）

---

## 1. FastAPI 速成（1 小时，前置技能）

微服务的每个"服务"就是一个独立的 Web 后端程序。Python 里写这种程序最顺手的框架是 **FastAPI**——性能好、自带接口文档、写法和前端思维很接近。先花一小时把它拿下。

### 1.1 你的第一个后端接口

创建文件 `hello.py`：

```python
from fastapi import FastAPI

app = FastAPI()          # 创建应用实例，类似前端创建 Vue/React 应用

@app.get("/hello")       # 路由：URL 路径 /hello 映射到这个函数，类似 Vue Router
def say_hello():
    return {"msg": "hello backend"}   # 返回 dict，FastAPI 自动转成 JSON 响应
```

启动服务：

```bash
uvicorn hello:app --reload --port 8000
```

- `uvicorn` 是 ASGI 服务器，角色类似前端开发时的 `vite dev` / `webpack-dev-server`——负责监听端口、把 HTTP 请求交给你的代码。
- `hello:app` 意思是"hello.py 文件里的 app 对象"。
- `--reload` 类似前端的热更新，改代码自动重启。

**验证**：浏览器打开 `http://127.0.0.1:8000/hello`，看到 `{"msg":"hello backend"}`。

再打开 `http://127.0.0.1:8000/docs` ——这是 FastAPI **自动生成的接口文档（Swagger UI）**，可以直接在页面上点按钮调试接口。前端联调时你一定想要过这种东西，后端用 FastAPI 是免费送的。

### 1.2 三种接收参数的方式

对应前端发请求时携带数据的三种位置：

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# 方式一：路径参数 —— GET /users/42
@app.get("/users/{user_id}")
def get_user(user_id: int):          # 声明 int，FastAPI 自动转换并校验
    return {"user_id": user_id}

# 方式二：查询参数 —— GET /search?keyword=phone&page=2
@app.get("/search")
def search(keyword: str, page: int = 1):   # 有默认值就是可选参数
    return {"keyword": keyword, "page": page}

# 方式三：请求体（JSON）—— POST /users，对应前端 fetch 的 body
class UserIn(BaseModel):     # Pydantic 模型：给 JSON 定义"形状"，类似 TS interface
    name: str
    age: int
    email: str | None = None   # 可选字段

@app.post("/users")
def create_user(user: UserIn):
    return {"received": user.name, "age_next_year": user.age + 1}
```

**Pydantic 的价值**：如果前端传来的 JSON 里 `age` 是字符串 `"abc"`，FastAPI 会自动返回 422 错误并指出哪个字段不合法——入参校验不用自己写，这和你在前端用 PropTypes/TypeScript 是一个思想。

### 1.3 async：和 JS 的 async/await 几乎一样

```python
import asyncio
from fastapi import FastAPI

app = FastAPI()

@app.get("/slow")
async def slow():
    await asyncio.sleep(2)      # 类似 JS 的 await new Promise(r => setTimeout(r, 2000))
    return {"msg": "done"}
```

核心概念一句话：**`async` 函数在等待 IO（网络、数据库、文件）时会挂起自己，让服务器去处理别的请求**，和浏览器事件循环调度 Promise 是同一个模型。后端接口大多涉及数据库/网络调用，所以 `async def` 是 FastAPI 的常态。

> **注意**：Python 的 `await` 后面只能接"协程"（async 函数）。调用数据库、HTTP 时要选支持异步的库（如 `httpx`、`redis.asyncio`），否则会把整个事件循环堵住——类似在 JS 里写了同步的死循环。

### 1.4 小节练习

写一个 `calc.py`：提供 `POST /calc`，接收 `{"a": 1, "b": 2, "op": "add"}`，返回计算结果，op 只支持 add/sub/mul/div，除零返回友好错误。在 `/docs` 页面里调试通过再继续。

---

## 2. 核心概念一：单体与微服务（45 分钟）

### 2.1 从一个大文件说起

回忆你最早写前端：一个 HTML 文件里塞了所有 JS——能跑，但代码一多就乱。后来你学会了拆组件、拆模块。后端系统有完全相同的演进：

- **单体架构（Monolith）**：整个电商系统——用户、商品、订单、支付——所有代码在一个项目里，打包成一个程序部署。就像把所有组件写在一个文件里。
- **微服务架构（Microservices）**：按业务把系统拆成多个**独立部署**的小程序：用户服务管用户、订单服务管订单……各自开发、各自上线、各自扩容。就像把页面拆成独立维护的组件，只不过"组件"变成了"可独立部署的程序"。

### 2.2 为什么要拆？（驱动力）

| 单体的痛点 | 微服务如何解决 |
|---|---|
| 改一行代码要全量发布，发版互相打架 | 服务独立部署，改订单服务只发订单服务 |
| 搜索功能把 CPU 吃满，全站一起卡 | 把搜索拆出去单独扩容机器 |
| 代码库巨大，新人三个月不敢动代码 | 每个服务代码量小，边界清晰 |
| 全公司被一种技术栈绑死 | 不同服务可以用不同语言/框架 |

### 2.3 代价（一定要讲清楚）

微服务不是免费的。拆完之后你立刻面对一堆新问题：

1. **调用从函数调用变成网络调用**——网络会超时、会断、对端会挂；
2. **数据一致性变难**——下单要同时改订单库和库存库，它们现在是两个数据库；
3. **运维复杂度爆炸**——从部署 1 个程序变成部署 N 个，还需要监控、日志聚合；
4. **排查问题变难**——一个请求穿过 5 个服务，出错时追链路很费劲。

> **经验法则**：小团队、业务早期，用"模块化单体"（代码内部按模块组织，但部署为一个程序）就够了；当团队协作和性能瓶颈真的出现时再拆。**为拆而拆是后端新手最常见的过度设计。**

### 2.4 拆分原则（怎么拆才对）

1. **按业务边界拆（DDD 限界上下文）**：拆分的刀口要沿着业务切——用户、商品、订单、支付。不要按技术层切（"所有的查询一个服务、所有的写入一个服务"是错误示范）。
2. **数据私有**：每个服务独占自己的数据库。**这是微服务和"分布式单体"的分水岭**——如果订单服务直接连用户服务的数据库读用户表，那你们只是把一个烂单体拆成了两个互相纠缠的烂单体。
3. **单一职责 + 团队自治**：一个服务由一个小团队完整负责（开发、测试、部署、值班）。康威定律说：系统结构会复制组织的沟通结构——所以拆服务时先看团队怎么分组。
4. **先粗后细**：不确定该不该拆的两个模块，先合在一起；边界会随着业务理解变清晰。

**自检问题**：拆完之后，如果"用户服务挂了"，系统的其他功能还能部分工作吗？能，说明边界基本合理；全站瘫痪，说明耦合还藏在数据或调用里。

---

## 3. 核心概念二：服务间通信（45 分钟）

拆分之后，"下单时检查用户是否存在"这种原本一次函数调用的事，变成了**跨网络的服务调用**。怎么通信、通信出错了怎么办，是微服务的日常。

### 3.1 两种基本方式

**同步通信（请求-响应）**：A 服务发 HTTP 请求给 B 服务，等 B 返回结果再继续。
- 类比前端：`await fetch(...)`。
- 适合：需要立刻拿到结果的场景（下单前查用户、查库存）。

**异步通信（消息队列）**：A 服务把"订单已创建"这件事扔到消息队列里就不管了，短信服务、积分服务各自从队列里取消息慢慢处理。
- 类比前端：EventEmitter 的 `emit` 和 `on`，发事件的人不关心谁在听。
- 适合：不需要立即完成的事（发通知、加积分、同步统计）。
- Day 4 会详细学，今天先建立概念。

### 3.2 同步调用的三大生存法则

网络调用和函数调用最大的区别是**它会失败，而且失败的方式千奇百怪**。所有生产级的服务调用都必须处理这三件事：

**法则一：必须设超时。**
不设超时的调用，等于把你的服务的命运交给对方——对方卡住，你的线程/协程就被挂住，挂多了你的服务也死。一般设 2~5 秒。

**法则二：重试，但只对幂等接口重试。**
网络抖动时重试一次往往就好了。但重试的前提是这个操作**幂等**。

> **幂等（idempotent）**：同一个操作执行 1 次和执行 N 次，效果相同。
> 类比：按电梯按钮，按 1 次和按 10 次，电梯都只会来一次——"按按钮"是幂等的。
> 反例："给账户扣 100 元"执行两次就扣了 200，不幂等。
> 查询（GET）天然幂等，可以随意重试；创建类操作（POST 下单）要靠唯一单号等手段人为做出幂等。

**法则三：熔断与降级。**
如果对方持续故障，每来一个请求你都傻等 2 秒超时，等于被对方拖下水。熔断就是"连续失败 N 次后，一段时间内直接快速失败（返回默认值或友好错误），不再真的调用"，给对方喘息时间，也保护自己。

### 3.3 常见误区

| 误区 | 真相 |
|---|---|
| "内网很快，不用设超时" | 对方服务 GC、重启、死锁时响应会无限期挂起，和内网快慢无关 |
| "POST 失败就重试三次" | 可能重复下单。要么接口幂等，要么不重试，把错误交给上游 |
| "调用链越长越好拆分" | A→B→C→D 的链路，延迟叠加、可用性相乘（每个 99% 则整体约 96%）。链路超过 3 层就该反思设计 |

---

## 4. 核心概念三：API 网关（30 分钟）

### 4.1 解决什么问题

微服务拆完后，前端要面对 5 个服务、5 个地址、5 套端口——就像你要寄快递得记住每家公司的分拣中心地址。API 网关就是**所有请求的统一入口**：

```
前端/App ──▶ API 网关（:8000） ──┬──▶ 用户服务（:8001）
   只认这一个地址               ├──▶ 订单服务（:8002）
                                └──▶ 商品服务（:8003）
```

**前端类比**：网关很像你项目里封装的那个统一 axios 实例——所有请求从这里出去，统一加 token、统一处理错误。只不过它工作在服务端，挡在所有微服务前面。

### 4.2 网关的四大职责

1. **路由转发**：`/users/*` 转给用户服务，`/orders/*` 转给订单服务。
2. **认证鉴权**：统一校验 token，非法请求在门口就拦下，业务服务零鉴权代码。
3. **限流**：每个用户每秒最多 N 次请求，防止恶意刷接口打垮后端。
4. **可观测性**：统一记录访问日志、耗时——所有请求都经过这里，是最好的埋点位置。

业界成熟方案有 Kong、APISIX、Nginx。今天我们用 FastAPI 手写一个迷你网关——不是为了替代它们，而是为了**理解它们内部在干什么**。

---

## 5. 核心概念四：服务注册与发现（30 分钟）

### 5.1 问题：地址是活的

云环境下，服务的实例（IP:端口）是动态变化的：扩容加 2 个实例、发布时旧实例下线新实例上线、容器挂了被调度到别的机器。订单服务想调用用户服务，**不能把 `http://10.0.1.5:8001` 写死在代码里**——明天这个地址就不存在了。

### 5.2 解决方案：注册中心（服务的"通讯录"）

引入一个新角色——注册中心（Registry），工作流程四步走：

```
① 注册：服务启动时告诉注册中心 "我是 user-service，我在 10.0.1.5:8001"
② 心跳：每隔几秒发一次 "我还活着"
③ 发现：订单服务想调用时，问注册中心 "user-service 有哪些活着的实例？"
        拿到地址列表，自己挑一个调用
④ 摘除：某实例心跳超时（如 15 秒没消息），注册中心把它从列表里剔除
```

**前端类比**：这和你用 DNS 很像——浏览器不记 IP，记域名，每次问 DNS 服务器"这个域名现在解析到哪"。注册中心就是微服务世界里的 DNS + 健康检查。

### 5.3 为什么注册中心选 AP 而不是 CP

CAP 理论说分布式系统在**网络分区（P）**时，一致性（C）和可用性（A）只能二选一。注册中心几乎都选 AP：宁可让调用方拿到一份**稍微过时**的实例列表（里面可能有个刚挂掉的实例，调用失败一次再换一个就行），也不能因为注册中心内部数据没同步完，就**一个地址都不返回**——后者会让整个系统瘫痪。

常用注册中心：Nacos（国内主流，还兼配置中心）、Consul、Etcd。今天我们会手写一个 60 行的迷你注册中心，把上面四步跑通。

---

## 6. 代码实战（3 小时）

> 目录结构先建好：
> ```
> backend-bootcamp/
> ├── user_service/main.py
> ├── order_service/main.py
> ├── gateway/main.py
> └── registry/main.py
> ```

### 实战 1：用户服务（30 分钟）

```python
# user_service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI(title="user-service")

# 先用内存字典模拟数据库（Day 3 再换成真的）
USERS = {
    1: {"id": 1, "name": "张三", "level": "vip"},
    2: {"id": 2, "name": "李四", "level": "normal"},
}

class UserIn(BaseModel):
    name: str
    level: str = "normal"

@app.get("/health")                    # 健康检查接口：网关/注册中心用来探活
def health():
    return {"status": "up", "service": "user-service"}

@app.get("/users/{user_id}")
def get_user(user_id: int):
    user = USERS.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="user not found")
    return user

@app.post("/users", status_code=201)
def create_user(user: UserIn):
    new_id = max(USERS) + 1
    USERS[new_id] = {"id": new_id, **user.model_dump()}
    return USERS[new_id]
```

启动（在 `user_service` 目录下）：

```bash
uvicorn main:app --port 8001
```

**预期验证**：
- 浏览器开 `http://127.0.0.1:8001/docs`，试 `GET /users/1` 返回张三；
- 试 `GET /users/99` 返回 404 和 `{"detail":"user not found"}`；
- 试 POST 创建用户，再 GET 新 id 能查到。

**常见报错**：
| 报错 | 原因 |
|---|---|
| `address already in use` | 端口被占，换个端口或关掉旧进程 |
| 404 路径对但就是找不到 | uvicorn 启动目录不对，导致跑的是旧文件 |

### 实战 2：订单服务调用用户服务（45 分钟）

服务间 HTTP 调用用 `httpx`——它就是 Python 版的 axios，API 都像：

```python
# order_service/main.py
import asyncio
import httpx
from fastapi import FastAPI, HTTPException

app = FastAPI(title="order-service")

USER_SERVICE_URL = "http://127.0.0.1:8001"   # 先写死地址，实战 4 再改成注册中心发现

async def fetch_user(user_id: int) -> dict:
    """带超时和有限重试的用户查询（GET 是幂等的，可以放心重试）"""
    last_error = None
    for attempt in range(3):                              # 最多试 3 次
        try:
            async with httpx.AsyncClient(timeout=2.0) as client:   # 法则一：超时
                resp = await client.get(f"{USER_SERVICE_URL}/users/{user_id}")
            if resp.status_code == 200:
                return resp.json()
            if resp.status_code == 404:
                raise HTTPException(status_code=400, detail="user not found")
        except (httpx.TimeoutException, httpx.ConnectError) as e:
            last_error = e
        await asyncio.sleep(0.2 * (attempt + 1))          # 指数退避：0.2s、0.4s
    raise HTTPException(status_code=504, detail=f"user-service unavailable: {last_error}")

@app.post("/orders")
async def create_order(user_id: int, item: str, amount: float):
    user = await fetch_user(user_id)                      # 法则二：只对查询重试
    price = amount * 0.9 if user["level"] == "vip" else amount
    return {
        "order_no": f"ORD-{user_id}-001",
        "user": user["name"],
        "original": amount,
        "price": price,
    }
```

启动（新终端窗口，`order_service` 目录）：

```bash
uvicorn main:app --port 8002
```

**预期验证**：
- `POST /orders?user_id=1&item=键盘&amount=100` → 张三 VIP 打 9 折，price = 90；
- `user_id=99` → 400 user not found；
- **关键实验**：把用户服务停掉（Ctrl+C），再下单 → 约 2.6 秒后收到 504（重试两次 + 退避后放弃）。感受"超时+重试"是怎么保护你的。

**想一想**：如果 `fetch_user` 里调用的是"扣积分"的 POST 接口，这段重试逻辑还敢用吗？为什么？

### 实战 3：手写迷你 API 网关（45 分钟）

```python
# gateway/main.py
import time
import httpx
from fastapi import FastAPI, Request, HTTPException

app = FastAPI(title="mini-gateway")

# 路由表：路径前缀 → 后端服务地址
ROUTES = {
    "users": "http://127.0.0.1:8001",
    "orders": "http://127.0.0.1:8002",
}

# 模拟用户令牌（Day 4 换成真正的 JWT）
TOKENS = {"token-zhangsan": 1, "token-lisi": 2}

# ---------- 职责 2：限流（滑动窗口：每个用户每秒最多 5 次） ----------
buckets: dict[int, list[float]] = {}

def rate_limit(user_id: int, limit: int = 5):
    now = time.time()
    window = [t for t in buckets.get(user_id, []) if now - t < 1.0]  # 只留 1 秒内的记录
    if len(window) >= limit:
        raise HTTPException(status_code=429, detail="rate limit exceeded")
    window.append(now)
    buckets[user_id] = window

# ---------- 职责 4：统一访问日志 ----------
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start = time.time()
    response = await call_next(request)          # 类似 axios 响应拦截器的位置
    cost = (time.time() - start) * 1000
    print(f"[gateway] {request.method} {request.url.path} -> {response.status_code} {cost:.1f}ms")
    return response

# ---------- 职责 1：路由转发 ----------
@app.api_route("/{path:path}", methods=["GET", "POST", "PUT", "DELETE"])
async def proxy(path: str, request: Request):
    # 职责 3：统一鉴权（健康检查路径除外）
    if path != "docs" and not path.startswith("openapi"):
        token = request.headers.get("Authorization", "")
        user_id = TOKENS.get(token)
        if not user_id:
            raise HTTPException(status_code=401, detail="unauthorized")
        rate_limit(user_id)

    prefix = path.split("/")[0]
    base = ROUTES.get(prefix)
    if not base:
        raise HTTPException(status_code=404, detail="no route for this path")

    body = await request.body()
    async with httpx.AsyncClient(timeout=3.0) as client:
        resp = await client.request(
            method=request.method,
            url=f"{base}/{path}",
            params=dict(request.query_params),
            content=body or None,
            headers={"Content-Type": request.headers.get("Content-Type", "application/json")},
        )
    return resp.json()
```

启动：`uvicorn main:app --port 8000`

**预期验证**（前端以后只访问 8000 这一个端口）：
```bash
# 无 token → 401
curl "http://127.0.0.1:8000/users/1"
# 带 token → 200（网关转发给用户服务）
curl -H "Authorization: token-zhangsan" "http://127.0.0.1:8000/users/1"
# 下单走网关 → VIP 9 折
curl -X POST -H "Authorization: token-zhangsan" "http://127.0.0.1:8000/orders?user_id=1&item=键盘&amount=100"
# 快速连刷 10 次 → 第 6 次开始 429
for i in $(seq 10); do curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: token-zhangsan" "http://127.0.0.1:8000/users/1"; done
```

**回头看**：你刚刚手写的这个网关，路由、鉴权、限流、日志四件事俱全。Kong/APISIX 本质上就是把这些做得更强（配置化、插件化、性能更高），核心思想你已经掌握了。

### 实战 4：手写迷你注册中心（60 分钟）

```python
# registry/main.py
import time
from fastapi import FastAPI

app = FastAPI(title="mini-registry")

# 数据结构：{服务名: {实例地址: 最后一次心跳时间戳}}
registry: dict[str, dict[str, float]] = {}
TTL = 15   # 超过 15 秒没心跳，认为实例已下线

@app.post("/register")
def register(service: str, address: str):
    registry.setdefault(service, {})[address] = time.time()
    print(f"[registry] {service} @ {address} registered")
    return {"msg": "registered"}

@app.post("/heartbeat")
def heartbeat(service: str, address: str):
    if address in registry.get(service, {}):
        registry[service][address] = time.time()
        return {"msg": "ok"}
    raise HTTPException(status_code=404, detail="not registered")

@app.get("/discover/{service}")
def discover(service: str):
    now = time.time()
    instances = registry.get(service, {})
    alive = [addr for addr, ts in instances.items() if now - ts < TTL]   # 过滤掉超时实例
    return {"service": service, "instances": alive}
```

启动：`uvicorn main:app --port 9000`

然后**改造用户服务**，让它启动时注册、并持续发心跳。在 `user_service/main.py` 末尾追加：

```python
import asyncio
import httpx

REGISTRY = "http://127.0.0.1:9000"
MY_ADDRESS = "http://127.0.0.1:8001"

async def heartbeat_loop():
    # 启动时先注册一次
    async with httpx.AsyncClient() as client:
        await client.post(f"{REGISTRY}/register",
                          params={"service": "user-service", "address": MY_ADDRESS})
    # 之后每 5 秒心跳一次
    while True:
        await asyncio.sleep(5)
        try:
            async with httpx.AsyncClient(timeout=2.0) as client:
                await client.post(f"{REGISTRY}/heartbeat",
                                  params={"service": "user-service", "address": MY_ADDRESS})
        except httpx.HTTPError:
            pass   # 注册中心暂时挂了也没关系，下一轮再试

@app.on_event("startup")
async def start_heartbeat():
    asyncio.create_task(heartbeat_loop())   # 后台协程，不阻塞服务本身
```

**改造订单服务**：不再写死地址，而是先发现再调用。把实战 2 中的 `USER_SERVICE_URL` 替换为：

```python
import random

REGISTRY = "http://127.0.0.1:9000"

async def discover(service: str) -> str:
    """向注册中心查询服务地址，随机挑一个实例（最朴素的负载均衡）"""
    async with httpx.AsyncClient(timeout=2.0) as client:
        resp = await client.get(f"{REGISTRY}/discover/{service}")
    instances = resp.json()["instances"]
    if not instances:
        raise HTTPException(status_code=503, detail=f"no instance of {service}")
    return random.choice(instances)

# fetch_user 内部改为：
#     base = await discover("user-service")
#     resp = await client.get(f"{base}/users/{user_id}")
```

**预期验证（这是今天最有成就感的一个实验）**：
1. 依次启动 registry、user_service、order_service；
2. 查 `http://127.0.0.1:9000/discover/user-service` → 返回 `{"instances":["http://127.0.0.1:8001"]}`；
3. 下单成功（订单服务通过注册中心找到了用户服务）；
4. **停掉用户服务，等 15 秒**，再查 discover → `instances` 变成空列表（心跳超时摘除生效）；
5. 此时下单 → 503 "no instance of user-service"，错误明确而不是卡死。

---

## 7. 练手项目：迷你电商服务拆分（2 小时）

**任务**：把今天学的全部组装起来，搭一个四组件的迷你电商后端。

**规格**：

| 组件 | 端口 | 职责 |
|---|---|---|
| registry | 9000 | 服务注册/心跳/发现 |
| gateway | 8000 | 对外唯一入口：鉴权 + 限流 + 路由 + 日志 |
| user-service | 8001 | 用户查询、注册；注册到 registry |
| product-service | 8003 | 商品列表、库存查询与扣减（内存模拟）；注册到 registry |
| order-service | 8002 | 下单：查用户（发现+调用）→ 查商品并扣库存 → 生成订单 |

**下单链路要求**：`POST /orders` 需携带网关 token；order-service 通过 registry 发现 user-service 和 product-service 的地址；VIP 用户 9 折；库存不足返回 400；任一依赖服务下线时返回明确错误（超时 + 503）。

**验收标准**：
1. 通过网关下单成功，VIP 折扣正确；
2. 停掉 product-service 等 15 秒后下单，收到 503 而不是卡死；
3. 同一 token 连刷 10 次，第 6 次起 429；
4. 画一张你的系统调用关系图（手绘也行），能向"面试官"讲清每个组件存在的理由。

---

## 8. Day 1 自测清单

不看资料能回答以下问题，今天才算过关：

- [ ] 说出微服务拆分的 4 条原则，并举一个"不该拆"的场景
- [ ] 为什么微服务之间绝不能共享数据库？
- [ ] 什么是幂等？举生活和技术中各一个例子。为什么重试只对幂等接口安全？
- [ ] 同步调用三大生存法则是什么？（超时/重试/熔断）
- [ ] API 网关的四大职责是什么？它和前端的统一 axios 实例像在哪？
- [ ] 完整口述注册中心的"注册→心跳→发现→摘除"流程；TTL 起什么作用？
- [ ] 注册中心为什么通常选 AP？
- [ ] 实操：关掉所有代码，40 分钟内重新搭出"网关 + 两个服务 + 注册中心"链路

**明天预告**：今天所有服务的数据都在内存字典里，重启就没了，而且慢查询会直接拖垮服务。明天学 Redis——后端的"共享内存层"，看大厂是怎么用它扛住千万级请求的。
