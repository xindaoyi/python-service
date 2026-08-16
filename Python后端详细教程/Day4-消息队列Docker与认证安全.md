# Day 4：消息队列、Docker 部署与认证安全 · 详细教程

> 今日目标：用 RabbitMQ 换掉秒杀系统的"简配队列"；用 Docker 把整套系统打包成一键启动；给系统装上生产级登录认证与权限控制。
> 时间：约 8 小时（MQ 3h + Docker 2.5h + 安全 2h + 项目穿插其中）

---

## 1. 消息队列：从"简配"到"专业"（1.5 小时理论）

### 1.1 Day 3 留下的隐患

秒杀项目里我们用 Redis List 当队列：`RPUSH` 入队、`BLPOP` 出队。能用，但有两个隐患：

1. **消息会丢**：worker 用 `BLPOP` 取出消息的瞬间宕机——消息已经从队列里弹出来了，但还没落库。这条订单任务就永远消失了（用户抢了单却没订单）。
2. **没有确认与重试机制**：消息处理失败了，只能靠我们自己写代码回补库存、删防重标记，逻辑散落在业务里，容易漏。

**专业消息队列（MQ）就是为"消息绝不能丢"而生的中间件。** 国内业务系统最常用 RabbitMQ，今天用它。

### 1.2 RabbitMQ 的核心模型（一张图记住）

```
Producer ──▶ Exchange（交换机）──▶ Queue（队列）──▶ Consumer（消费者）
 生产者        按规则分发            存消息的         取消息处理
```

五个角色，逐个说清：

- **Producer（生产者）**：发消息的一方，比如秒杀接口。
- **Exchange（交换机）**：消息的"分拣中心"。生产者不直接发给队列，而是发给交换机，交换机按规则决定送去哪个（些）队列。三种常用类型：
  - `direct`：按 routing key **精确匹配**（key 为 "order" 的消息只进绑定了 "order" 的队列）；
  - `fanout`：**广播**，所有绑定的队列都收到一份（比如"订单创建"事件，短信、积分、统计服务各收一份）；
  - `topic`：按**通配符**匹配（`order.*` 能匹配 `order.created`、`order.paid`）。
- **Binding（绑定）**：交换机和队列之间的连线规则。
- **Queue（队列）**：真正存消息的地方，先进先出。
- **Consumer（消费者）**：从队列取消息处理，比如我们的落库 worker。

**前端类比**：整个模型就是你熟悉的 EventEmitter 的"分布式可靠版"——Exchange 是事件总线，routing key 是事件名，Queue 是每个订阅者各自的信箱。区别在于：EventEmitter 里没监听者的事件直接蒸发，MQ 里消息会**持久化躺在队列里**，等消费者上线慢慢取。

### 1.3 可靠性三板斧（MQ 的立身之本，必考）

消息从生产到消费要经过三段路，每段都可能丢，各有一件防护武器：

```
生产者 ──①──▶ broker(RabbitMQ) ──②──▶ 消费者处理
       confirm机制              手动ACK
              ①和②之间 broker 重启怎么办？→ 持久化
```

**第一板斧：生产者确认（confirm）**
生产者发完消息后，broker 回一个"我收到了"的回执。没收到回执就重发。防的是"网络抖动，消息在路上丢了"。

**第二板斧：持久化（durable + persistent）**
队列声明 `durable=True` + 消息标记 `persistent`：消息不仅存内存，还写磁盘。broker 重启后消息还在。防的是"broker 宕机/重启"。

**第三板斧：消费者手动 ACK（最重要）**
这是解决 Day 3 隐患的关键。Redis List 的 BLPOP 是"取出即删除"；RabbitMQ 默认模式是：

```
worker 取到消息 → 处理业务（落库）→ 处理成功，手动发 ACK → broker 才删除消息
                                  ↘ 处理失败/宕机，没发 ACK → broker 把消息重新投递给别人
```

