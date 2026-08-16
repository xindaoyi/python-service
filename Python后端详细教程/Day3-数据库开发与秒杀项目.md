# Day 3：数据库开发与秒杀综合项目 · 详细教程

> 今日目标：从零学会 SQL 基础；熟练使用 SQLAlchemy；看懂执行计划、会设计索引；理解事务与连接池；下午完成综合项目——秒杀系统，把微服务、Redis、数据库全部焊在一起。
> 时间：约 8 小时（SQL 速成 1h + 理论实战 3h + 综合项目 4h）

---

## 1. 数据库与 SQL 速成（1 小时，零基础友好）

### 1.1 关系型数据库是什么

你已经用过两种存数据的方式：内存字典（Day 1）和 Redis（Day 2）。它们都有硬伤：内存字典重启即丢、无法共享；Redis 快但内存贵、查询能力弱（只能按 key 查）。

**MySQL 是数据的"总账本"**：数据持久化在磁盘、支持复杂查询、支持多表关联、支持事务保证正确性。代价是比 Redis 慢得多——所以架构里它们分工明确：**Redis 扛读流量，MySQL 记最终账**。

核心概念用一张表说清（类比 Excel）：

| 数据库概念 | 类比 | 说明 |
|---|---|---|
| 数据库（database） | 一个 Excel 文件 | 一个应用通常用一个库 |
| 表（table） | 一个 sheet | 存一类事物：users 表存用户 |
| 行（row/record） | 一行数据 | 一个具体用户 |
| 列（column/field） | 一列 | 用户的某个属性：name、age |
| 主键（primary key） | 工号列 | 每行的唯一标识，不能重复 |
| 索引（index） | 书的目录 | 加速按某列查找 |

### 1.2 SQL 上手：五种必会操作

连接数据库练习（MySQL 容器已在跑）：

```bash
docker exec -it mysql mysql -uroot -proot123 shop
```

**建表**（定义"用户"长什么样）：

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,   -- 主键，自动递增
    name VARCHAR(50) NOT NULL,           -- 字符串，最长 50，不能为空
    level VARCHAR(20) DEFAULT 'normal',  -- 默认值
    balance DECIMAL(10,2) DEFAULT 0.00,  -- 金额，共 10 位其中 2 位小数
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**增 INSERT**：

```sql
INSERT INTO users (name, level, balance) VALUES ('张三', 'vip', 1000.00);
INSERT INTO users (name) VALUES ('李四');
```

**查 SELECT**（用得最多）：

```sql
SELECT * FROM users;                                  -- 查全部（* = 所有列，生产环境慎用）
SELECT name, balance FROM users WHERE level = 'vip';  -- 条件查询，只取需要的列
SELECT * FROM users WHERE balance > 100 ORDER BY balance DESC LIMIT 10;  -- 排序+取前10
SELECT COUNT(*) FROM users WHERE level = 'vip';       -- 统计行数
```

**改 UPDATE / 删 DELETE**（⚠️ 一定带 WHERE）：

```sql
UPDATE users SET balance = balance - 100 WHERE id = 1;   -- 张三扣 100 元
DELETE FROM users WHERE id = 2;
-- 忘掉 WHERE 就是全表更新/全表删除，生产事故经典剧本
```

**多表关联 JOIN**（关系型数据库的招牌能力）：

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,                -- 外键：指向 users.id，记录"谁下的单"
    item VARCHAR(100),
    amount DECIMAL(10,2),
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user (user_id)             -- 给 user_id 建索引，马上会讲为什么
);

-- 查"每个订单连同下单人的名字"：把两张表按 user_id 拼起来
SELECT o.id, o.item, u.name
FROM orders o
JOIN users u ON o.user_id = u.id;
```

**前端类比**：JOIN 就像你在前端把两个数组按 id 关联合并（`orders.map(o => ({...o, user: users.find(u => u.id === o.user_id)}))`）——只不过数据库帮你做了，而且百万行也是毫秒级（前提是索引建对了）。

### 1.3 练习

1. 插入 3 个用户、5 条订单；
2. 查出"张三的所有订单按金额降序"；
3. 用 JOIN 查出每条订单的用户名；
4. 把 id=1 的用户余额加 200，再查出来确认。

---

## 2. SQLAlchemy：用 Python 对象操作数据库（1 小时）

### 2.1 为什么需要 ORM

直接在 Python 里拼 SQL 字符串，有三个问题：字符串拼接容易出 **SQL 注入**漏洞（后面会讲）；表结构变了要满世界改字符串；查出来的原始元组要自己手动组装成对象。

**ORM（对象关系映射）把"表"映射成"类"、把"行"映射成"对象"**：你操作 Python 对象，ORM 帮你生成并执行安全的 SQL。SQLAlchemy 是 Python 的事实标准 ORM。

```python
# db_demo.py —— 完整可跑
from sqlalchemy import create_engine, String, ForeignKey, select
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, Session, selectinload

