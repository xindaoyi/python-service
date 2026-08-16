# Day 2：Redis 应用 · 详细教程

> 今日目标：看到业务场景能立刻说出该用哪种 Redis 数据结构；讲透缓存穿透/击穿/雪崩并写出防护代码；手写生产可用的分布式锁。
> 时间：约 7.5 小时（理论 2.5h + 实战 3h + 项目 2h）

---

## 1. Redis 是什么、为什么需要它（45 分钟）

### 1.1 从一个性能故事讲起

Day 1 你的电商服务把数据存在内存字典里。考虑两个现实问题：

1. **数据库很慢**。真实业务里数据存在 MySQL（磁盘上）。一次磁盘读大约几毫秒，而一次内存读大约 0.1 微秒——**差了上万倍**。如果一个商品详情页每次访问都查数据库，几万 QPS 的秒杀流量瞬间就把数据库打垮。
2. **内存字典不能共享**。你的服务部署了 4 个实例（4 个进程），每个进程各有一份 `USERS` 字典，互不同步；进程重启数据全丢。

Redis 同时解决这两个问题：**它是一个运行在独立进程里的、基于内存的键值数据库**。

```
服务实例1 ─┐
服务实例2 ──┼──▶ Redis（内存，所有服务共享，可选持久化到磁盘）──▶ MySQL（兜底存储）
服务实例3 ─┘
```

**前端类比**：Redis 之于后端服务，就像 localStorage 之于你的页面——都是键值存储、都用来缓存数据加速。但 localStorage 属于单个浏览器标签页，Redis 属于**整个服务端集群共享**。

### 1.2 Redis 的三个关键词

- **内存存储**：所有数据在内存里，所以快（单机轻松 10 万+ QPS）；
- **键值（KV）模型**：通过 key 存取 value，没有 SQL、没有表关联——用简单换速度；
- **丰富的数据结构**：value 不只是字符串，还可以是哈希、列表、集合、有序集合等——这是 Redis 和"只会 get/set 的缓存"的本质区别，也是今天的主菜。

> 补充：Redis 也能把数据定期写入磁盘（RDB/AOF 持久化），所以它不是"重启就丢"的纯缓存，但绝大多数业务把它当缓存用，真正的账还是记在 MySQL 里。

### 1.3 先和 Redis 握个手

确认容器在跑（`docker ps` 里有 redis），然后用自带的命令行客户端玩玩：

```bash
redis-cli
127.0.0.1:6379> SET name "zhangsan"
OK
127.0.0.1:6379> GET name
"zhangsan"
127.0.0.1:6379> SET counter 100
127.0.0.1:6379> INCR counter      # 原子自增，返回 101
(integer) 101
127.0.0.1:6379> EXPIRE name 60    # 给 key 设置 60 秒过期
127.0.0.1:6379> TTL name          # 查看剩余秒数
(integer) 55
```

注意 `INCR` 是**原子操作**——100 个客户端同时执行 INCR，结果精确地加 100。Redis 是单线程处理命令的，每条命令天然排队执行，不会出现"两个请求同时改一个值改乱了"的问题。这个特性今天会反复用到。

---

## 2. 核心概念一：数据结构及适用场景（1.5 小时）

> 学习心法：不要背命令，背**场景**。面试和工作中被问的都是"这个场景你用什么存"，命令现查就行。

### 2.1 String（字符串）——万能基础款

value 就是一个字符串（可以放文本、数字、JSON）。

```python
import redis
r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)
# decode_responses=True：返回 str 而不是 bytes，建议总是加上

r.set("article:1001:title", "Redis 从入门到精通")
r.incr("article:1001:views")              # 阅读量 +1，原子操作
r.set("session:abc123", "user_id:1", ex=3600)   # 登录态，1 小时自动过期
```

**典型场景**：
- **计数器**：阅读量、点赞数（INCR 原子自增，不怕并发）；
- **缓存对象**：把 Python 对象 `json.dumps` 后整个存进去；
- **登录态/验证码**：`set 键 值 ex=秒数`，过期自动消失；
- **分布式锁**：今天的压轴戏，就靠 `set nx ex`。

### 2.2 Hash（哈希）——存"一个对象的多个字段"

