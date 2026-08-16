# Python 后端开发 4 天强化学习教程

> 适用对象：已掌握 Python 基础语法，希望系统性提升后端实战能力的学习者
> 技术栈：Python 3.11+ / FastAPI / Redis / SQLAlchemy 2.x / MySQL 8 / RabbitMQ / Docker
> 总时长：4 天，每天约 7~8 小时（含练习与项目）

---

## 目录

- [第零部分：整体学习大纲](#第零部分整体学习大纲)
- [环境准备（开课前 30 分钟）](#环境准备开课前-30-分钟)
- [Day 1：微服务架构](#day-1微服务架构)
- [Day 2：Redis 应用](#day-2redis-应用)
- [Day 3：数据库开发 + 综合实战项目](#day-3数据库开发--综合实战项目)
- [Day 4：消息队列、Docker 部署与认证安全](#day-4消息队列docker-部署与认证安全)
- [结业验收标准](#结业验收标准)

---

## 第零部分：整体学习大纲

| 天数 | 主题模块 | 学习目标 | 知识点清单 | 预计耗时 |
|------|----------|----------|------------|----------|
| Day 1 | 微服务架构 | 能将单体应用按业务边界拆分为多个服务，并实现服务间通信、统一入口与服务注册发现 | 单体 vs 微服务、服务拆分原则（DDD 限界上下文）、同步通信（REST/gRPC）、异步通信（消息队列）、API 网关（路由/认证/限流）、服务注册与发现（Nacos/Consul 原理）、超时/重试/幂等 | 7.5 h（理论 2.5h + 实战 3h + 项目 2h） |
| Day 2 | Redis 应用 | 能根据业务场景选择合适的数据结构，能设计安全的缓存方案与分布式锁 | 五大数据结构及扩展结构、缓存穿透/击穿/雪崩的成因与对策、Cache-Aside 模式、布隆过滤器、分布式锁（SET NX EX、唯一标识、Lua 原子释放、看门狗续期）、缓存与数据库一致性 | 7.5 h（理论 2.5h + 实战 3h + 项目 2h） |
| Day 3 | 数据库开发 + 综合项目 | 能熟练使用 ORM、设计索引、优化 SQL、管理事务与连接池；并通过秒杀系统串联三天全部知识 | SQLAlchemy 2.0 实践、N+1 问题、SQL 执行计划（EXPLAIN）、索引设计（B+Tree、最左前缀、覆盖索引）、事务 ACID 与隔离级别、连接池管理、综合项目：秒杀系统 | 8 h（理论 2.5h + 实战 1.5h + 综合项目 4h） |
| Day 4 | 消息队列 + Docker 部署 + 认证安全 | 能用 RabbitMQ 改造异步链路、用 docker-compose 一键编排整套系统、实现生产级认证与权限控制 | Exchange/Queue/ACK/死信队列、消息可靠性三板斧、Dockerfile 与 compose 编排、bcrypt 密码哈希、JWT、RBAC、常见 Web 攻击防护 | 8 h（理论 2.5h + 实战 4h + 项目 1.5h） |

**难度曲线**：Day 1 建立架构思维（偏概念+动手）→ Day 2 深入中间件原理（概念+代码并重）→ Day 3 落到数据层并完成全栈综合项目（重实战）→ Day 4 打通工程化最后一公里：异步消息、容器化交付与安全防护。

---

## 环境准备（开课前 30 分钟）

```bash
# 1. 创建虚拟环境
python -m venv venv
# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

# 2. 安装依赖
pip install fastapi uvicorn httpx redis sqlalchemy pymysql cryptography \
    pika PyJWT bcrypt

# 3. 用 Docker 启动 Redis、MySQL 和 RabbitMQ（推荐）
docker run -d --name redis -p 6379:6379 redis:7
docker run -d --name mysql -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -e MYSQL_DATABASE=shop mysql:8
# Day 4 才用到：5672 是服务端口，15672 是管理后台（账号密码 guest/guest）
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management

# 没有 Docker 的替代方案：
# - Redis：Windows 可用 Memurai 或微软版 Redis；Mac 用 brew install redis
# - MySQL：可先用 SQLite 学习 ORM 与事务（SQLAlchemy 改一行连接串即可），
#   但 Day 3 的 EXPLAIN 与索引实战建议还是用 MySQL
```

验证：执行 `uvicorn main:app --reload` 能启动、`redis-cli ping` 返回 `PONG`，即环境就绪。

---

# Day 1：微服务架构

## 1.1 学习目标

完成今天学习后，你应当能够：

1. 说清楚什么场景该拆微服务、什么场景不该拆；
2. 按业务边界把一个电商单体拆成用户/商品/订单服务；
3. 用 FastAPI 实现两个服务并通过 HTTP 互相调用；
4. 实现一个简易 API 网关（路由转发 + 鉴权 + 限流）；
5. 理解服务注册与发现的工作机制，并能手写一个迷你注册中心。

## 1.2 知识点清单与时间安排

| 时段 | 内容 | 耗时 |
|------|------|------|
| 上午 1 | 理论：单体 vs 微服务、拆分原则 | 1.5 h |
| 上午 2 | 理论：服务间通信 + API 网关 + 注册发现 | 1 h |
| 下午 1 | 实战 1-2：FastAPI 双服务 + 服务间调用 | 1.5 h |
| 下午 2 | 实战 3-4：简易网关 + 迷你注册中心 | 1.5 h |
| 下午 3 | 练手项目：迷你电商服务拆分 | 2 h |

## 1.3 理论讲解

### 知识点 1：单体 vs 微服务、服务拆分原则

**理论要点**

- 单体架构：所有功能打包在一个进程里。优点是开发简单、部署简单、本地事务强一致；缺点是代码耦合、改一处全量发布、无法按模块独立扩容。
- 微服务架构：按业务边界拆成多个独立部署的服务，各自拥有自己的数据库。优点是独立开发/部署/扩容、技术栈自由；代价是引入了分布式系统的全部复杂度（网络不可靠、分布式事务、运维成本）。
- **拆分原则（重点）**：
  1. **单一职责 + 业务边界**：按领域驱动设计（DDD）的"限界上下文"拆分。电商典型拆法：用户服务、商品服务、订单服务、支付服务、库存服务。
  2. **数据私有**：每个服务独占自己的数据库，禁止跨服务直接连库——这是微服务和"分布式单体"的分水岭。
  3. **康威定律**：系统结构会复制组织的沟通结构。团队怎么分组，服务就怎么拆。
  4. **拆分粒度经验法则**：一个服务由一个小团队（2 pizza team）维护；拆到你"讲得清边界"为止，宁可先粗后细。

**典型应用场景**

- 团队超过 10 人、单体发布互相打架 → 拆分；
- 某个模块（如搜索、图片处理）资源消耗远高于其他模块 → 单独拆出独立扩容；
- 初创项目 MVP 阶段 → **不要拆**，先用"模块化单体"（代码内部分层分模块），等业务稳定再拆。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 微服务是不是越多越好？ | 不是。服务数量每多一个，运维、调用链、排障成本都上升。拆分的收益必须大于分布式复杂度带来的成本。 |
| 两个服务能不能共用一个数据库？ | 不能。共享库意味着 schema 耦合，一方改表另一方就可能挂，等于回到了单体。正确做法是通过 API 获取对方数据。 |
| 什么时候不该用微服务？ | 团队小、业务早期、没有 DevOps 能力（CI/CD、监控、日志）时，微服务是负担不是收益。 |

### 知识点 2：服务间通信

**理论要点**

- **同步通信**：
  - REST/HTTP：最简单通用，Python 侧用 `httpx`/`requests` 调用。缺点：性能一般、强耦合于对端可用性。
  - gRPC：基于 HTTP/2 + Protobuf，性能高、有强类型契约，适合内部高频调用。
- **异步通信（消息队列）**：生产者把消息发到 MQ（RabbitMQ/Kafka/Redis Stream），消费者异步处理。带来三大好处：**解耦**（下游挂了不阻塞上游）、**削峰**（秒杀流量先堆在队列里慢慢消费）、**最终一致性**。
- 同步调用必须考虑的三件事：**超时**（必须设，不设超时等于把命运交给对端）、**重试**（只对幂等接口重试，指数退避）、**熔断降级**（对端持续失败时快速失败，避免线程池被拖垮）。

**典型应用场景**

- 下单需要实时校验用户信息 → 同步 REST/gRPC；
- 下单后发短信、加积分、更新统计 → 异步 MQ，主流程不等待；
- 秒杀下单 → 请求先入 MQ，消费者按能力消化，保护数据库。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 重试导致重复下单怎么办？ | 接口幂等设计：客户端生成唯一请求号（如订单号），服务端用唯一索引或"请求号去重表"保证同一请求只生效一次。 |
| 同步调用链太长（A→B→C→D）有什么风险？ | 延迟叠加、任一环节失败全链路失败。长链路应考虑异步化或合并服务。 |
| 什么时候用 gRPC 而不是 REST？ | 内部服务间高频、对延迟敏感、希望有接口契约（.proto）时。对外公开 API 仍用 REST。 |

### 知识点 3：API 网关

**理论要点**

- 网关是所有客户端请求的**统一入口**，位于客户端与微服务之间。
- 核心职责：**路由转发**（按路径把请求转给对应服务）、**认证鉴权**（统一校验 Token，业务服务不再各自验证）、**限流熔断**（保护后端）、**日志监控**（统一埋点）、**协议聚合**（一个聚合接口代替客户端多次调用）。
- 业界方案：Kong、APISIX、Nginx+Lua、Spring Cloud Gateway。今天我们用 FastAPI 手写一个迷你版理解原理。

**典型应用场景**

- 移动端首页需要聚合用户、商品、订单三个服务的数据 → 网关做聚合层；
- 全站统一 JWT 校验、统一限流（如单用户 100 次/秒）→ 网关层实现，业务服务零侵入。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 网关成了单点怎么办？ | 网关本身无状态，可水平扩容多实例，前面挂负载均衡（Nginx/云 LB）。 |
| 网关在微服务里的定位 vs Nginx？ | Nginx 偏流量入口（四层/七层负载），API 网关偏业务（鉴权、聚合、限流策略），实际常两者叠加使用。 |

### 知识点 4：服务注册与发现

**理论要点**

- 问题背景：服务实例的 IP:Port 是动态变化的（扩容、重启、容器漂移），调用方不能写死地址。
- **工作流程**：服务启动时向注册中心**注册**自己的地址 → 定时发送**心跳**续约 → 调用方从注册中心**拉取**可用实例列表并本地缓存 → 实例下线/心跳超时后被**摘除**。
- **两种发现模式**：客户端发现（调用方自己查注册中心并做负载均衡，如早期 Eureka+Ribbon）；服务端发现（通过网关/负载均衡器转发，如 K8s Service）。
- 常用注册中心：Nacos（国内主流，兼配置中心）、Consul、Etcd、Zookeeper。核心权衡是 CAP：注册中心通常选 AP（可用性优先），因为短时间内拿到过期的实例列表比拿不到列表好。

**典型应用场景**

- 订单服务需要调用用户服务：不写死 `http://10.0.1.5:8001`，而是问注册中心"user-service 有哪些健康实例"，随机/轮询挑一个调用。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 注册中心挂了，服务还能互相调用吗？ | 能撑一阵。调用方本地有实例列表缓存，短时间内可继续调用；但新实例无法注册、下线实例无法感知。所以注册中心要做集群。 |
| 心跳间隔设多少合适？ | 常见 5~30 秒。太短注册中心压力大，太长故障发现慢。实例异常剔除通常容忍 2~3 次心跳丢失。 |

## 1.4 代码实战

### 实战 1：FastAPI 搭建用户服务（30 分钟）

```python
# user_service/main.py
from fastapi import FastAPI, HTTPException

app = FastAPI(title="user-service")

USERS = {
    1: {"id": 1, "name": "张三", "level": "vip"},
    2: {"id": 2, "name": "李四", "level": "normal"},
}

@app.get("/health")
def health():
    return {"status": "up", "service": "user-service"}

@app.get("/users/{user_id}")
def get_user(user_id: int):
    user = USERS.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="user not found")
    return user
```

启动：`uvicorn main:app --port 8001`

**练习**：给服务加上 `/users` 列表接口和一个创建用户的 POST 接口，用 Pydantic 模型做入参校验。

### 实战 2：订单服务调用用户服务（30 分钟）

```python
# order_service/main.py
import httpx
from fastapi import FastAPI, HTTPException

app = FastAPI(title="order-service")

USER_SERVICE_URL = "http://127.0.0.1:8001"   # 先写死，实战 4 改为从注册中心获取

@app.post("/orders")
async def create_order(user_id: int, item: str, amount: float):
    # 同步调用用户服务校验用户，必须设置超时
    async with httpx.AsyncClient(timeout=2.0) as client:
        try:
            resp = await client.get(f"{USER_SERVICE_URL}/users/{user_id}")
        except httpx.TimeoutException:
            raise HTTPException(status_code=504, detail="user-service timeout")
    if resp.status_code != 200:
        raise HTTPException(status_code=400, detail="invalid user")
    user = resp.json()
    # VIP 用户打 9 折，体现"跨服务数据影响业务逻辑"
    price = amount * 0.9 if user["level"] == "vip" else amount
    return {"order_no": f"ORD-{user_id}-001", "user": user["name"], "price": price}
```

启动：`uvicorn main:app --port 8002`

**练习**：给这段调用加上"最多重试 2 次、指数退避"的逻辑；思考为什么这里只读接口可以安全重试。

### 实战 3：手写迷你 API 网关（45 分钟）

```python
# gateway/main.py
import time
import httpx
from fastapi import FastAPI, Request, HTTPException

app = FastAPI(title="mini-gateway")

ROUTES = {
    "/users": "http://127.0.0.1:8001",
    "/orders": "http://127.0.0.1:8002",
}
TOKENS = {"token-zhangsan": 1, "token-lisi": 2}

# 简易令牌桶限流：每个用户 5 次/秒
buckets: dict[int, list[float]] = {}

def rate_limit(user_id: int, limit: int = 5):
    now = time.time()
    window = [t for t in buckets.get(user_id, []) if now - t < 1]
    if len(window) >= limit:
        raise HTTPException(status_code=429, detail="rate limit exceeded")
    window.append(now)
    buckets[user_id] = window

@app.api_route("/{path:path}", methods=["GET", "POST"])
async def proxy(path: str, request: Request):
    # 1. 统一鉴权
    token = request.headers.get("Authorization", "")
    user_id = TOKENS.get(token)
    if not user_id:
        raise HTTPException(status_code=401, detail="unauthorized")
    # 2. 统一限流
    rate_limit(user_id)
    # 3. 路由转发
    prefix = "/" + path.split("/")[0]
    base = ROUTES.get(prefix)
    if not base:
        raise HTTPException(status_code=404, detail="no route")
    body = await request.body()
    async with httpx.AsyncClient(timeout=3.0) as client:
        resp = await client.request(
            method=request.method,
            url=f"{base}/{path}",
            params=dict(request.query_params),
            content=body or None,
        )
    return resp.json()
```

启动：`uvicorn main:app --port 8000`，之后所有请求统一走 `http://127.0.0.1:8000`。

**练习**：给网关加请求日志中间件，记录每次调用的路径、耗时、状态码——这就是可观测性的雏形。

### 实战 4：迷你注册中心（45 分钟）

```python
# registry/main.py  —— 一个 60 行的注册中心，理解原理用
import time
from fastapi import FastAPI

app = FastAPI(title="mini-registry")

# 结构：{服务名: {实例地址: 最后心跳时间}}
registry: dict[str, dict[str, float]] = {}
TTL = 15  # 15 秒无心跳视为下线

@app.post("/register")
def register(service: str, address: str):
    registry.setdefault(service, {})[address] = time.time()
    return {"msg": "registered"}

@app.post("/heartbeat")
def heartbeat(service: str, address: str):
    if address in registry.get(service, {}):
        registry[service][address] = time.time()
        return {"msg": "ok"}
    return {"msg": "not registered"}, 404

@app.get("/discover/{service}")
def discover(service: str):
    now = time.time()
    instances = registry.get(service, {})
    alive = [addr for addr, ts in instances.items() if now - ts < TTL]
    return {"service": service, "instances": alive}
```

然后改造实战 2 的订单服务：启动时向注册中心注册自己，并每 5 秒发一次心跳；调用用户服务前先 `discover("user-service")` 拿地址列表，随机选一个。

**练习**：思考并验证——把用户服务停掉 15 秒后，发现接口返回什么？这就是"心跳超时摘除"。

## 1.5 练手项目：迷你电商服务拆分（2 小时）

**需求**：把下面这个假想的单体电商拆成微服务并跑通完整下单链路：

- `user-service`（8001）：用户查询、注册；
- `product-service`（8003）：商品列表、库存查询（库存先写死在内存字典里）；
- `order-service`（8002）：创建订单——需调用用户服务校验用户、调用商品服务校验并扣减库存；
- `gateway`（8000）：统一入口，鉴权 + 限流 + 路由；
- `registry`（9000）：所有业务服务注册到它，服务间调用通过它发现地址。

**验收标准**：

1. 通过网关 `POST /orders` 下单成功，且 VIP 用户价格打 9 折；
2. 停掉 product-service 后下单，能得到明确的错误提示而不是卡死（超时生效）；
3. 同一 token 连续刷 10 次请求，第 6 次开始返回 429；
4. 画出你的服务调用关系图，能讲清每次请求经过了哪些组件。

## 1.6 Day 1 自测检验标准

能不看资料回答以下问题，即视为达标：

- [ ] 说出微服务拆分的 3 条原则，并举一个"不该拆"的反例
- [ ] 为什么微服务之间不能共享数据库？
- [ ] 同步调用必须处理的三个问题是什么？（超时/重试/熔断）
- [ ] 什么情况下重试是安全的？（幂等）
- [ ] API 网关的 4 个核心职责是什么？
- [ ] 口述服务注册与发现的完整流程（注册→心跳→发现→摘除）
- [ ] 实操：30 分钟内不看示例，独立搭出"网关 + 两个服务"的调用链路

---

# Day 2：Redis 应用

## 2.1 学习目标

1. 看到业务场景能立刻说出该用哪种 Redis 数据结构；
2. 能讲透缓存穿透、击穿、雪崩的区别，并写出对应的防护代码；
3. 能手写一个生产可用的分布式锁（含防误删与原子释放）；
4. 理解缓存与数据库一致性问题的常见解法。

## 2.2 知识点清单与时间安排

| 时段 | 内容 | 耗时 |
|------|------|------|
| 上午 1 | 理论：五大数据结构 + 扩展结构及场景 | 1.5 h |
| 上午 2 | 理论：缓存三大问题 + 一致性方案 | 1 h |
| 下午 1 | 实战 1-2：数据结构操作 + 缓存防护代码 | 1.5 h |
| 下午 2 | 实战 3：分布式锁 | 1.5 h |
| 下午 3 | 练手项目：带缓存与锁的商品服务 | 2 h |

## 2.3 理论讲解

### 知识点 1：常用数据结构及适用场景

| 数据结构 | 底层（了解） | 典型场景 | 关键命令 |
|----------|--------------|----------|----------|
| String | SDS 动态字符串 | 计数器（阅读量）、Session/Token、缓存对象 JSON、分布式锁 | `SET/GET/INCR/SETEX/SET NX` |
| Hash | ziplist/hashtable | 存对象（用户信息），可单独更新某字段，比整个 JSON 序列化省内存 | `HSET/HGET/HINCRBY` |
| List | quicklist | 简单消息队列、最新列表（最新 10 条评论）、时间线 | `LPUSH/RPOP/LRANGE/BRPOP` |
| Set | intset/hashtable | 标签、去重、共同好友/共同关注（交集） | `SADD/SISMEMBER/SINTER` |
| ZSet | skiplist | 排行榜（分数排序）、延迟队列（时间戳当分数）、权重推荐 | `ZADD/ZRANGE/ZRANGEBYSCORE` |
| Bitmap（扩展） | String 位操作 | 用户签到、在线状态，亿级用户只占 MB 级内存 | `SETBIT/GETBIT/BITCOUNT` |
| HyperLogLog（扩展） | 概率结构 | UV 统计，误差约 0.81%，固定 12KB | `PFADD/PFCOUNT` |

**典型应用场景速记**：计数用 String，对象用 Hash，排队用 List，去重交友用 Set，排名延时用 ZSet，签到用 Bitmap，数 UV 用 HyperLogLog。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 存用户对象用 Hash 还是 JSON 字符串？ | 字段需要单独读写（如只更新积分）用 Hash；整体读写、结构复杂用 JSON 字符串更简单。 |
| List 能当专业消息队列用吗？ | 只能算"够用"。`LPUSH+BRPOP` 没有 ACK、没有消息持久化语义，丢消息风险高。严肃场景用 Redis Stream 或 RabbitMQ/Kafka。 |
| 什么是大 Key 和热 Key？ | 大 Key：单个 value 过大（如百万成员的 Set），删除和传输都会阻塞 Redis——要拆分。热 Key：单个 Key 被超高频访问，压垮单节点——要本地缓存或 Key 拆散。 |

### 知识点 2：缓存策略——穿透 / 击穿 / 雪崩

先明确最基础的 **Cache-Aside（旁路缓存）模式**，这是 90% 业务的标准用法：

```
读：先查缓存 → 命中直接返回 → 未命中查数据库 → 写入缓存 → 返回
写：先更新数据库 → 再删除缓存（注意：是"删除"不是"更新"）
```

**三大问题对比（必考必问）**：

| 问题 | 成因 | 一句话区分 | 解决方案 |
|------|------|------------|----------|
| 缓存**穿透** | 查询一个**数据库里根本不存在**的数据，缓存永远 miss，请求全部打到 DB | 查"没有的数据" | ① 缓存空值（`key→null`，TTL 30~60s）；② 布隆过滤器拦截不存在的 Key |
| 缓存**击穿** | 某个**热点 Key 恰好过期**，瞬间大量并发同时去重建缓存 | 一个"热点 Key"过期 | ① 互斥锁：只允许一个线程重建，其他等待；② 逻辑过期：不设 TTL，异步线程刷新 |
| 缓存**雪崩** | **大量 Key 同时过期**，或 Redis 宕机，DB 被打爆 | 一批 Key 同时失效 | ① TTL 加随机值（如 300s ± 50s）错峰过期；② 多级缓存（本地缓存+Redis）；③ Redis 高可用（哨兵/集群）+ 熔断降级兜底 |

**典型应用场景**

- 恶意请求用不存在的商品 ID 刷接口 → 穿透，用空值缓存或布隆过滤器；
- 微博热搜、秒杀商品详情 → 击穿风险，热点 Key 用互斥重建；
- 批量导入数据时设了相同 TTL → 雪崩隐患，必须加随机过期时间。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 为什么写操作是"删缓存"而不是"更新缓存"？ | 更新缓存在并发下易产生脏数据（两个写请求交叉更新缓存和 DB 的顺序不可控）；删除缓存让下次读时重建，逻辑更简单。且若该 Key 很少被读，更新缓存是浪费。 |
| 缓存和数据库一致性怎么保证？ | 没有完美的强一致方案，工程上追求最终一致：① 先更 DB 再删缓存（可能出现旧数据，概率低）；② 延迟双删（更新前后各删一次）；③ 订阅 binlog（Canal）异步删缓存，最常用的大厂方案；④ 给缓存设较短 TTL 作为兜底。 |
| 布隆过滤器有什么坑？ | 有误判（存在的说存在，不存在的也可能说存在——但实际不存在），且不支持删除（可用布谷鸟过滤器替代）。元素量要预估好，否则误判率飙升。 |

### 知识点 3：分布式锁

**理论要点**

- 单机锁（`threading.Lock`）在多实例部署下完全失效——每个进程一把锁，形同虚设。分布式场景需要跨进程互斥，Redis 是最常用实现。
- **演进之路（理解每一步在解决什么问题）**：
  1. `SETNX` + 事后 `EXPIRE`：两条命令不是原子的，加锁后还没设过期就宕机 → 死锁；
  2. `SET key value NX EX 30`：加锁和过期原子化，解决死锁；
  3. **value 存唯一标识（UUID）**：防止"业务执行太久锁过期了，被别的线程拿到锁，自己执行完把别人的锁删了"——即误删问题；
  4. **Lua 脚本原子释放**："判断是不是自己的锁 + 删除"必须原子执行，否则判断完、删除前锁刚好过期被他人获取，还是会误删；
  5. **看门狗续期**：业务没执行完锁快过期了，后台线程自动续期（Redisson 的核心机制）；
  6. 争议与边界：Redis 主从切换时锁可能丢失（异步复制），对强一致要求极高的场景（金融扣款）应改用数据库乐观锁/Zookeeper，RedLock 方案在业内有争议，了解即可。

**典型应用场景**

- 防止缓存击穿时的"互斥重建"；
- 秒杀扣库存、抢优惠券——同一资源只允许一个请求操作；
- 定时任务多实例部署，保证只有一个实例真正执行。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 锁过期时间设多少？ | 没有银弹。常规设 10~30s + 看门狗续期；没有续期机制时，设"预估业务耗时 × 3"。 |
| 拿不到锁的线程怎么办？ | 看业务：秒杀类直接快速失败返回"稍后再试"；任务类可以自旋重试（带次数上限和退避）或用 `BLPOP` 阻塞等待。 |
| Redis 分布式锁能保证 100% 安全吗？ | 不能。主从切换、时钟跳变、GC 停顿都可能出问题。它是"高可用场景下的工程最优解"，不是数学意义上的绝对互斥。 |

## 2.4 代码实战

### 实战 1：数据结构操作练习（30 分钟）

```python
# redis_practice.py
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

# String：文章阅读量计数
r.incr("article:1001:views")

# Hash：用户信息，单独更新积分字段
r.hset("user:1", mapping={"name": "张三", "points": 100})
r.hincrby("user:1", "points", 20)

# ZSet：热搜排行榜
r.zadd("hot:rank", {"Python教程": 98, "Redis实战": 85, "微服务入门": 76})
print(r.zrevrange("hot:rank", 0, 2, withscores=True))   # Top 3

# Set：共同关注
r.sadd("user:1:follows", "a", "b", "c")
r.sadd("user:2:follows", "b", "c", "d")
print(r.sinter("user:1:follows", "user:2:follows"))     # {'b', 'c'}

# Bitmap：用户签到（2026 年第 227 天签到）
r.setbit("sign:1:2026", 227, 1)
print(r.bitcount("sign:1:2026"))                        # 累计签到天数
```

**练习**：用 ZSet 实现一个延迟队列——`ZADD delay_queue <当前时间戳+延迟秒数> 任务`，消费者轮询 `ZRANGEBYSCORE 0 now` 取出到期任务。

### 实战 2：缓存穿透/击穿/雪崩防护（45 分钟）

```python
# cache_guard.py
import json
import random
import time
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def fake_db_query(product_id: int):
    """模拟数据库查询：只有 1~100 的商品存在"""
    time.sleep(0.1)
    if 1 <= product_id <= 100:
        return {"id": product_id, "name": f"商品{product_id}", "price": 99.0}
    return None

def get_product(product_id: int):
    key = f"product:{product_id}"
    # 1. 查缓存
    cached = r.get(key)
    if cached is not None:
        return None if cached == "null" else json.loads(cached)  # 命中空值缓存 → 防穿透

    # 2. 互斥锁防击穿：只有一个请求去重建缓存
    lock_key = f"lock:product:{product_id}"
    if r.set(lock_key, "1", nx=True, ex=10):
        try:
            data = fake_db_query(product_id)
            if data is None:
                r.set(key, "null", ex=60)                        # 缓存空值，防穿透
            else:
                # TTL 加随机值，防雪崩
                r.set(key, json.dumps(data), ex=300 + random.randint(0, 60))
            return data
        finally:
            r.delete(lock_key)
    else:
        # 没拿到锁：短暂等待后重读缓存
        time.sleep(0.05)
        cached = r.get(key)
        return json.loads(cached) if cached and cached != "null" else None
```

**练习**：用 `SETBIT` 手写一个简单布隆过滤器（多次哈希置位），放在 `get_product` 之前拦截不存在的 ID，对比拦截前后的 DB 查询次数。

### 实战 3：生产可用的分布式锁（45 分钟）

```python
# distributed_lock.py
import asyncio
import uuid
import redis.asyncio as aioredis

UNLOCK_SCRIPT = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
"""

class RedisLock:
    def __init__(self, r: aioredis.Redis, key: str, expire: int = 30):
        self.r = r
        self.key = key
        self.expire = expire
        self.token = str(uuid.uuid4())   # 唯一标识，防误删

    async def acquire(self, retry: int = 3, interval: float = 0.1) -> bool:
        for _ in range(retry):
            ok = await self.r.set(self.key, self.token, nx=True, ex=self.expire)
            if ok:
                return True
            await asyncio.sleep(interval)
        return False

    async def release(self):
        # Lua 脚本保证"校验身份 + 删除"原子执行
        await self.r.eval(UNLOCK_SCRIPT, 1, self.key, self.token)

    async def __aenter__(self):
        if not await self.acquire():
            raise RuntimeError("acquire lock failed")
        return self

    async def __aexit__(self, *exc):
        await self.release()

# 使用示例：秒杀扣库存
async def seckill(r, stock_key: str):
    async with RedisLock(r, "lock:seckill", expire=10):
        stock = int(await r.get(stock_key) or 0)
        if stock <= 0:
            return "sold out"
        await r.decr(stock_key)
        return "success"
```

**练习**：用 `asyncio.gather` 并发起 50 个协程抢 10 件库存，验证最终库存恰好扣到 0、成功人数恰好 10 人；然后把锁去掉再跑一次，观察超卖。

## 2.5 练手项目：带缓存与分布式锁的商品服务（2 小时）

**需求**：在 Day 1 的 product-service 基础上升级：

1. **商品详情加缓存**：Cache-Aside 模式，TTL 300s + 随机值；查询不存在的商品要防穿透；
2. **热点商品防击穿**：重建缓存时加互斥锁；
3. **库存扣减加分布式锁**：`POST /products/{id}/deduct` 接口，用 RedisLock 保证不超卖；
4. **缓存一致性**：`PUT /products/{id}` 更新商品时，先更新数据（内存字典模拟 DB）再删除缓存；
5. **压测验证**：用 `ab` 或 Python 并发脚本，对比加缓存前后 QPS，并验证并发扣库存不超卖。

**验收标准**：

- 连续请求同一商品 100 次，模拟 DB 的查询函数只被调用 1~2 次；
- 50 并发抢 20 件库存，最终成功 20 次、库存为 0；
- 能画出本次请求的完整缓存读写流程图。

## 2.6 Day 2 自测检验标准

- [ ] 不看表格，说出 5 种数据结构各自的 2 个典型场景
- [ ] 一句话区分穿透、击穿、雪崩，并各给出至少 2 种解法
- [ ] 为什么更新数据时推荐"删除缓存"而不是"更新缓存"？
- [ ] 分布式锁为什么要用 UUID 当 value？为什么要用 Lua 脚本释放？
- [ ] 主从架构下 Redis 锁为什么可能失效？什么业务不能用？
- [ ] 实操：20 分钟内手写一个带防穿透、防击穿逻辑的商品查询接口

---

# Day 3：数据库开发 + 综合实战项目

## 3.1 学习目标

1. 熟练使用 SQLAlchemy 2.0 完成模型定义、关联查询、批量操作；
2. 能看懂 `EXPLAIN` 执行计划，会设计索引并识别索引失效场景；
3. 理解事务隔离级别，能在代码中正确使用事务和连接池；
4. **完成综合实战项目：秒杀系统**，串联微服务 + Redis + 数据库全部知识点。

## 3.2 知识点清单与时间安排

| 时段 | 内容 | 耗时 |
|------|------|------|
| 上午 1 | 理论+实战：SQLAlchemy ORM | 1.5 h |
| 上午 2 | 理论：SQL 优化与索引设计 | 1.5 h |
| 上午 3 | 理论：事务与连接池 | 1 h |
| 下午 | 综合实战项目：秒杀系统 | 4 h |

## 3.3 理论讲解

### 知识点 1：ORM 框架实践（SQLAlchemy 2.0）

**理论要点**

- ORM 把表映射为类、行映射为对象，避免手写 SQL 字符串拼接（天然防注入）；代价是可能生成低效 SQL，需要保持警惕。
- 核心概念：`Engine`（连接管理）、`Session`（工作单元，管理对象生命周期与事务）、`declarative Base`（模型基类）。
- **N+1 问题（ORM 最经典的坑）**：查询 100 个订单后再逐个访问 `order.user`，会额外发出 100 条 SQL。解决：`selectinload/joinedload` 预加载关联。
- 最佳实践：Session 用完必须关闭（用上下文管理器）；批量插入用 `add_all` 或 `insert().values(list)`；只查需要的列，不要 `SELECT *`。

**典型应用场景**

- FastAPI 项目中用依赖注入 `Depends(get_db)` 给每个请求分配 Session；
- 管理后台列表页：主表 + 关联表用 `selectinload` 一次查全。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| ORM 和手写 SQL 怎么选？ | 日常 CRUD 用 ORM 提效；复杂报表、批量统计、性能敏感查询手写 SQL（ORM 也支持 `text()` 执行原生 SQL）。两者混用是常态。 |
| Session 能全局共享吗？ | 不能。Session 不是线程/协程安全的，必须每个请求独立创建、用完关闭。用 `sessionmaker` + 依赖注入。 |

### 知识点 2：SQL 查询优化与索引设计

**理论要点**

- **EXPLAIN 必看字段**：`type`（访问类型，从好到差：`const > eq_ref > ref > range > index > ALL`，出现 `ALL` 全表扫描要警惕）、`key`（实际用到的索引）、`rows`（扫描行数）、`Extra`（`Using filesort`/`Using temporary` 是危险信号，`Using index` 说明覆盖索引生效）。
- **B+Tree 索引原理（面试常考）**：数据存在叶子节点且有序链表相连，非叶子节点只存键值，树高通常 3 层就能存千万级数据——所以索引查找最多 3 次磁盘 IO。
- **索引设计三原则**：
  1. **最左前缀**：联合索引 `(a, b, c)` 能命中 `a`、`a,b`、`a,b,c` 的查询，但 `b,c` 单独查询不命中；
  2. **覆盖索引**：查询的列都在索引里，无需回表，Extra 显示 `Using index`；
  3. **高选择性优先**：区分度高的列（如订单号）适合索引，区分度低的列（如性别）不适合。
- **索引失效常见场景**：对索引列用函数/运算（`WHERE DATE(create_time)=...`）、隐式类型转换（字符串列用数字查）、`LIKE '%xx'` 前导通配符、`OR` 连接无索引列。
- 慢查询治理：开 `slow_query_log`（阈值如 1s）→ 定期分析 → 针对性加索引或改写 SQL。

**典型应用场景**

- 订单列表按 `user_id + create_time` 查并排序 → 建联合索引 `(user_id, create_time)`，一次索引搞定过滤和排序；
- 分页深翻页（`LIMIT 100000, 20`）→ 改用游标分页 `WHERE id > last_id LIMIT 20`。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 索引是不是越多越好？ | 不是。索引占磁盘，且每次写入都要维护索引树，写多读少的表要克制。一张表常规不超过 5~6 个索引。 |
| 为什么 `SELECT *` 是坏习惯？ | 多读无用列浪费 IO 和网络，且无法使用覆盖索引，强制回表。 |

### 知识点 3：事务与连接池管理

**理论要点**

- **ACID**：原子性（undo log）、隔离性（锁+MVCC）、持久性（redo log）、一致性（前三者共同保障）。
- **四种隔离级别与并发问题**：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 说明 |
|----------|------|------------|------|------|
| Read Uncommitted | 有 | 有 | 有 | 几乎不用 |
| Read Committed | 无 | 有 | 有 | Oracle 默认 |
| **Repeatable Read** | 无 | 无 | 基本无（InnoDB 用临键锁解决） | **MySQL InnoDB 默认** |
| Serializable | 无 | 无 | 无 | 性能差，慎用 |

- **乐观锁 vs 悲观锁**：乐观锁（版本号 `UPDATE ... SET stock=stock-1, version=version+1 WHERE id=? AND version=?`）适合冲突少的场景；悲观锁（`SELECT ... FOR UPDATE` 行锁）适合冲突激烈的场景，但要注意死锁和锁持有时间。
- **连接池**：数据库连接建立成本高（TCP + 认证），必须复用。SQLAlchemy 关键参数：`pool_size`（常驻连接数，默认 5）、`max_overflow`（临时超发，默认 10）、`pool_timeout`（拿不到连接等待多久）、`pool_recycle`（连接最大存活秒数，必须小于 MySQL 的 `wait_timeout` 默认 28800s，否则会拿到"已被服务端关闭"的连接报 `Lost connection`）。
- 池耗尽排查：高并发下请求卡在获取连接 → 检查是否有连接泄漏（Session 没关闭）、慢 SQL 占用连接过久、池配置过小。

**典型应用场景**

- 下单流程"扣库存 + 建订单"必须在一个事务里，任一步失败整体回滚；
- 扣库存高并发场景：用乐观锁 `UPDATE stock SET num=num-1 WHERE id=? AND num>0`，利用单行原子性避免超卖，比加锁吞吐高得多。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 事务里能调用外部接口吗？ | 绝对不要。事务持有期间不释放锁和连接，外部调用耗时不确定，会把连接池拖死。顺序应是：先调外部接口 → 再开短事务写库。 |
| 长事务有什么危害？ | 锁持有久、undo log 膨胀、主从延迟、连接占用。事务要短小精悍。 |

## 3.4 代码实战：SQLAlchemy 快速上手（45 分钟）

```python
# db_demo.py
from sqlalchemy import create_engine, String, Integer, ForeignKey, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, Session, selectinload

engine = create_engine(
    "mysql+pymysql://root:root123@127.0.0.1:3306/shop",
    pool_size=10, max_overflow=20, pool_recycle=3600,   # 连接池配置
)

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    orders: Mapped[list["Order"]] = relationship(back_populates="user")

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    item: Mapped[str] = mapped_column(String(100))
    user: Mapped[User] = relationship(back_populates="orders")

Base.metadata.create_all(engine)

# 事务：成功提交，异常回滚
with Session(engine) as session:
    try:
        user = User(name="张三")
        user.orders = [Order(item="键盘"), Order(item="鼠标")]
        session.add(user)
        session.commit()
    except Exception:
        session.rollback()
        raise

# 预加载解决 N+1：一条 SQL 查订单，一条 SQL 批量查用户
with Session(engine) as session:
    stmt = select(Order).options(selectinload(Order.user))
    for order in session.scalars(stmt):
        print(order.item, order.user.name)
```

**练习**：先去掉 `selectinload` 跑一遍，打开 `echo=True` 观察 SQL 日志中的 N+1；再加回来对比 SQL 条数。

## 3.5 综合实战项目：秒杀系统（4 小时）

这是三天学习的总装项目。目标：实现一个可压测的迷你秒杀系统，把微服务拆分、Redis 缓存/锁/预扣库存、数据库事务与索引全部用上。

### 系统架构

```
                        ┌─────────────┐
   客户端请求 ────────▶ │  API 网关    │  鉴权 + 限流(令牌桶) + 路由
                        └──────┬──────┘
              ┌────────────────┼─────────────────┐
              ▼                ▼                 ▼
      ┌──────────────┐ ┌──────────────┐  ┌──────────────┐
      │ user-service │ │product-service│  │ order-service │
      │   :8001      │ │   :8003       │  │   :8002      │
      └──────────────┘ └──────┬───────┘  └──────┬───────┘
                              │                 │
                              ▼                 ▼
                        ┌──────────┐      ┌──────────┐
                        │  Redis   │      │  MySQL   │
                        │ 缓存/库存 │      │ 订单落库  │
                        │ 锁/限流   │      │ 事务+索引 │
                        └──────────┘      └──────────┘
```

### 秒杀核心流程（order-service）

```
1. 网关层：校验 Token + 单用户限流（如 10 次/秒）
2. 校验活动状态与商品（商品信息走 Redis 缓存，Cache-Aside）
3. Redis 预扣库存：DECR seckill:stock:{product_id}
   - 返回值 < 0 → 已抢完，INCR 回补，直接返回"已售罄"（绝大多数请求在此被拦下）
4. 同一用户防重复下单：SET seckill:user:{uid}:{pid} 1 NX EX 3600
5. 生成订单号，把下单任务 LPUSH 到 Redis List 队列（削峰）
6. 立即返回"抢购中，请稍后查询结果"
7. 后台 worker 从队列 BRPOP 任务，在事务中：
   - UPDATE product SET stock=stock-1 WHERE id=? AND stock>0 （乐观锁兜底）
   - INSERT 订单
   - 失败则回滚并回补 Redis 库存
```

### 数据库表设计（含索引考点）

```sql
CREATE TABLE product (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0
);

CREATE TABLE seckill_order (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_no VARCHAR(32) NOT NULL,
    user_id INT NOT NULL,
    product_id INT NOT NULL,
    status TINYINT NOT NULL DEFAULT 0,
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_order_no (order_no),              -- 幂等：防重复插入
    UNIQUE KEY uk_user_product (user_id, product_id), -- 幂等：一人一单
    KEY idx_product_time (product_id, create_time)   -- 联合索引：按商品查订单
);
```

### 关键代码骨架

```python
# order-service 秒杀接口核心逻辑
import json
import uuid
import redis.asyncio as aioredis
from fastapi import FastAPI, HTTPException

app = FastAPI()
r = aioredis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

@app.post("/seckill/{product_id}")
async def seckill(product_id: int, user_id: int):
    # 1. 查商品（走缓存，含防穿透）
    product = await get_product_cached(product_id)
    if not product:
        raise HTTPException(status_code=404, detail="商品不存在")

    # 2. 防重复下单
    dup = await r.set(f"seckill:user:{user_id}:{product_id}", "1", nx=True, ex=3600)
    if not dup:
        raise HTTPException(status_code=400, detail="请勿重复下单")

    # 3. Redis 预扣库存
    stock = await r.decr(f"seckill:stock:{product_id}")
    if stock < 0:
        await r.incr(f"seckill:stock:{product_id}")    # 回补
        raise HTTPException(status_code=400, detail="已售罄")

    # 4. 任务入队，异步落库
    task = {"order_no": uuid.uuid4().hex[:24], "user_id": user_id, "product_id": product_id}
    await r.rpush("seckill:queue", json.dumps(task))
    return {"msg": "抢购中", "order_no": task["order_no"]}
```

### 实施里程碑（按顺序完成）

1. **M1（45 min）**：建表 + SQLAlchemy 模型 + 商品服务带缓存的查询接口；
2. **M2（60 min）**：秒杀接口：防重 + Redis 预扣库存 + 入队；
3. **M3（60 min）**：Worker 消费队列：事务落库（乐观锁扣库存 + 插订单），失败回补；
4. **M4（30 min）**：接入网关：鉴权 + 限流；
5. **M5（45 min）**：压测与调优。

### 压测验证（M5）

```python
# stress_test.py：200 并发抢 50 件库存
import asyncio
import httpx

async def one_request(client, uid):
    resp = await client.post(f"http://127.0.0.1:8000/seckill/1?user_id={uid}")
    return resp.status_code

async def main():
    async with httpx.AsyncClient() as client:
        tasks = [one_request(client, uid) for uid in range(200)]
        results = await asyncio.gather(*tasks)
    print("成功(200):", results.count(200))        # 预期恰好 50
    print("售罄/拦截:", len(results) - results.count(200))

asyncio.run(main())
```

**验收标准（综合项目达标线）**：

- [ ] 200 并发抢 50 件：成功人数恰好 50，Redis 与 MySQL 库存最终一致且为 0，**无超卖**；
- [ ] 同一用户重复请求返回"请勿重复下单"（幂等生效）；
- [ ] 商品查询走缓存，压测时 MySQL 的 product 表几乎无读压力；
- [ ] 订单表有合理的唯一索引与联合索引，能用 EXPLAIN 证明命中；
- [ ] 服务全部注册到注册中心，外部流量统一走网关；
- [ ] 能画出完整架构图并讲清每一层为什么存在。

## 3.6 Day 3 自测检验标准

- [ ] 什么是 N+1 问题？SQLAlchemy 里怎么解决？
- [ ] EXPLAIN 中出现哪两个值说明查询有问题？（type=ALL、Extra=filesort/temporary）
- [ ] 联合索引 `(a,b,c)` 能命中哪些查询条件组合？为什么不能命中 `WHERE b=?`？
- [ ] 写出 3 个索引失效的场景
- [ ] MySQL 默认隔离级别是什么？它解决了哪些并发问题？
- [ ] 为什么 `pool_recycle` 要小于 MySQL 的 `wait_timeout`？
- [ ] 秒杀系统里为什么用"Redis 预扣 + 队列异步落库"而不是直接操作数据库？
- [ ] 实操：独立讲清楚你的秒杀项目从用户点击到订单落库的完整链路

---

# Day 4：消息队列、Docker 部署与认证安全

## 4.1 学习目标

1. 理解消息队列的核心概念，能用 RabbitMQ 改造秒杀系统的异步下单链路；
2. 会写 Dockerfile，并用 docker-compose 把"网关 + 三个服务 + Redis + MySQL + RabbitMQ"一键编排启动；
3. 实现生产级认证：bcrypt 密码哈希 + JWT 签发与网关验签 + RBAC 接口级权限；
4. 能说出常见 Web 攻击的原理与防护手段。

## 4.2 知识点清单与时间安排

| 时段 | 内容 | 耗时 |
|------|------|------|
| 上午 1 | 理论：消息队列核心概念与可靠性 | 1.5 h |
| 上午 2 | 实战 1-2：RabbitMQ 收发 + 改造秒杀队列 | 1.5 h |
| 下午 1 | 理论+实战 3：Docker 与 docker-compose 编排 | 2.5 h |
| 下午 2 | 理论+实战 4：认证安全（bcrypt/JWT/RBAC） | 2 h |
| 晚间机动 | 练手项目：秒杀系统生产化升级 | 1.5 h |

## 4.3 理论讲解

### 知识点 1：消息队列（RabbitMQ）

**理论要点**

- **核心模型**：Producer 把消息发给 **Exchange**（交换机），Exchange 按 **Binding/Routing Key** 规则路由到一个或多个 **Queue**，Consumer 从 Queue 取消息消费。Exchange 常见类型：`direct`（精确匹配路由键）、`fanout`（广播到所有绑定队列）、`topic`（通配符匹配，如 `order.*`）。
- **三大价值**：解耦（上下游互不知晓）、削峰（洪峰先堆队列，按消费能力消化）、异步（主流程不等慢操作）。
- **可靠性三板斧（重点）**：
  1. **生产者确认（confirm）**：broker 收到消息后回执，防止"生产 → broker"环节丢失；
  2. **持久化**：队列声明 `durable=True` + 消息 `persistent`，防止 broker 重启丢消息；
  3. **消费者手动 ACK**：处理完业务才 ack，处理中宕机则消息重新投递，防止"消费"环节丢失。
- **死信队列（DLX）**：消费失败达到上限的消息转入死信队列，人工或异步兜底处理，避免坏消息反复阻塞主队列。
- 消费端必须**幂等**：消息可能重复投递（至少一次语义），用业务唯一键去重。

**典型应用场景**

- 秒杀下单：请求先入队立即返回，worker 慢慢落库（Day 3 项目用 Redis List 是简化版，今天换成真 MQ）；
- 注册后发邮件/短信：主流程只写入库 + 发消息，通知异步完成；
- 跨服务数据同步：订单服务发 `order.created` 事件，积分、统计服务各自订阅。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 消息在哪些环节会丢？ | 三个环节：生产者→broker（用 confirm）、broker 自身重启（持久化）、broker→消费者（手动 ACK）。三板斧全做才能基本不丢。 |
| 消息积压怎么办？ | 短期：加消费者实例扩容消费；长期：检查消费者性能瓶颈。临时救急可新建临时队列把消息分流。 |
| 怎么保证顺序消费？ | RabbitMQ 只保证单队列内有序。需要顺序的业务（如同一订单的状态流转）路由到同一队列、单消费者串行处理。 |
| RabbitMQ vs Kafka 怎么选？ | 业务消息（订单、通知，要求可靠投递、复杂路由）用 RabbitMQ；日志、埋点、大数据流（高吞吐、可回放）用 Kafka。 |

### 知识点 2：Docker 与 docker-compose

**理论要点**

- 三个核心概念：**镜像**（只读模板，分层的文件系统）、**容器**（镜像的运行实例）、**仓库**（Docker Hub 等）。
- Dockerfile 关键指令：`FROM`（基础镜像）、`COPY`（拷代码）、`RUN`（装依赖）、`EXPOSE`（声明端口）、`CMD`（启动命令）。利用**分层缓存**：把不常变的 `pip install` 放在前面，代码 COPY 放后面，改代码不必重装依赖。
- **docker-compose**：一个 YAML 描述整套系统的服务、端口、依赖顺序、网络和数据卷，`docker compose up` 一键起所有组件——开发环境"在我机器上能跑"问题的终极答案。
- **容器网络要点**：同一 compose 网络内，容器之间用**服务名**互相访问（如 `mysql:3306`），不能用 `localhost`（容器内 localhost 指容器自己）。
- 数据持久化用 **volume**：MySQL/Redis 的数据挂卷，否则容器删掉数据就没了。

**典型应用场景**

- 新同事入职：`git clone` + `docker compose up`，10 分钟跑起全套环境；
- 交付与部署：开发/测试/生产用同一镜像，杜绝环境差异；
- CI 流水线：每次提交自动构建镜像、跑测试。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| 镜像太大怎么瘦身？ | 用 `python:3.11-slim` 基础镜像、加 `.dockerignore`（排除 venv/.git）、多阶段构建。 |
| 容器里为什么不能跑多个进程？ | 容器以主进程（PID 1）为生命周期，主进程退出容器即停。多服务应拆成多容器，用 compose 编排。 |
| depends_on 能保证数据库就绪后再启动服务吗？ | 默认只保证启动顺序不保证"就绪"。需配合健康检查（`healthcheck` + `condition: service_healthy`）。 |

### 知识点 3：认证与安全（bcrypt / JWT / RBAC）

**理论要点**

- **密码存储**：绝不明文，也绝不用 MD5/SHA256（太快，彩虹表秒破）。用 **bcrypt**——自带随机盐、计算慢（故意慢，抗暴力破解），每次哈希结果不同但可校验。
- **JWT（JSON Web Token）**：结构是 `header.payload.signature` 三段。服务端登录后签发，客户端之后每次请求带在 `Authorization` 头里，网关验签即可，无需查库——天然契合微服务网关统一鉴权。注意三点：payload 只是 Base64 编码**不是加密**，不放敏感信息；过期时间设短（如 2h）+ refresh token 机制；必须走 HTTPS。
- **RBAC（基于角色的访问控制）**：用户 → 角色 → 权限三层。如"管理员能上下架商品，普通用户只能下单"。实现上：用户表加 `role` 字段，JWT payload 带角色，接口依赖注入校验。
- **常见攻击防护速查**：SQL 注入（用 ORM/参数化查询，永不拼字符串）；XSS（输出转义、CSP）；CSRF（SameSite Cookie、Token 校验）；**越权/IDOR**（最容易被忽视——接口必须校验"这个资源是不是当前用户的"，不能只校验登录态）；爆破（登录接口限流 + 验证码）。

**典型应用场景**

- 微服务体系：登录服务签发 JWT → 网关统一验签 → 业务服务从网关注入的用户信息中取身份，零重复鉴权代码；
- 管理后台：RBAC 控制菜单和接口级权限。

**常见问题解析**

| 问题 | 解析 |
|------|------|
| JWT 签发后怎么主动失效（如用户登出、封号）？ | JWT 无状态，默认只能等过期。方案：短过期 + refresh token；或把 jti（令牌 ID）记入 Redis 黑名单，网关验签时查黑名单。 |
| Session 和 JWT 怎么选？ | 单体应用、需要服务端随时踢人 → Session 更可控；微服务、多端、水平扩展 → JWT 更合适。 |
| 越权漏洞为什么常见？ | 框架只帮你做"登录校验"，"资源归属校验"永远要自己写。改 URL 里的 id 就能看别人数据，是 OWASP 常年榜首级漏洞。 |

## 4.4 代码实战

### 实战 1：RabbitMQ 可靠收发（45 分钟）

```python
# producer.py —— 生产者：持久化队列 + 持久化消息
import json
import pika

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()
ch.queue_declare(queue="seckill_tasks", durable=True)      # 队列持久化

task = {"order_no": "ORD-10001", "user_id": 1, "product_id": 1}
ch.basic_publish(
    exchange="",
    routing_key="seckill_tasks",
    body=json.dumps(task),
    properties=pika.BasicProperties(delivery_mode=2),       # 消息持久化
)
print("sent:", task)
conn.close()
```

```python
# consumer.py —— 消费者：手动 ACK，处理完才确认
import json
import pika

def handle(task: dict):
    print("处理下单任务:", task)   # 这里接 Day 3 的事务落库逻辑

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()
ch.queue_declare(queue="seckill_tasks", durable=True)
ch.basic_qos(prefetch_count=1)     # 一次只取一条，处理完再取，公平分发

def callback(ch, method, properties, body):
    try:
        handle(json.loads(body))
        ch.basic_ack(delivery_tag=method.delivery_tag)        # 成功才确认
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)  # 进死信

ch.basic_consume(queue="seckill_tasks", on_message_callback=callback)
ch.start_consuming()
```

**练习**：在消费回调里 `raise Exception`，配合给队列声明死信交换机（`x-dead-letter-exchange` 参数），观察失败消息流入死信队列。

### 实战 2：改造秒杀下单链路（45 分钟）

把 Day 3 秒杀接口中的 `r.rpush("seckill:queue", ...)` 替换为 RabbitMQ 投递，worker 从 Redis `BRPOP` 改为 `basic_consume`。流程不变：**预扣库存 → 投递 MQ → 立即返回；worker 消费 → 事务落库 → 手动 ACK**。体会：消息可靠了（worker 宕机消息不丢），且 `prefetch_count` 天然实现了消费限速削峰。

### 实战 3：容器化编排整套系统（75 分钟）

```dockerfile
# Dockerfile（网关和各服务通用，通过环境变量区分）
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt   # 分层缓存：依赖层不变就不重装
COPY . .
EXPOSE 8000
CMD ["sh", "-c", "uvicorn main:app --host 0.0.0.0 --port ${PORT}"]
```

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: shop
    volumes:
      - mysql_data:/var/lib/mysql          # 数据持久化
    healthcheck:                            # 就绪检查，供依赖等待
      test: ["CMD", "mysqladmin", "ping", "-proot123"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7

  rabbitmq:
    image: rabbitmq:3-management

  gateway:
    build: ./gateway
    environment:
      PORT: 8000
      REDIS_HOST: redis                    # 容器间用服务名互访
    ports:
      - "8000:8000"                        # 只有网关对外暴露
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
      order-service:
        condition: service_started

  order-service:
    build: ./order_service
    environment:
      PORT: 8002
      MYSQL_URL: mysql+pymysql://root:root123@mysql:3306/shop
      REDIS_HOST: redis
      RABBITMQ_HOST: rabbitmq
    depends_on:
      mysql:
        condition: service_healthy

  # user-service / product-service / worker 同理，省略

volumes:
  mysql_data:
```

**练习**：`docker compose up --build` 一键启动后，跑 Day 3 的压测脚本验证全链路；然后 `docker compose down` 再 up，确认 MySQL 数据还在（volume 生效）。

### 实战 4：JWT + RBAC 认证（60 分钟）

```python
# auth_demo.py —— 登录签发 + 网关验签 + 角色校验
import bcrypt
import jwt
from datetime import datetime, timedelta, timezone
from fastapi import FastAPI, Depends, HTTPException, Header

app = FastAPI()
SECRET = "change-me-in-production"   # 生产中放环境变量
USERS = {"zhangsan": {"pwd": bcrypt.hashpw(b"123456", bcrypt.gensalt()), "role": "admin"}}

@app.post("/login")
def login(username: str, password: str):
    user = USERS.get(username)
    if not user or not bcrypt.checkpw(password.encode(), user["pwd"]):
        raise HTTPException(status_code=401, detail="用户名或密码错误")
    payload = {
        "sub": username,
        "role": user["role"],
        "exp": datetime.now(timezone.utc) + timedelta(hours=2),
    }
    return {"token": jwt.encode(payload, SECRET, algorithm="HS256")}

# 网关/业务服务通用：验签依赖
def current_user(authorization: str = Header(default="")):
    token = authorization.removeprefix("Bearer ")
    try:
        return jwt.decode(token, SECRET, algorithms=["HS256"])
    except jwt.PyJWTError:
        raise HTTPException(status_code=401, detail="invalid token")

# RBAC：仅管理员可上下架商品
def require_admin(user=Depends(current_user)):
    if user["role"] != "admin":
        raise HTTPException(status_code=403, detail="forbidden")
    return user

@app.post("/admin/products")
def create_product(name: str, user=Depends(require_admin)):
    return {"msg": f"{user['sub']} 创建了商品 {name}"}
```

**练习**：用错误密钥伪造一个 token 请求 `/admin/products`，确认返回 401；把 `exp` 改成过去的时间，确认过期 token 被拒。

## 4.5 练手项目：秒杀系统生产化升级（1.5 小时）

在 Day 3 秒杀系统基础上完成三项升级：

1. **MQ 升级**：Redis List 队列 → RabbitMQ（手动 ACK + 死信队列兜底）；
2. **认证升级**：新增登录接口（bcrypt 验密、签发 JWT）→ 网关统一验签 → 秒杀接口从 JWT 取 `user_id`（不再信任请求参数，**防越权**）；商品管理接口仅 admin 角色可用；
3. **容器化**：所有组件写入 docker-compose，一条命令启动整套系统。

**验收标准**：

- `docker compose up --build` 后，完整跑通"登录 → 拿 token → 秒杀 → 订单落库"全链路；
- 伪造/过期 token 被网关 401，普通用户调管理接口被 403；
- 消费 worker 处理失败的消息进入死信队列，不丢不堵；
- 容器重建后历史订单数据仍在。

## 4.6 Day 4 自测检验标准

- [ ] 画出 Producer → Exchange → Queue → Consumer 模型，说出三种 Exchange 类型
- [ ] 消息丢失发生在哪三个环节？各自的对策是什么？
- [ ] 为什么消费者要手动 ACK？`prefetch_count` 起什么作用？
- [ ] 容器之间为什么不能通过 localhost 互访？数据如何持久化？
- [ ] 为什么密码必须用 bcrypt 而不能用 MD5/SHA256？
- [ ] JWT 的 payload 是加密的吗？如何实现"主动登出失效"？
- [ ] 什么是水平越权（IDOR）？代码里怎么防？
- [ ] 实操：不看示例，30 分钟内写出登录签发 JWT + 接口验签的最小闭环

---

## 结业验收标准

四天全部完成后，用以下清单做总验收：

| 能力 | 检验方式 |
|------|----------|
| 微服务拆分 | 给一个陌生业务（如外卖系统），15 分钟内口头拆出服务边界并说清依据 |
| 服务通信与治理 | 说出同步/异步的选择依据；说出超时、重试、幂等、限流各自解决什么问题 |
| Redis 数据结构 | 随机给 5 个业务场景，10 秒内各说出该用的数据结构 |
| 缓存设计 | 默写 Cache-Aside 流程；完整讲清穿透/击穿/雪崩的区别与对策 |
| 分布式锁 | 不看代码手写加锁/解锁核心逻辑，说出每一步防的是什么 |
| SQL 与索引 | 给一条慢 SQL，能用 EXPLAIN 定位问题并给出索引方案 |
| 事务与连接池 | 说清隔离级别、长事务危害、连接池关键参数 |
| 消息队列 | 说清消息丢失的三个环节与对策；说明死信队列和消费幂等的作用 |
| 容器化交付 | 不看示例写出一个服务的 Dockerfile，并用 compose 编排多组件一键启动 |
| 认证与安全 | 手写登录签发 JWT + 验签闭环；说出 bcrypt、RBAC、越权防护的要点 |
| 综合实战 | 秒杀项目通过 Day 3 + Day 4 全部验收标准，且能向别人完整讲解架构 |

**后续进阶方向**：分布式事务（TCC/Saga/本地消息表）、可观测性（Prometheus + Grafana 监控、SkyWalking 链路追踪）、测试体系（pytest + 接口自动化）、Elasticsearch、Kafka 与流处理、Kubernetes 云原生部署、领域驱动设计深入。

祝学习顺利，四天后见。