# ① Engine：数据库连接的管理者（内含连接池，下午细讲）
engine = create_engine(
    "mysql+pymysql://root:root123@127.0.0.1:3306/shop",
    pool_size=10, max_overflow=20, pool_recycle=3600,
    echo=False,        # 改成 True 会打印每条执行的 SQL，学习时建议打开观察
)

# ② 模型：类 ↔ 表
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    orders: Mapped[list["Order"]] = relationship(back_populates="user")  # 一对多关系

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    item: Mapped[str] = mapped_column(String(100))
    user: Mapped[User] = relationship(back_populates="orders")

Base.metadata.create_all(engine)   # ③ 根据模型自动建表（已存在则跳过）

# ④ Session：一次"工作单元"，管一批操作和事务
with Session(engine) as session:
    user = User(name="张三")
    user.orders = [Order(item="键盘"), Order(item="鼠标")]
    session.add(user)
    session.commit()               # 提交事务： INSERT 用户 + 两条订单一起生效

# ⑤ 查询：select 语法接近 SQL，但返回的是对象
with Session(engine) as session:
    stmt = select(User).where(User.name == "张三")
    zhangsan = session.scalars(stmt).first()
    print(zhangsan.name, [o.item for o in zhangsan.orders])
```

三个关键角色的分工，记住这个类比：**Engine 是车队（连接池），Session 是派出去办事的一次任务（事务），模型类是货物清单（表结构）。**

### 2.2 N+1 问题：ORM 最经典的坑

把上面 `echo=True` 打开，然后跑这段代码：

```python
with Session(engine) as session:
    orders = session.scalars(select(Order)).all()     # 查所有订单（1 条 SQL）
    for o in orders:
        print(o.item, o.user.name)                    # 访问关联对象
```

观察日志：除了第 1 条查订单的 SQL，**每个订单又额外发出 1 条查用户的 SQL**——100 个订单就是 1 + 100 = 101 条 SQL。这就是 N+1 问题：ORM 在你访问 `o.user` 时"贴心"地替你现查，结果把一次查询变成了循环查询，性能雪崩。

**解决：预加载（eager loading）**——告诉 ORM"我待会儿要用到关联对象，请一次性查好"：

```python
stmt = select(Order).options(selectinload(Order.user))   # 第 2 条 SQL 用 IN 批量查用户
for o in session.scalars(stmt):
    print(o.item, o.user.name)     # 总共只有 2 条 SQL，无论多少订单
```

> **职业习惯**：用 ORM 的项目，把 `echo=True` 或慢 SQL 日志开着做开发，看到循环里冒出 N 条相似 SQL 立刻警觉。

### 2.3 Session 使用规范

- **每个请求一个 Session，用完就关**（Session 不是线程/协程安全的，绝不能做成全局单例）。在 FastAPI 里用依赖注入：

```python
from sqlalchemy.orm import sessionmaker

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# 接口里：def list_users(db: Session = Depends(get_db)): ...
```

---

## 3. 索引与 SQL 优化（1 小时）

### 3.1 索引是什么：书的目录

没有索引时，`SELECT * FROM users WHERE name = '张三'` 只能**全表扫描**——一行一行翻，100 万行就翻 100 万次。这就像在一本没有目录的书里找某个词。

索引就是给列建的"目录"：按 name 排序好并配上快速定位结构，找"张三"只需要几次跳转。**MySQL InnoDB 的索引结构是 B+Tree**，你只需要记住它的三个特性：

1. 树很矮：3 层的 B+Tree 能存上千万条记录——**任何索引查找最多 3 次磁盘读取**；
2. 叶子节点有序且相连：所以范围查询（`BETWEEN`、`>`）和排序也很快；
3. 索引本身要占空间、写入时要维护——**索引不是免费的，乱建索引会拖慢写入**。

### 3.2 用 EXPLAIN 给 SQL 做体检

在 SQL 前加 `EXPLAIN`，MySQL 会告诉你这条查询打算怎么执行：

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1;
```