value 本身又是一组 field-value 对，类似 Python 的嵌套字典。

```python
r.hset("user:1", mapping={"name": "张三", "points": "100", "level": "vip"})
r.hget("user:1", "name")            # 只取一个字段
r.hincrby("user:1", "points", 20)   # 只给积分加 20，不用读出整个对象
r.hgetall("user:1")                 # {'name': '张三', 'points': '120', 'level': 'vip'}
```

**典型场景**：存用户信息、商品信息等"对象"，且**经常只更新其中一两个字段**。
**和 String 存 JSON 的对比**：JSON 方式每次改一个字段都要"读出整个 JSON → 反序列化 → 改 → 序列化 → 写回"，并发下还会互相覆盖；Hash 直接改单个字段，原子且省流量。

### 2.3 List（列表）——有序的队列

value 是一个有序字符串列表，可以两头进出。

```python
r.lpush("notifications", "消息1", "消息2", "消息3")   # 从左边塞入
r.rpop("notifications")                                # 从右边取出 → "消息1"（先进先出）
r.lrange("notifications", 0, 9)                        # 取前 10 条（类似数组切片）
```

**典型场景**：
- **简单队列**：`LPUSH` 入队 + `BRPOP` 阻塞出队（Day 3 的秒杀项目会用它削峰）；
- **最新 N 条**：最新评论、最新动态——永远 LPUSH 到头部，LTRIM 只保留前 100 条。

> ⚠️ List 队列没有"消费确认"机制，消费者取出后宕机消息就丢了。严肃场景用 RabbitMQ（Day 4），List 只用于"丢了也不心疼"的场景。

### 2.4 Set（集合）——去重与关系运算

value 是一组**不重复**的字符串，支持交并差运算。

```python
r.sadd("article:1:likers", "u1", "u2", "u3")
r.sadd("article:1:likers", "u1")        # 重复添加无效 → 天然防重复点赞
r.sismember("article:1:likers", "u2")   # True：u2 点过赞吗
r.sadd("user:u1:follows", "a", "b", "c")
r.sadd("user:u2:follows", "b", "c", "d")
r.sinter("user:u1:follows", "user:u2:follows")   # {'b', 'c'} 共同关注！
```

**典型场景**：点赞去重、标签系统、共同好友/共同关注（SINTER）、抽奖（SRANDMEMBER 随机抽人）。

### 2.5 ZSet（有序集合）——排行榜之王

value 是 Set 的升级版：每个成员带一个**分数（score）**，自动按分数排序。

```python
r.zadd("hot:rank", {"Python教程": 98, "Redis实战": 85, "微服务入门": 76})
r.zincrby("hot:rank", 5, "Redis实战")                  # 热度 +5 → 90 分，自动升到第 1
r.zrevrange("hot:rank", 0, 2, withscores=True)         # Top 3（分数从高到低）
```

**典型场景**：
- **排行榜**：热搜、销量榜、积分榜——新增/加分都是 O(log N)，百万级数据毫秒响应；
- **延迟队列**：把"到期时间戳"当 score，`ZRANGEBYSCORE 0 现在` 捞出所有到期任务（定时取消未支付订单就是这么做的）；
- **范围查询**：`ZRANGEBYSCORE score 60 100` 查 60~100 分的所有成员。

### 2.6 扩展结构（了解即可，面试加分）

| 结构 | 一句话 | 场景 |
|---|---|---|
| Bitmap | String 的位操作，一个 key 最多存 42 亿个布尔位 | 用户签到：`SETBIT sign:u1:2026 227 1` 表示今年第 227 天签到了，亿级用户一年只占几百 MB |
| HyperLogLog | 概率型计数器，误差约 0.81%，**固定只占 12KB** | 统计 UV（页面独立访客），不需要精确、只要量级时用 |
| Geo | 基于 ZSet 存经纬度 | "附近的人/附近的店" |

### 2.7 速记口诀与常见误区

**口诀**：计数 String，对象 Hash，排队 List，去重交友 Set，排名延时 ZSet，签到 Bitmap，数 UV  HyperLogLog。