**"处理完才确认"这一个机制，就保证了消息至少被处理一次（at-least-once）**——代价是可能重复投递（broker 没收到 ACK 可能是处理完了但 ACK 丢了），所以**消费端必须幂等**。回头看 Day 3 的两道唯一索引（`uk_order_no`、`uk_user_product`）——幂等设计我们早就铺好路了。

### 1.4 死信队列：坏消息的收容所

如果一条消息的处理代码有 bug，消费一次失败一次，无限重投会变成"毒消息"卡死队列。解法是给队列配一个**死信队列（DLX）**：被拒绝/过期/重试超限的消息自动转入死信队列，主队列继续健康运转，坏消息攒在那里由人工或专门程序兜底。

### 1.5 常见面试题速答

| 问题 | 答案 |
|---|---|
| 消息积压了怎么办 | 短期加消费者实例；长期查消费者性能瓶颈；应急可分流到临时队列 |
| 怎么保证顺序 | 只保证单队列内有序：相关消息路由到同一队列 + 单消费者串行消费 |
| 重复消费怎么办 | 消费端幂等（业务唯一键去重），没有别的银弹 |
| RabbitMQ vs Kafka | 业务消息（要可靠、要复杂路由）选 RabbitMQ；日志/埋点/大数据流（要高吞吐）选 Kafka |

---

## 2. RabbitMQ 实战：改造秒杀链路（1.5 小时）

确认 rabbitmq 容器在跑，浏览器开 `http://127.0.0.1:15672`（guest/guest）——这是管理后台，等会儿在这里亲眼看到消息流转。

### 2.1 生产者

```python
# mq_producer.py
import json
import pika

# 建立连接（真实项目里要封装成带重连的连接管理器）
conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()

# 声明队列：durable=True 表示队列持久化（broker 重启队列还在）
ch.queue_declare(queue="seckill_tasks", durable=True)

task = {"order_no": "ORD-10001", "user_id": 1, "product_id": 1}
ch.basic_publish(
    exchange="",                          # 空交换机 = 默认直发同名队列
    routing_key="seckill_tasks",
    body=json.dumps(task),
    properties=pika.BasicProperties(delivery_mode=2),   # 2 = 消息持久化
)
print("已投递:", task)
conn.close()
```

运行后到管理后台 Queues 标签页：`seckill_tasks` 队列里 Ready 消息数 +1。**消息已经在 broker 里躺着了，即使现在关掉所有 Python 程序它也不会丢。**

### 2.2 消费者（手动 ACK）

```python
# mq_consumer.py
import json
import pika

def handle(task: dict):
    """这里替换成 Day 3 worker 的事务落库逻辑"""
    print("处理下单任务:", task)

conn = pika.BlockingConnection(pika.ConnectionParameters("127.0.0.1"))
ch = conn.channel()
ch.queue_declare(queue="seckill_tasks", durable=True)
ch.basic_qos(prefetch_count=1)   # 一次只发 1 条给我，处理完 ACK 后再发下一条——公平分发+限速

def callback(ch, method, properties, body):
    try:
        handle(json.loads(body))
        ch.basic_ack(delivery_tag=method.delivery_tag)          # 处理成功，确认删除
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)  # 拒收且不再入队（配 DLX 则进死信）

ch.basic_consume(queue="seckill_tasks", on_message_callback=callback)
print("等待任务中...")
ch.start_consuming()
```

**三个必做实验（每个都直中一个概念）**：
1. **先跑 producer 再跑 consumer**：producer 投递 3 条后退出，consumer 启动后依然逐条收到——消息持久化生效；
2. **ACK 实验**：consumer 里在 `handle` 之前加一行 `1/0`，重新投递的消息处理报错被 nack；再改成处理中直接 Ctrl+C 杀掉 consumer——管理后台里消息从 Unacked 回到 Ready，**证明"没 ACK 就重投"**；
3. **prefetch 实验**：开两个 consumer，prefetch_count=1 时任务被轮流分发；注释掉 basic_qos，一个 consumer 会抢走全部任务。