重点看四个字段：

| 字段 | 看什么 | 健康标准 |
|---|---|---|
| `type` | 访问方式 | `const/ref/range` 都好；**`ALL` = 全表扫描，危险** |
| `key` | 实际用了哪个索引 | 不为 NULL 才算用上索引 |
| `rows` | 预计扫描多少行 | 越小越好，几十万就要警惕 |
| `Extra` | 附加信息 | `Using index`（覆盖索引，很好）；**`Using filesort`/`Using temporary`（危险信号）** |

**亲手实验**：先 `DROP INDEX idx_user ON orders;` 再 EXPLAIN——type 变成 ALL；把索引加回来再看——type 变 ref。**亲眼看到索引的作用，比任何讲解都深刻。**

### 3.3 索引设计三原则

**原则一：最左前缀（联合索引的匹配规则）**
建联合索引 `(a, b, c)`，相当于建了一部按 a、再按 b、再按 c 排序的目录。它能服务：
- `WHERE a = ?` ✅
- `WHERE a = ? AND b = ?` ✅
- `WHERE a = ? AND b = ? AND c = ?` ✅
- `WHERE b = ?` ❌ ——跳过了 a，目录没法直接用（类比：电话簿按"姓+名"排序，只给"名"没法查）

**原则二：覆盖索引（查询的列都在索引里，不用回表）**
普通索引的叶子节点只存"索引列 + 主键"。如果查询还需要别的列，就要拿主键再回主索引查一次（叫"回表"）。如果 SELECT 的列恰好都在联合索引里，直接返回，`Extra` 显示 `Using index`，一次都不用回表。

```sql
-- 建索引 (user_id, create_time) 后，这条查询覆盖索引生效：
SELECT user_id, create_time FROM orders WHERE user_id = 1;
```

**原则三：高选择性优先**
"选择性" = 不同值数量 / 总行数。订单号几乎每行不同（选择性高，适合索引）；性别只有两种值（选择性极低，索引没意义，数据库索性全表扫）。

### 3.4 索引失效的四个经典场景（背下来）

1. **对索引列做运算/函数**：`WHERE DATE(create_time) = '2026-08-16'` ❌ → 改成范围：`WHERE create_time >= '2026-08-16' AND create_time < '2026-08-17'` ✅
2. **隐式类型转换**：phone 列是字符串，却 `WHERE phone = 13800001111`（数字）❌
3. **前导通配符**：`WHERE name LIKE '%三'` ❌（`LIKE '张%'` 可以用索引 ✅）
4. **OR 连接无索引列**：`WHERE indexed_col = 1 OR no_index_col = 2` ❌

### 3.5 深分页问题

`LIMIT 100000, 20`：数据库要取出前 100020 行再扔掉前 100000 行，越翻越慢。
**游标分页**：`WHERE id > 上次最后一条的id ORDER BY id LIMIT 20`——直接跳到位置，速度恒定。这就是"下拉加载更多"比"跳转到第 N 页"好做的原因。

---

## 4. 事务与连接池（1 小时）

### 4.1 事务：要么全成功，要么全不发生

**经典故事**：张三给李四转账 100 元，需要两步：张三 -100，李四 +100。如果第一步成功、第二步时系统断电——钱凭空消失了。

**事务把多步操作捆成一个原子整体**：

```python
with Session(engine) as session:
    try:
        zhangsan.balance -= 100
        lisi.balance += 100
        session.commit()      # 两步一起生效
    except Exception:
        session.rollback()    # 任何一步出错：全部撤销，像什么都没发生过
        raise
```

**ACID 四个特性**（面试常考，理解即可）：
- **A 原子性**：要么全做，要么全不做（上面转账的例子）；
- **C 一致性**：事务前后数据都满足业务规则（如余额不为负）；
- **I 隔离性**：并发事务互不干扰（下一个话题）；
- **D 持久性**：提交后即使断电数据也在。

### 4.2 隔离级别：并发事务的三种怪象

多个事务同时跑，如果没有隔离会出三种乱子：

| 怪象 | 一句话解释 |
|---|---|
| 脏读 | 读到了别人**还没提交**的数据（对方可能回滚，你读到的是"假数据"）|
| 不可重复读 | 同一事务里两次读同一行，结果不一样（被别人改了并提交了）|
| 幻读 | 同一事务里两次按条件查询，第二次多出新行（别人插入了符合条件的行）|