| 误区 | 解析 |
|---|---|
| 大 Key | 一个 value 里塞几百万成员（如全站粉丝列表放一个 Set）。删除/传输时阻塞单线程的 Redis，全站卡顿。**大 Key 要拆分** |
| 热 Key | 某个 key 被超高频访问（如顶流明星的微博计数），单节点扛不住。**本地缓存或把 key 拆成多份** |
| 把 Redis 当万能数据库 | 内存贵且容量有限。热数据放 Redis，冷数据、账本数据永远在 MySQL |

---

## 3. 核心概念二：缓存三大问题（1 小时）

### 3.1 先立规矩：Cache-Aside 模式

在讲三大问题之前，先确立业界 90% 业务的缓存标准用法——**旁路缓存（Cache-Aside）**：

```
读请求：
  ① 先查 Redis ──命中──▶ 直接返回（完）
       │未命中
  ② 查 MySQL
  ③ 把结果写回 Redis（设过期时间）
  ④ 返回

写请求：
  ① 先更新 MySQL
  ② 再【删除】Redis 里对应的 key（注意：是删除，不是更新！）
```

**为什么写操作是"删缓存"而不是"改缓存"？**
假设改缓存：请求 A 和请求 B 并发更新同一数据，"A 改库 → B 改库 → B 改缓存 → A 改缓存"这样的交错会让缓存里留下 A 的旧值，和数据库不一致。而"删缓存"不管怎么交错，下次读都会从数据库重建出最新值。简单且安全。

### 3.2 缓存穿透：查"根本不存在"的数据

**故事**：你的商品接口是 `/products/{id}`，缓存和数据库里 id 都是 1~100。有人写了个脚本，用 id = -1、-2、-3……疯狂请求。这些 id **缓存里永远不会有**（因为数据库里也没有，没什么可缓存的），于是每个请求都穿过缓存直达数据库——缓存形同虚设，数据库被打垮。

**对策**：
1. **缓存空值**：数据库查不到，也把 `key → "null"` 存进 Redis（TTL 短一点，30~60 秒）。下次同样的 id 再来，直接命中空值返回。简单有效，缺点是恶意请求 id 每次都不同（随机生成）时会撑出大量垃圾 key。
2. **布隆过滤器**：一个超省内存的概率结构，提前把所有"存在的 id"录入。请求来了先问它："这个 id 可能存在吗？"——回答"不存在"就**一定不存在**，直接拦截；回答"存在"才放行去查缓存。缺点是有小概率误判（把不存在的说成存在，但漏过去一个请求也无妨）。

### 3.3 缓存击穿：一个"超级热点"刚好过期

**故事**：某顶流商品详情在 Redis 里的 key 过期的那一瞬间，正好涌进来 1 万个请求。它们全部发现缓存未命中，**同时**去查数据库、同时重建缓存——1 万次重复的数据库查询在同一秒砸下去，数据库当场倒地。

**和穿透的区别**：穿透是"数据不存在"，击穿是"数据存在、是热点、但缓存恰好失效"。

**对策**：
1. **互斥重建**：第一个发现缓存失效的请求**加一把锁**，只有它去查库重建；其他请求等锁释放后读新缓存。1 万次数据库查询变成 1 次。
2. **逻辑过期**：热点 key 不设物理 TTL（永不过期），把"过期时间"存进 value 里。发现逻辑上过期了，也不删 key，而是**异步**开一个线程去更新，其他请求继续返回旧数据——永远不会有请求打到数据库，代价是短暂的数据不新鲜。

### 3.4 缓存雪崩：一批 key 同时倒下

**故事**：运营批量导入了 10 万个商品到缓存，过期时间都设成 300 秒。5 分钟后，这 10 万个 key **在同一秒内集体过期**，流量瞬间全部涌向数据库。另一种更惨的雪崩：Redis 本身宕机了。

**对策**：
1. **TTL 加随机值**：`300 + random(0, 60)` 秒，让过期时间错开，避免齐步走；
2. **多级缓存**：服务本地内存再放一层短 TTL 缓存（如 5 秒），Redis 挂了还能顶一阵；
3. **Redis 高可用**：主从 + 哨兵/集群，单点故障自动切换；
4. **熔断降级兜底**：数据库前面加限流，宁可部分请求返回"系统繁忙"，也不能让数据库死掉（数据库死了就是全站死）。

