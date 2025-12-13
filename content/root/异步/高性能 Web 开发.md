---
date created: 12月13日 , 3:20 , 2025
date modified: 12月13日 , 18:59 , 2025
---

这正是高性能 Python 后端的"深水区"。

在这一阶段，我们不仅要会写 CRUD，更要理解 **"为什么 SQLAlchemy 作为一个老牌同步 ORM，突然能支持异步了？"** 以及 **"asyncpg 为什么比其他驱动快？"**

我们将深入源码，揭示连接 **FastAPI (异步)** 与 **Database (IO)** 之间的桥梁。

---

## 🏛️ 架构全景图

在进入源码前，建立这个心理模型至关重要。你的数据流是这样走的：

`FastAPI (Asyncio Loop)` -> `SQLAlchemy (AsyncSession)` -> **`Greenlet (魔法层)`** -> `SQLAlchemy Core (Sync)` -> `Asyncpg Dialect` -> `asyncpg (Driver)` -> `PostgreSQL`

---

## 1. SQLAlchemy 的魔法：Greenlet 桥接源码解析

**源码位置：** `sqlalchemy/util/_concurrency_py3k.py` (及 `sqlalchemy/ext/asyncio/engine.py`)

这是最颠覆认知的点：**SQLAlchemy 的核心（Core）逻辑大部分依然是同步的！** 它并没有重写这部分代码，而是引入了 **Greenlet**（轻量级协程库）来"欺骗" Python。

### 📝 核心源码逻辑

当你调用 `await session.execute(stmt)` 时，SQLAlchemy 内部发生了"乾坤大挪移"。

```Python
# sqlalchemy/util/_concurrency_py3k.py

async def greenlet_spawn(fn, *args, **kwargs):
    """
    在 greenlet 中运行一个同步函数 fn，
    如果 fn 内部需要等待 IO，它会切回 asyncio 的 loop。
    """
    context = _AsyncIoContext(get_running_loop())
    # 创建一个新的 greenlet，目标是运行传入的同步函数
    run = greenlet(fn, context.switch)
    
    # 启动 greenlet
    result = run.switch(*args, **kwargs)
    
    # 循环：只要 greenlet 还没结束，且需要等待 awaitable
    while not run.dead:
        try:
            # ⭐ 关键点：这里把控制权交还给了 asyncio 事件循环
            # 等待实际的 IO (比如 asyncpg 的网络请求) 完成
            value = await result 
        except BaseException as err:
            result = run.throw(err)
        else:
            # IO 完成，带着结果切回 greenlet 继续运行 ORM 的逻辑
            result = run.switch(value)
            
    return result
```

### 🔍 深度解析

1. **同步代码异步化：** SQLAlchemy 的 ORM 层构建 SQL、处理映射全是同步 CPU 操作。
    
2. **切换机制：** 当代码运行到需要真正发送 SQL 的地方（调用 Driver），它会挂起当前的 Greenlet，抛出一个 `awaitable` 对象给外层的 `asyncio` 循环。
    
3. **IO 等待：** `asyncio` 循环（主线程）执行真正的网络 IO (`asyncpg`)。
    
4. **恢复现场：** 网络数据回来后，`await` 结束，主线程带着数据 `switch` 回 Greenlet，ORM 继续处理结果。
    

**实战启示：** 既然 ORM 构建过程涉及 CPU 计算且在 Greenlet 中运行，**如果你的 SQL 构建逻辑极其复杂（几千行），它依然会阻塞 Event Loop 一小会儿**，因为它本质上是在主线程中切换 CPU 上下文。

---

## 2. 速度之王：[[asyncpg]] 的协议层源码

**源码位置：** `asyncpg/protocol/protocol.pyx` (Cython) 和 `asyncpg/connection.py`

`asyncpg` 之所以快，不仅因为它异步，更因为它**绕过了 Python DB-API 2.0 标准**（该标准是为同步设计的），直接实现了 PostgreSQL 的 **二进制协议**。

### 📝 Connection 建立源码 (简化)