四个隔离级别逐级加码：**Read Uncommitted（全都有）→ Read Committed（防脏读）→ Repeatable Read（防脏读+不可重复读，MySQL 默认，InnoDB 还基本防住了幻读）→ Serializable（全防，但慢到基本不用）**。

> 记忆点：**MySQL 默认 Repeatable Read，PostgreSQL 默认 Read Committed**——这是两者最常见的差异考点。

### 4.3 乐观锁 vs 悲观锁：秒杀防超卖的数据库方案

**悲观锁**：`SELECT ... FOR UPDATE`——先把这行锁住，别人排队等我改完。安全但并发度低。

**乐观锁**：不加锁，更新时检查"我读的版本/库存还有效吗"：

```sql
-- 一条 SQL 完成"检查 + 扣减"，利用单行更新的原子性：
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0;
-- 返回 affected_rows = 1 → 扣成功；= 0 → 库存不足，扣失败
```

没有显式加锁，但 `WHERE stock > 0` 保证了**绝不会扣成负数**。冲突少时乐观锁吞吐量远高于悲观锁——秒杀落库就用它（下午项目见）。

### 4.4 连接池：为什么需要、怎么配

**每次请求都新建数据库连接**的代价：TCP 握手 + MySQL 认证，一次要几十毫秒——比查询本身还慢。连接池的思路是**预先建好一批连接反复复用**，类比共享单车：不是每个人买一辆车（新建连接），而是池子里取一辆用完还回去。

SQLAlchemy 的四个关键参数：

| 参数 | 含义 | 建议 |
|---|---|---|
| `pool_size` | 池里常驻连接数 | 默认 5；按"实例数 × pool_size < MySQL max_connections"估算 |
| `max_overflow` | 高峰时允许临时多开几个 | 默认 10，超过就排队等待 |
| `pool_timeout` | 排队等连接的最长秒数 | 默认 30，超时抛错——**线上卡住先看是不是池耗尽** |
| `pool_recycle` | 连接最大存活秒数 | **必须小于 MySQL 的 wait_timeout（默认 28800s）**，建议 3600 |

**pool_recycle 的经典坑**：MySQL 会把 8 小时没活动的连接悄悄断掉，但池子不知道。第二天第一个请求拿到这具"尸体连接"，报 `Lost connection to MySQL server`。设了 `pool_recycle=3600`，连接活到 1 小时就主动退休换新，永远踩不到这个坑。

**池耗尽的排查思路**：请求卡在获取连接 → 检查是不是有 Session 没关（泄漏）、慢 SQL 长期占用连接、池配得太小。

### 4.5 事务使用红线

- **事务里绝不调用外部接口**（HTTP、RPC）：外部调用耗时不确定，事务一直不提交，锁和连接一直占着，高并发下连接池几分钟就死。正确顺序：先调外部接口拿结果，再开短事务写库。
- **事务越短越好**：只把"必须同生共死的写操作"放进事务，查询和业务判断放外面。

---

## 5. 综合实战项目：秒杀系统（4 小时）

> 这是三天的总装工程。跟着里程碑一步步做，每完成一个里程碑就验证一次，不要跳步。

### 5.0 架构总览

```
                        ┌─────────────┐
   客户端请求 ────────▶ │  API 网关    │  Day 1：鉴权 + 限流 + 路由
                        └──────┬──────┘
              ┌────────────────┼─────────────────┐
              ▼                ▼                 ▼
      ┌──────────────┐ ┌──────────────┐  ┌──────────────┐
      │ user-service │ │product-service│  │ order-service │
      │    :8001     │ │    :8003      │  │    :8002     │
      └──────────────┘ └──────┬───────┘  └──────┬───────┘
                              │                 │
                              ▼                 ▼
                        ┌──────────┐      ┌──────────┐      ┌──────────┐
                        │  Redis   │      │ RabbitMQ/ │      │  MySQL   │
                        │ 缓存+库存 │      │ Redis队列 │      │ 订单落库  │
                        │ 预扣+锁   │      │  削峰     │      │ 事务+索引 │
                        └──────────┘      └──────────┘      └──────────┘
                          Day 2            Day 4 升级         Day 3
```