### 3.5 一图总结

| | 穿透 | 击穿 | 雪崩 |
|---|---|---|---|
| 一句话 | 查不存在的数据 | 一个热点 key 过期 | 一批 key 同时过期 |
| 关键词 | 不存在 | 单个、热点 | 批量、同时 |
| 对策 | 缓存空值 / 布隆过滤器 | 互斥锁重建 / 逻辑过期 | 随机 TTL / 多级缓存 / 高可用 |

---

## 4. 核心概念三：分布式锁（1 小时）

### 4.1 问题：单机锁出了机房就失灵

Python 的 `threading.Lock` 你肯定见过。但微服务部署了 4 个实例（4 台机器、4 个进程），每个进程各有一把自己的锁——A 进程锁住的东西，B 进程畅通无阻。**跨进程的互斥，必须借助一个大家都认的"第三方"来裁决**，Redis 就是最常用的裁判。

### 4.2 五步演进：从能用到生产可用

每一步都在解决上一步的一个致命缺陷，跟着推导一遍比背结论管用十倍。

**第 1 步：SETNX + EXPIRE（有死锁风险）**

```python
r.setnx("lock", 1)     # set if not exists：不存在才能设置成功，成功者获得锁
r.expire("lock", 30)   # 再设置 30 秒过期，防止忘了解锁
```
缺陷：两条命令**不是原子的**。如果 `setnx` 成功后进程宕机，`expire` 没执行——锁永远不释放，**死锁**。

**第 2 步：一条命令原子加锁**

```python
r.set("lock", 1, nx=True, ex=30)   # 加锁和过期时间一条命令搞定，原子性解决死锁
```

**第 3 步：value 存唯一标识（防误删）**
新缺陷：业务执行超过 30 秒，锁自动过期；别的线程拿到锁开始工作；此时你的线程执行完了，一句 `delete` **把别人的锁删了**。
对策：加锁时 value 存一个 UUID（只有你自己知道），解锁前先比对"这把锁是不是我的"。

**第 4 步：Lua 脚本原子释放**
新缺陷："比对 UUID + 删除"是两步操作，如果比对通过、删除之前锁刚好过期被别人拿走——还是误删。
对策：把这两步写进一段 Lua 脚本发给 Redis 执行，**脚本在 Redis 里是原子执行的**（单线程模型保证脚本执行期间不插入其他命令）。

**第 5 步：看门狗续期**
最后的不完美：业务到底要执行多久说不准，30 秒可能不够。成熟方案（如 Java 的 Redisson）会启动一个后台"看门狗"线程，业务没结束就每 10 秒自动把锁续期到 30 秒。Python 项目里了解原理、合理设置过期时间即可。

### 4.3 冷静一下：Redis 锁的边界

Redis 分布式锁是**工程上的高可用最优解，不是数学上的绝对互斥**：主从切换时锁信息可能丢失（异步复制），极端情况下两个客户端会同时认为自己持有锁。所以——**金融扣款这种错一次就赔钱的场景，要用数据库乐观锁（Day 3 会学）；秒杀防超卖这种"高并发但允许极端小概率重试兜底"的场景，Redis 锁完全够用。**

---

## 5. 代码实战（3 小时）

### 实战 1：数据结构综合练习（30 分钟）

新建 `redis_practice.py`，把第 2 节的代码全部亲手跑一遍，再完成两个挑战：

**挑战 A（热搜榜）**：用 ZSet 实现一个迷你热搜：`search(word)` 每次被调用给词加 1 分，`top10()` 返回当前前十。模拟 50 个词的随机搜索，打印最终榜单。

**挑战 B（延迟队列）**：

```python
import time
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

def add_delay_task(task: str, delay_seconds: int):
    """把任务放入延迟队列，delay_seconds 秒后可被执行"""
    run_at = time.time() + delay_seconds      # 到期时间戳当 score
    r.zadd("delay_queue", {task: run_at})

def poll_due_tasks():
    """取出所有已到期的任务（由消费者循环调用）"""
    now = time.time()
    tasks = r.zrangebyscore("delay_queue", 0, now)   # score <= 现在的都到期了
    for t in tasks:
        r.zrem("delay_queue", t)                      # 取出来就删掉
    return tasks

# 试一试
add_delay_task("取消15分钟未支付的订单1001", 3)
time.sleep(4)
print(poll_due_tasks())     # 应该打印出那条订单任务
```