```Python
# asyncpg/connection.py

class Connection:
    async def connect(self, ...):
        # 直接使用 asyncio 底层的 create_connection
        # 这就是我们在第一阶段学的 loop 机制
        transport, protocol = await loop.create_connection(
            lambda: Protocol(self, ...),
            host, port, ...
        )
```

### 🔍 深度解析：Pipeline 与 Prepare

`asyncpg` 在源码层面做了大量的**预编译 (Prepared Statements)** 缓存。

当你执行：

```Python
await conn.fetch('SELECT * FROM users WHERE id = $1', 1)
```

`asyncpg` 内部 (Cython 层) 实际上做了：

1. **Parse:** 发送 SQL 结构给 PG，PG 解析并返回一个 Statement ID。
    
2. **Bind:** 发送参数（$1=1）给 PG。
    
3. **Execute:** 执行。
    

**源码级优化：** `asyncpg` 会自动缓存第一步。第二次执行相同 SQL 时，它直接发送 Statement ID 和参数。**这就是它比其他驱动快 3-5 倍的原因之一。**

---

## 3. 连接池：AsyncAdaptedQueuePool

**源码位置：** `sqlalchemy/pool/impl.py`

在 FastAPI 中，你不会为每个请求创建一个新连接，而是从池子里借。

### 📝 源码逻辑

异步连接池的核心在于如何**不阻塞地**等待空闲连接。

```Python
# 伪代码描述其核心逻辑
class AsyncAdaptedQueuePool:
    def __init__(self):
        # 使用 asyncio.Condition 而不是 threading.Condition
        self._cond = asyncio.Condition()
        self._connections = []

    async def acquire(self):
        async with self._cond:
            while not self._connections:
                if self._size < self._max_overflow:
                    # 创建新连接
                    return await self._create_connection()
                # ⭐ 关键：池子满了，await 等待，释放 Loop 给其他请求
                await self._cond.wait() 
            return self._connections.pop()
```

**实战启示：**

- **Pool Size 设置：** 如果你的 `pool_size` 太小（默认 5），高并发下大量请求会堆积在 `await self._cond.wait()` 这里。
    
- **Timeout：** 必须设置 `pool_timeout`。如果等待超过 30秒（默认），抛出 `TimeoutError`，防止请求无限挂起。

---

## 4. FastAPI 集成：最佳实践代码

结合以上源码理解，我们在 FastAPI 中配置 SQLAlchemy 2.0 的标准姿势如下：

```Python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from fastapi import Depends

# 1. 驱动选择：必须是 postgresql+asyncpg
DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/db"

# 2. Engine 配置
# echo=True 在开发时打开，能看到生成的 SQL，验证 Greenlet 是否正常工作
engine = create_async_engine(
    DATABASE_URL,
    echo=False,
    pool_size=20,     # 根据并发量调整
    max_overflow=10,  # 允许临时超出的连接数
    pool_timeout=30,
    pool_recycle=1800 # 防止连接被防火墙切断
)

# 3. Session 工厂
# expire_on_commit=False 是异步模式必须的
# 因为 commit 后连接归还池子，没法隐式刷新属性
AsyncSessionLocal = async_sessionmaker(
    bind=engine, 
    class_=AsyncSession, 
    expire_on_commit=False
)

# 4. Dependency (核心：利用 Generator 的生命周期)
async def get_db():
    async with AsyncSessionLocal() as session:
        # 这里实际上就是 contextmanager 的 __aenter__
        try:
            yield session
            # 请求处理完毕，FastAPI 重新激活这个生成器
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            # 这里的 __aexit__ 会自动 close session，归还连接给 Pool
            await session.close()
```

---

## 总结与下一步

通过深入源码，我们明白了：

1. **Greenlet** 是 SQLAlchemy 在不重写核心代码前提下实现异步的关键黑科技。
    
2. **Asyncpg** 通过二进制协议和 pipeline 提供了极致性能。
    
3. **AsyncAdaptedQueuePool** 利用 `asyncio.Condition` 实现了非阻塞的连接获取。