**为什么这样设计（每层存在的理由，面试必问）**：
- 秒杀流量是平时的几百倍，**直接打数据库 = 自杀**，所以要层层拦截；
- **Redis 预扣库存**：99% 的请求（库存已空、重复下单）在 Redis 层就被拦掉，根本到不了数据库；
- **消息队列削峰**：抢到的人不直接同步写库，而是任务入队立即返回"抢购中"，worker 按数据库能承受的速度慢慢落库——洪峰被队列"削平"；
- **数据库乐观锁兜底**：即使前面所有层都失守，`UPDATE ... WHERE stock > 0` 保证账绝不会错。

### 5.1 建表与索引（里程碑 M1，45 分钟）

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
    UNIQUE KEY uk_order_no (order_no),                -- 幂等防线 1：单号唯一，重复插入直接报错
    UNIQUE KEY uk_user_product (user_id, product_id), -- 幂等防线 2：同一用户同一商品只能买一次
    KEY idx_product_time (product_id, create_time)    -- 联合索引：按商品查订单并按时间排序，一次命中
);

-- 造数据：1 号商品，库存 50
INSERT INTO product (name, price, stock) VALUES ('秒杀手机', 3999.00, 50);
```

**每行索引的设计意图**（这是考点）：`uk_order_no` 和 `uk_user_product` 两道唯一索引是**幂等的最后防线**——即使 Redis 防重失效、消息重复投递，数据库层面也绝不可能出现重复订单。`idx_product_time` 同时满足"按商品过滤"和"按时间排序"（最左前缀）。

用 SQLAlchemy 建好对应模型（参照第 2 节的写法），并用 `echo=True` 确认建表语句符合预期。

### 5.2 秒杀接口：Redis 防线（里程碑 M2，60 分钟）

在 order-service 中实现核心接口：

```python
# order-service 的秒杀接口
import json
import uuid
import redis.asyncio as aioredis
from fastapi import FastAPI, HTTPException

app = FastAPI()
r = aioredis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

async def get_product_cached(product_id: int):
    """商品信息走缓存（Day 2 的 Cache-Aside + 防穿透）"""
    key = f"product:{product_id}"
    cached = await r.get(key)
    if cached is not None:
        return None if cached == "null" else json.loads(cached)
    # 未命中则查库重建（此处省略锁，复用 Day 2 实战 2 的完整版）
    ...  # 查 MySQL，写缓存，不存在则缓存 "null"

@app.post("/seckill/{product_id}")
async def seckill(product_id: int, user_id: int):
    # 第 1 道防线：商品校验（走缓存，不打数据库）
    product = await get_product_cached(product_id)
    if not product:
        raise HTTPException(status_code=404, detail="商品不存在")

    # 第 2 道防线：防重复下单——SET NX 天然原子，一个人只能成功一次
    dup = await r.set(f"seckill:user:{user_id}:{product_id}", "1", nx=True, ex=3600)
    if not dup:
        raise HTTPException(status_code=400, detail="请勿重复下单")

    # 第 3 道防线：Redis 预扣库存——DECR 原子，库存空了的请求在这里全被拦下
    stock = await r.decr(f"seckill:stock:{product_id}")
    if stock < 0:
        await r.incr(f"seckill:stock:{product_id}")   # 没抢到，把多扣的还回去
        raise HTTPException(status_code=400, detail="已售罄")

    # 三关都过了：生成订单任务入队，立即返回（不在这儿写数据库！）
    task = {"order_no": uuid.uuid4().hex[:24], "user_id": user_id, "product_id": product_id}
    await r.rpush("seckill:queue", json.dumps(task))
    return {"msg": "抢购中，请稍后查询结果", "order_no": task["order_no"]}
```

**活动开始前**，先把数据库库存同步进 Redis：

```bash
redis-cli SET seckill:stock:1 50
```

**此刻验证**：用 curl 调接口——第 51 个请求开始全部返回"已售罄"，同一 user_id 第二次请求返回"请勿重复下单"。

### 5.3 Worker：异步落库（里程碑 M3，60 分钟）

独立运行的 worker 进程，从队列取任务、在事务中落库：

```python
# worker.py —— 单独 python worker.py 运行
import json
import time
import redis as sync_redis
from sqlalchemy import update
from sqlalchemy.orm import Session
# from models import engine, Product, SeckillOrder   # 复用 M1 的模型