### 2.3 升级秒杀项目

改造点只有两处：
- order-service 的秒杀接口：`r.rpush(...)` → 上面的 `basic_publish(...)`；
- worker：`r.blpop(...)` → `basic_consume` + 手动 ACK，落库失败时 `basic_nack`。

改完重新压测（Day 3 的 stress_test.py），结果应保持一致：50 人抢到、无超卖——但现在的队列**专业了**：worker 宕机消息不丢、处理失败自动重投。

---

## 3. Docker：把整个系统装进集装箱（2.5 小时）

### 3.1 解决什么问题："在我机器上明明能跑"

后端协作的经典灾难：你电脑上 Python 3.11 + MySQL 8 跑得好好的，同事是 Python 3.9，服务器是 CentOS 自带老版本——同一份代码三种环境三种错误。

**Docker 的思路：把代码 + Python 运行时 + 依赖 + 系统环境，整个打包成一个"镜像"。任何装了 Docker 的机器运行这个镜像，得到的都是一模一样的东西。**

**前端类比**：构建镜像 ≈ `npm run build` 打出可部署产物，只不过 Docker 打包的不只是静态文件，而是**包含操作系统层的完整运行环境**；运行容器 ≈ 把这个产物部署起来跑一个实例。

三个核心概念：

| 概念 | 定义 | 类比 |
|---|---|---|
| 镜像（Image） | 只读的环境+代码模板，分层的 | npm 包 + node_modules 的快照 |
| 容器（Container） | 镜像跑起来的一个实例 | 类的一个实例对象 |
| 仓库（Registry） | 存放镜像的地方（Docker Hub） | npm registry |

> 和虚拟机的区别：虚拟机虚拟整套硬件+操作系统（GB 级、分钟级启动）；容器共享宿主机内核，只打包应用和依赖（MB~几百 MB、秒级启动）。轻量所以才能成为微服务部署的标准单位。

### 3.2 镜像分层：一条必会的优化知识

镜像由一层层只读层叠成，每条 Dockerfile 指令产生一层，**层有缓存**：构建时如果某层的输入没变，直接复用缓存。所以写 Dockerfile 的黄金法则是——**把不常变的操作放前面，常变的放后面**：

```dockerfile
# Dockerfile（放在每个服务的目录里）
FROM python:3.11-slim            # 基础镜像：slim 版去掉无用组件，镜像小
WORKDIR /app                     # 容器内的工作目录

COPY requirements.txt .          # 先只拷依赖清单
RUN pip install --no-cache-dir -r requirements.txt
# ↑ 只要 requirements.txt 没变，下次构建直接命中缓存，几秒完成

COPY . .                         # 再拷代码（代码天天改，只让这一层失效重建）

EXPOSE 8000
CMD ["sh", "-c", "uvicorn main:app --host 0.0.0.0 --port ${PORT}"]
# ↑ 容器启动时执行的命令；--host 0.0.0.0 必须写，否则容器外访问不到
```

如果顺序反过来（先 COPY 代码再 pip install），你每改一行代码，构建时都要重装十分钟依赖——分层缓存的意义就在这。

单个服务试一下：

```bash
cd order_service
docker build -t order-service:v1 .     # 构建镜像，-t 起名字
docker run -p 8002:8000 -e PORT=8000 order-service:v1   # 跑起来：宿主机8002 → 容器8000
```

### 3.3 docker-compose：一键启动全世界

我们的系统有 7 个组件（网关、3 个服务、worker、MySQL、Redis、RabbitMQ），一个个 `docker run` 会疯掉。**compose 用一个 YAML 文件描述所有组件及其关系，`docker compose up` 一条命令全部启动**：