**想一想**：这个延迟队列如果两个消费者同时 poll，会不会拿到同一条任务？（提示：zrangebyscore 和 zrem 不是原子的——这正是下一场实战要解决的思维。）

### 实战 2：缓存穿透/击穿/雪崩防护（60 分钟）

目标：写一个 `get_product()`，把第 3 节的所有对策变成代码。

```python
# cache_guard.py
import json
import random
import time
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

DB_CALLS = 0   # 统计数据库被查了几次，用来验证缓存是否真的生效

def fake_db_query(product_id: int):
    """模拟数据库：只有 1~100 的商品存在；每次查询耗时 100ms（磁盘就是慢）"""
    global DB_CALLS
    DB_CALLS += 1
    time.sleep(0.1)
    if 1 <= product_id <= 100:
        return {"id": product_id, "name": f"商品{product_id}", "price": 99.0}
    return None

def get_product(product_id: int):
    key = f"product:{product_id}"

    # ① 先查缓存（命中空值也算命中——防穿透）
    cached = r.get(key)
    if cached is not None:
        return None if cached == "null" else json.loads(cached)

    # ② 缓存未命中：加互斥锁，只让一个请求去重建（防击穿）
    lock_key = f"lock:product:{product_id}"
    if r.set(lock_key, "1", nx=True, ex=10):
        try:
            data = fake_db_query(product_id)
            if data is None:
                r.set(key, "null", ex=60)                         # 防穿透：缓存空值
            else:
                ttl = 300 + random.randint(0, 60)                 # 防雪崩：TTL 加抖动
                r.set(key, json.dumps(data), ex=ttl)
            return data
        finally:
            r.delete(lock_key)
    else:
        # 没拿到锁：别人正在重建，等一会儿读缓存
        time.sleep(0.05)
        cached = r.get(key)
        if cached and cached != "null":
            return json.loads(cached)
        return None
```

**验证实验（亲眼看到三个对策生效）**：

```python
# 实验 1：防穿透 —— 查询不存在的商品 20 次
DB_CALLS = 0
for _ in range(20):
    get_product(9999)
print("实验1 查不存在商品20次，DB 被调用：", DB_CALLS, "次")   # 预期：只有 1 次

# 实验 2：防击穿 —— 50 个线程同时查同一个热点商品（缓存为空时）
import threading
DB_CALLS = 0
r.delete("product:1")
threads = [threading.Thread(target=get_product, args=(1,)) for _ in range(50)]
for t in threads: t.start()
for t in threads: t.join()
print("实验2 50并发查热点商品，DB 被调用：", DB_CALLS, "次")    # 预期：1 次（互斥锁生效）
```

**练习**：实现一个简单布隆过滤器——初始化时把 1~100 的 id 用 3 个不同哈希函数（如 `hash(f"{salt}{id}") % 10000`）在 `SETBIT` 里置位；查询前先检查 3 个位是否都为 1，任一不为 1 直接返回"不存在"。加在 `get_product` 最前面，再跑实验 1，DB 调用应变为 0 次。

### 实战 3：生产可用的分布式锁（60 分钟）

```python
# distributed_lock.py
import asyncio
import uuid
import redis.asyncio as aioredis

# Lua 脚本：只有 value 匹配（锁是我的）才删除——两步操作原子执行
UNLOCK_SCRIPT = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
"""

class RedisLock:
    """异步上下文管理器形式的分布式锁：async with RedisLock(...) 自动加解锁"""

    def __init__(self, r: aioredis.Redis, key: str, expire: int = 30):
        self.r = r
        self.key = key
        self.expire = expire
        self.token = str(uuid.uuid4())     # 唯一标识：防止误删别人的锁（第 3 步）

    async def acquire(self, retry: int = 3, interval: float = 0.1) -> bool:
        for _ in range(retry):
            # 一条命令完成加锁+过期，原子（第 2 步）
            ok = await self.r.set(self.key, self.token, nx=True, ex=self.expire)
            if ok:
                return True
            await asyncio.sleep(interval)   # 拿不到就稍等重试
        return False

    async def release(self):
        await self.r.eval(UNLOCK_SCRIPT, 1, self.key, self.token)   # Lua 原子释放（第 4 步）

    async def __aenter__(self):
        if not await self.acquire():
            raise RuntimeError("acquire lock failed")
        return self

    async def __aexit__(self, *exc):
        await self.release()
```