r = sync_redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def handle_task(task: dict) -> bool:
    with Session(engine) as session:
        try:
            # 乐观锁扣库存：WHERE stock > 0 保证不会扣成负数（Day 3 第 4.3 节）
            result = session.execute(
                update(Product)
                .where(Product.id == task["product_id"], Product.stock > 0)
                .values(stock=Product.stock - 1)
            )
            if result.rowcount == 0:
                return False            # 数据库库存不足（理论上 Redis 已拦，这是兜底）
            # 插订单：唯一索引 uk_order_no / uk_user_product 保证重复任务不会重复入库
            session.add(SeckillOrder(**task, status=1))
            session.commit()
            return True
        except Exception:
            session.rollback()
            return False

while True:
    item = r.blpop("seckill:queue", timeout=5)    # 阻塞取任务，没有就等 5 秒
    if not item:
        continue
    task = json.loads(item[1])
    ok = handle_task(task)
    if not ok:
        r.incr(f"seckill:stock:{task['product_id']}")      # 落库失败：回补 Redis 库存
        r.delete(f"seckill:user:{task['user_id']}:{task['product_id']}")  # 并解除防重标记
    print("processed:", task["order_no"], "success:", ok)
```

**此刻验证**：连续下 3 个单，worker 终端逐条打印 processed；查 MySQL `SELECT * FROM seckill_order;` 有 3 行；`SELECT stock FROM product WHERE id=1;` 是 47——**Redis 库存（47）与数据库库存（47）一致**。

### 5.4 接入网关（里程碑 M4，30 分钟）

把 order-service 注册进 Day 1 的注册中心，网关路由表加 `"seckill"` 前缀。所有秒杀流量统一从 8000 端口进入，享受鉴权和限流保护。

### 5.5 压测与调优（里程碑 M5，45 分钟）

```python
# stress_test.py —— 200 个并发用户抢 50 件库存
import asyncio
import httpx

async def one_request(client, uid):
    try:
        resp = await client.post(
            f"http://127.0.0.1:8000/seckill/1?user_id={uid}",
            headers={"Authorization": "token-zhangsan"},   # 网关鉴权
            timeout=10,
        )
        return resp.status_code
    except httpx.HTTPError:
        return -1

async def main():
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(*[one_request(client, uid) for uid in range(200)])
    from collections import Counter
    print(Counter(results))

asyncio.run(main())
```

**预期**：约 50 个 200（抢到），150 个 400（售罄或重复）。等 worker 消化完队列，验证三件一致性：
1. MySQL `seckill_order` 恰好 50 行，无重复 `order_no`、无重复 `user_id`；
2. MySQL `product.stock = 0`；
3. Redis `seckill:stock:1 = 0`。

**项目验收标准（全部满足才算毕业）**：
- [ ] 200 并发无超卖，三处库存数据一致
- [ ] 压测期间 MySQL 的 product 表读查询几乎为零（缓存在扛）
- [ ] 用 EXPLAIN 证明按商品查订单命中了联合索引
- [ ] 能向人完整讲解：一个秒杀请求从发出到落库，经过了哪 7 个环节、每一环防的是什么

---

## 6. Day 3 自测清单

- [ ] 写出 INSERT/SELECT（带 WHERE/ORDER BY/LIMIT）/UPDATE/DELETE 各一条，并解释 JOIN 在干什么
- [ ] 什么是 N+1 问题？怎么发现、怎么解决？
- [ ] EXPLAIN 的 type=ALL 和 Extra=Using filesort 分别意味着什么？
- [ ] 联合索引 `(a,b,c)` 能命中哪些查询？为什么不能命中 `WHERE b=?`？（电话簿类比）
- [ ] 背出索引失效的 4 个经典场景
- [ ] 事务的 ACID 各是什么？MySQL 默认隔离级别是什么，防住了哪些怪象？
- [ ] 乐观锁 `UPDATE ... WHERE stock > 0` 为什么能防超卖？和悲观锁比优势在哪？
- [ ] 连接池 4 个参数各管什么？`pool_recycle` 解决的是什么坑？
- [ ] 秒杀系统为什么要"Redis 预扣 + 队列 + 异步落库"三层，直接写库行不行？

**明天预告**：今天用 Redis List 当队列是"简配版"——消息会丢、没有确认机制。明天升级装备：专业消息队列 RabbitMQ、用 Docker 把整个系统一键打包、给系统装上真正的登录认证。