```yaml
# docker-compose.yml（放在项目根目录）
services:
  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: shop
    volumes:
      - mysql_data:/var/lib/mysql        # 数据挂卷：容器删了数据还在
    healthcheck:                          # 健康检查：证明"MySQL 真的能连了"
      test: ["CMD", "mysqladmin", "ping", "-proot123"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7

  rabbitmq:
    image: rabbitmq:3-management

  order-service:
    build: ./order_service               # 用该目录的 Dockerfile 现场构建
    environment:
      PORT: 8002
      MYSQL_URL: mysql+pymysql://root:root123@mysql:3306/shop
      #                                        ^^^^ 注意：不是 localhost，是服务名！
      REDIS_HOST: redis
      RABBITMQ_HOST: rabbitmq
    depends_on:
      mysql:
        condition: service_healthy       # 等 MySQL 通过健康检查再启动我

  gateway:
    build: ./gateway
    environment:
      PORT: 8000
      ORDER_SERVICE_URL: http://order-service:8002
    ports:
      - "8000:8000"                      # 只有网关把端口暴露给宿主机
    depends_on:
      order-service:
        condition: service_started

  # user-service / product-service / worker 写法同理，略

volumes:
  mysql_data:
```

**两个最容易踩的坑，务必理解**：

1. **容器之间用"服务名"互访，不能用 localhost**。每个容器是一个隔离的小世界，容器里的 `localhost` 指容器自己。compose 会给同网络的容器自动配置服务名解析——所以连接串里写 `mysql:3306`、`redis:6379`。这也是为什么配置要用环境变量注入（本地开发用 localhost，容器里用服务名）。
2. **depends_on 只管启动顺序，不管"就绪"**。MySQL 进程启动了不代表它能接受连接（初始化要好几秒）。所以要配 `healthcheck` + `condition: service_healthy`，让依赖方等到 MySQL 真正就绪。

**启动与验证**：

```bash
docker compose up --build -d      # 构建并后台启动全部组件
docker compose ps                 # 查看各组件状态
docker compose logs -f gateway    # 跟踪网关日志
curl http://127.0.0.1:8000/users/1 -H "Authorization: token-zhangsan"   # 全链路通！
docker compose down               # 一键全部停止
docker compose up -d              # 再启动——查数据库，数据还在（volume 生效）
```

---

## 4. 认证与安全（2 小时）

### 4.1 密码存储：为什么必须是 bcrypt

假设你的用户表被黑客拖库了（这在业界年年发生）。如果密码是明文存的——所有用户账号立刻沦陷，而且用户在别的网站也用同一个密码（撞库）。

**方案：不存密码，存密码的哈希**。登录时把用户输入的密码再算一次哈希，比对是否一致。但选哈希算法有讲究：

| 方案 | 问题 |
|---|---|
| MD5 / SHA256 | **太快了**——现代 GPU 每秒能算几十亿次，黑客离线暴力穷举"常见密码字典"分分钟破解；而且不加盐的话，相同密码哈希相同，彩虹表直接反查 |
| **bcrypt** | **故意慢**（单次哈希约 100ms，用户可以等，黑客穷举成本暴涨一亿倍）+ **自带随机盐**（同一密码每次哈希结果都不同，彩虹表作废） |

```python
import bcrypt

hashed = bcrypt.hashpw(b"123456", bcrypt.gensalt())   # 注册时：生成哈希（每次结果都不同）
bcrypt.checkpw(b"123456", hashed)   # 登录时：校验 → True
bcrypt.checkpw(b"wrong", hashed)    # → False
```

### 4.2 JWT：微服务时代的"通行证"

你在前端一定做过：登录成功后端返回一个 token，之后每次请求塞进 `Authorization` 头。这个 token 的主流格式就是 **JWT**。