**终极实验：50 个并发抢 10 件库存**

```python
import asyncio
import redis.asyncio as aioredis
from distributed_lock import RedisLock

async def seckill_without_lock(r, stock_key):          # 对照组：不加锁
    stock = int(await r.get(stock_key) or 0)
    if stock <= 0:
        return False
    await asyncio.sleep(0.001)      # 模拟业务耗时，放大并发窗口
    await r.set(stock_key, stock - 1)
    return True

async def seckill_with_lock(r, stock_key):             # 实验组：加分布式锁
    async with RedisLock(r, "lock:seckill", expire=10):
        stock = int(await r.get(stock_key) or 0)
        if stock <= 0:
            return False
        await asyncio.sleep(0.001)
        await r.set(stock_key, stock - 1)
        return True

async def main():
    r = aioredis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

    await r.set("stock", 10)
    results = await asyncio.gather(*[seckill_without_lock(r, "stock") for _ in range(50)])
    print("无锁：成功", sum(results), "人，剩余库存", await r.get("stock"))   # 成功远超 10，库存变负数——超卖！

    await r.set("stock", 10)
    results = await asyncio.gather(*[seckill_with_lock(r, "stock") for _ in range(50)])
    print("有锁：成功", sum(results), "人，剩余库存", await r.get("stock"))   # 恰好成功 10 人，库存 0
    await r.aclose()

asyncio.run(main())
```

**预期输出**：无锁组卖出 20+ 件（超卖），有锁组恰好 10 件、库存为 0。**这一刻你会真正理解锁的价值。**

---

## 6. 练手项目：带缓存与锁的商品服务（2 小时）

把 Day 1 的 product-service 升级为"抗得住压"的版本：

**需求清单**：
1. `GET /products/{id}`：走 Cache-Aside，含防穿透（空值缓存）、防击穿（互斥重建）、防雪崩（随机 TTL）；
2. `POST /products/{id}/deduct`：扣库存接口，用 RedisLock 保证并发不超卖；
3. `PUT /products/{id}`：更新商品 → 先改"数据库"（内存字典模拟）→ 再删缓存（体会先库后缓存的顺序）；
4. 写一个压测脚本 `stress.py`：统计接口耗时和 DB 调用次数。

**验收标准**：
- 同一商品连查 100 次，DB 查询函数只被调用 1~2 次，平均响应 < 5ms（无缓存时 > 100ms）；
- 50 并发抢 20 件库存：成功恰好 20 人、库存恰好为 0；
- 更新商品后立刻查询，返回的是新数据（缓存被正确删除）；
- 能画出一次请求完整的"缓存/锁/数据库"流转图。

---

## 7. Day 2 自测清单

- [ ] 不看资料说出 5 种数据结构各自的 2 个场景（口诀：计数 String，对象 Hash……）
- [ ] 什么是大 Key、热 Key？各有什么危害？
- [ ] 默写 Cache-Aside 的读流程和写流程；为什么写操作是"删"缓存而不是"改"缓存？
- [ ] 一句话区分穿透/击穿/雪崩，各说出 2 种对策
- [ ] 分布式锁五步演进：每一步在解决上一步的什么问题？（死锁→原子加锁→误删→原子释放→续期）
- [ ] 为什么释放锁要用 Lua 脚本而不是"先 GET 比对再 DEL"？
- [ ] Redis 锁在什么情况下会失效？什么业务不该用它？
- [ ] 实操：20 分钟不看代码，重写 `get_product`（含三种防护）

**明天预告**：缓存再快也只是缓存，数据的"账本"永远在数据库。明天学 MySQL——从一条 SQL 都不会写，到能看懂执行计划、设计索引、管理事务，下午直接用秒杀项目把三天的东西焊在一起。