**为什么微服务特别需要它？** 传统的 session 方案，登录态存在服务器内存/Redis 里，每次请求都要查一次"这个 session 是谁"——微服务架构下每个服务都去查，又慢又耦合。JWT 把用户信息**直接编码进 token 并签名**，任何服务拿到 token 自己验签就知道你是谁，不用查库。

**结构**：`header.payload.signature` 三段，用 `.` 连接：

```
header:   {"alg":"HS256"}                     → 用什么算法签名
payload:  {"sub":"zhangsan","role":"admin","exp":1755000000}  → 你是谁、什么角色、何时过期
signature: 用密钥对前两段算出的签名 → 防篡改
```

**关键认知（新手最容易错的地方）**：payload 只是 Base64 **编码**，任何人都能解码看内容——**它不是加密！** 所以 JWT 里绝不放密码、身份证等敏感信息。它的安全性来自签名：没有密钥的人改了 payload 任何一个字符，验签就失败——**可读取，但不可伪造**。

```python
import jwt
from datetime import datetime, timedelta, timezone

SECRET = "change-me-in-production"     # 生产中放环境变量，泄露 = 任何人可伪造身份

token = jwt.encode(
    {"sub": "zhangsan", "role": "admin",
     "exp": datetime.now(timezone.utc) + timedelta(hours=2)},
    SECRET, algorithm="HS256",
)

payload = jwt.decode(token, SECRET, algorithms=["HS256"])   # 验签+解析
print(payload)   # {'sub': 'zhangsan', 'role': 'admin', 'exp': ...}

jwt.decode(token, "wrong-secret", algorithms=["HS256"])     # 密钥不对 → 抛异常
```

**JWT 的短板与对策**：签发后服务端无法主动作废（无状态的代价），只能等它过期。对策：过期时间设短（2h）+ refresh token 续期；或者把 token 的 jti 记入 Redis 黑名单，登出/封号时拉黑，验签时多查一步（牺牲了部分无状态性，换取可控性）。

### 4.3 RBAC：角色权限模型

认证（你是谁）之后是授权（你能干什么）。业界标准是 **RBAC（基于角色的访问控制）**：**用户 → 角色 → 权限**三层。

```
张三 ──▶ admin角色 ──▶ 权限：上下架商品、查看所有订单
李四 ──▶ user角色  ──▶ 权限：下单、查看自己的订单
```

实现上可以很简单：用户表加 `role` 字段，签发 JWT 时把 role 放进 payload，敏感接口校验角色：

```python
from fastapi import Depends, HTTPException, Header

def current_user(authorization: str = Header(default="")):
    """从 Authorization 头解析并验签 JWT，得到当前用户——每个需要登录的接口都依赖它"""
    token = authorization.removeprefix("Bearer ")
    try:
        return jwt.decode(token, SECRET, algorithms=["HS256"])
    except jwt.PyJWTError:
        raise HTTPException(status_code=401, detail="invalid token")

def require_admin(user=Depends(current_user)):
    """在 current_user 之上再加角色校验——管理接口依赖它"""
    if user["role"] != "admin":
        raise HTTPException(status_code=403, detail="forbidden")
    return user

# 普通接口：def seckill(user=Depends(current_user))     → 登录即可
# 管理接口：def add_product(user=Depends(require_admin)) → 仅管理员
```

### 4.4 四大常见攻击与防护（安全素养底线）

1. **SQL 注入**：`"WHERE name = '" + user_input + "'"` 拼接 SQL，用户输入 `' OR '1'='1` 就能绕过条件甚至删库。**防护：永远用 ORM 或参数化查询，绝不拼字符串。**
2. **XSS（跨站脚本）**：用户提交的内容里藏 `<script>`，页面渲染时在其他用户浏览器执行。**防护：输出时转义、设置 CSP。**
3. **CSRF（跨站请求伪造）**：利用浏览器自动带 Cookie 的特性，诱导已登录用户点恶意链接发起非本意请求。**防护：SameSite Cookie、CSRF Token。用 JWT + Authorization 头的方案天然免疫（浏览器不会自动带自定义头）。**
4. **越权 / IDOR（最常被忽视）**：接口只校验了"登录了没有"，没校验"这条数据是不是你的"——把 URL 里的 `/orders/1001` 改成 `/orders/1002` 就看到别人的订单。**OWASP 常年榜首级漏洞。防护：每个涉及资源的接口，必须校验资源归属**：

```python
@app.get("/orders/{order_id}")
def get_order(order_id: int, user=Depends(current_user)):
    order = find_order(order_id)
    if order.user_id != user["sub"]:      # 关键一行：资源必须属于当前用户
        raise HTTPException(status_code=403, detail="not your order")
    return order
```

---

## 5. 练手项目：秒杀系统生产化升级（穿插在今天的实战中完成）

三项升级，今天学完一块就改一块：

**升级 1（MQ）**：Redis List → RabbitMQ（持久化 + 手动 ACK + 死信队列）。验收：压测结果与之前一致；杀掉 worker 后重启，未处理的消息自动被重新消费。

**升级 2（认证）**：
- user-service 增加 `POST /register`（bcrypt 存密码）和 `POST /login`（验密、签发 JWT）；
- 网关把"查静态 token 表"换成 JWT 验签，解析出的用户信息注入请求头传给下游；
- 秒杀接口的 `user_id` **从 JWT 里取，不再信任请求参数**（为什么？——请求参数用户可以随便填，那就是越权漏洞）；
- 商品管理接口加 `require_admin`。

**升级 3（容器化）**：所有组件写入 docker-compose，一条命令启动。

**总验收**：
```bash
docker compose up --build -d
# 1. 注册 → 登录拿 token → 带 token 秒杀 → worker 落库，全链路通
# 2. 伪造 token（改一个字符）→ 401；普通用户调 /admin/products → 403
# 3. 秒杀接口传 user_id=别人 → 无效，实际下单人是 token 里的自己
# 4. docker compose down && up → 历史订单数据仍在
```

---

## 6. Day 4 自测清单

- [ ] 画出 RabbitMQ 的 Producer→Exchange→Queue→Consumer 模型；三种 Exchange 类型各适合什么场景？
- [ ] 消息丢失发生在哪三个环节？三板斧各防哪一段？
- [ ] 手动 ACK 为什么能保证"消息至少被处理一次"？它带来的副作用是什么，如何化解？（重复投递 → 消费幂等）
- [ ] 死信队列解决什么问题？
- [ ] Docker 镜像分层缓存的黄金法则是什么？为什么 pip install 要放在 COPY 代码之前？
- [ ] 容器之间为什么不能用 localhost 互访？数据怎么持久化？depends_on 为什么要配 healthcheck？
- [ ] 为什么密码必须用 bcrypt？它"慢"和"加盐"各解决什么问题？
- [ ] JWT 的 payload 是加密的吗？防篡改靠什么？怎么实现"主动登出"？
- [ ] 什么是越权（IDOR）？秒杀接口为什么不能从请求参数取 user_id？
- [ ] 实操：不看代码，30 分钟写出"注册（bcrypt）→ 登录（签发 JWT）→ 受保护接口（验签 + 角色校验）"最小闭环

---

## 结业寄语

四天前你只会 Python 语法和前端；现在你已经亲手搭出了一个**有网关、有注册中心、有缓存防线、有消息队列、有事务保障、有认证权限、可以一条命令容器化部署**的分布式秒杀系统，并且理解每一层为什么存在。

这套"为什么这么设计"的直觉，比任何具体框架都值钱——框架三年一换，但缓存为什么防穿透、消息为什么要 ACK、事务为什么要短，这些问题十年后依然如此。

继续前行的路标：pytest 测试体系 → Prometheus/Grafana 可观测性 → Kafka 与流处理 → Kubernetes 编排 → 分布式事务（TCC/Saga）。江湖再见。
