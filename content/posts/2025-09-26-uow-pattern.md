+++
title = "Unit of Work (UoW) em Python com SQLAlchemy: transações explícitas, savepoints e outbox"
date = 2025-09-26T07:54:52+01:00
draft = false
description = "Como desenhei um UoW assíncrono e escalável com SQLAlchemy: fronteira transacional explícita, nesting com savepoints, outbox transacional, idempotência, e um gestor que funciona em API, CLI e workers."
tags = ["Python", "SQLAlchemy", "FastAPI", "Clean Architecture", "Unit of Work", "Arquitetura"]
categories = ["Engineering", "Architecture", "Persistence"]
slug = "uow-sqlalchemy-python"

[cover]
image = "images/cover/uow-cover.png"
alt = "Diagrama conceptual de um Unit of Work com camadas de aplicação e persistência"
+++

# Building a Production-Grade Unit of Work (UoW) System in Python with SQLAlchemy

When you build complex applications with **databases, services, and background jobs**, you eventually run into the **transaction problem**:

* How do I guarantee all operations happen **atomically**?
* How do I handle **nested scopes** (reuse vs rollback)?
* How do I ensure **events are only published once**?
* How do I protect against **duplicate commits** when retries happen?

The solution is a **Unit of Work (UoW) system** — a design pattern that **wraps all your persistence operations in a clear, explicit transaction boundary**.

In this post, I’ll show you how I built a **production-grade UoW system in Python 3.13** using **async SQLAlchemy**, with:

* Explicit transaction boundaries
* Safe nesting (reuse vs savepoint)
* Read-only transactions
* Transactional outbox for events
* Idempotency key protection
* Great testing story

---

## 🚨 The Problem: Repos Without Boundaries

A naive service might look like this:

```python
user = await user_repo.create(User(...))
order = await order_repo.create(Order(...))
await email_service.send_order_confirmation(order)
```

But what if the order commit fails while the email has already been sent?
You’ve just created an inconsistent state.

Without a transaction boundary, repositories can be called arbitrarily, leading to partial failures and ghost side effects.

---

## ✅ The Solution: Unit of Work + Manager

The **UoW system** enforces a strict rule:

👉 **All repos must be accessed through a UoW scope.**

A service now looks like this:

```python
async with uow_manager.enter(mode=UoWMode.REUSE) as uow:
    user = await uow.repos.user.create(User(...))
    order = await uow.repos.order.create(Order(...))
    uow.add_event("order.created", {"order_id": order.id})
```

At the end of the scope:

* If everything succeeded → **commit**.
* If something failed → **rollback**.
* Events are persisted only on **outermost commit**.

---

## 🧑‍💻 The Code

### SqlAlchemyUoWBase

This is the core UoW implementation. It wraps an `AsyncSession`, tracks events and idempotency keys, and handles nesting rules.

```python
class SqlAlchemyUoWBase[IRepos](IUnitOfWork[IRepos]):
    """Unit-of-Work facade with nesting support (reuse vs savepoint)."""

    def __init__(
        self,
        session: AsyncSession,
        level: int,
        *,
        read_only: bool = False,
        event_sink: EventSink,
        idem_box: IdemBox,
    ) -> None:
        self._s = session
        self._level = level
        self._read_only = read_only
        self._active = True
        self._tx = None
        self._events = event_sink
        self._idem_box = idem_box
        self._repos: IRepos | None = None

    ...
```

### UoWManager

The manager ensures that:

- Sessions are reused across nested scopes
- Events/idempotency are shared at all levels
- Commit/rollback/close are coordinated correctly

```python
class UoWManager[UowBaseT, IRepos](IUoWManager):
    """Creates UoW scopes over a shared AsyncSession with nesting support."""

    @asynccontextmanager
    async def enter(self, *, mode: UoWMode = UoWMode.REUSE, read_only: bool = False):
        mark_enter(self._label)
        try:
            if self._session is None:
                self._session = self._sf()
                self._event_sink = EventSink(items=[])
                self._idem_box = IdemBox()

            level = len(self._stack) + 1
            nested = level > 1 and mode is UoWMode.SAVEPOINT
            start_tx = (level == 1) or nested

            uow = self._uow_cls(
                self._session, level,
                read_only=read_only,
                event_sink=self._event_sink,
                idem_box=self._idem_box,
            )
            uow.set_repos(self._repo_factory(self._session))
            self._stack.append(uow)

            await uow.start(start_tx=start_tx)
            try:
                yield uow
                await uow.commit(started_tx=start_tx)
            except Exception:
                await uow.rollback(started_tx=start_tx)
                raise
            finally:
                self._stack.pop()
                if not self._stack:
                    await self._session.close()
                    self._session = None
                    self._event_sink = None
                    self._idem_box = None
        finally:
            mark_exit(self._label)
```

---

## 🔄 Visual Flow

### Outermost Transaction (Level 1)

```
BEGIN TRANSACTION
  ... repo operations ...
COMMIT
→ Flush outbox
→ Persist idempotency key
```

### Nested Reuse (Level > 1, REUSE)

```
(no BEGIN, reuse outer tx)
  ... repo operations ...
(no COMMIT, outer decides)
```

### Nested Savepoint (Level > 1, SAVEPOINT)

```
BEGIN NESTED (savepoint)
  ... repo operations ...
RELEASE SAVEPOINT
```

---

## 📦 Events + Idempotency

Both inner and outer scopes share **the same sinks**:

* `EventSink` → events are buffered and only persisted to the **outbox table** on the **outermost commit**
* `IdemBox` → idempotency key ensures retries don’t double-commit

```python
uow.add_event("user.created", {"id": 123})
uow.set_idempotency_key("req-uuid")

# Outer commit persists:
# - Outbox row
# - Idempotency key row
```

---

## 📊 Quick Comparison

| Mode          | Behavior                                         | Use Case                                |
| ------------- | ------------------------------------------------ | --------------------------------------- |
| **OUTERMOST** | Starts real transaction, commits, flushes events | Normal service call                     |
| **REUSE**     | Shares outer transaction, no BEGIN/COMMIT        | Helpers that must be atomic with parent |
| **SAVEPOINT** | Creates nested savepoint, rollback safe          | Risky operations (e.g., optional steps) |

---

## 🧪 Testing with Fake UoWManager

One of the biggest benefits of this design is that you can easily **fake the UoW in tests**.

Instead of creating a real SQLAlchemy session, you inject **in-memory repositories**.

### Fake Manager Example

```python
class FakeConfigRepo(IConfigRepo):
    def __init__(self):
        self.items = []

    async def get_configurations_by_group_name(self, group_name):
        return [i for i in self.items if i.group == group_name]


class FakeRepos(IConfigRepos):
    def __init__(self):
        self._config = FakeConfigRepo()

    @property
    def config(self) -> IConfigRepo:
        return self._config


class FakeUoW(IUnitOfWork[FakeRepos]):
    def __init__(self):
        self._repos = FakeRepos()
        self._active = True

    @property
    def repos(self) -> FakeRepos:
        return self._repos

    async def commit(self, *, started_tx: bool) -> None: ...
    async def rollback(self, *, started_tx: bool) -> None: ...
    async def close(self) -> None: ...
    def add_event(self, *, event_type: str, payload: dict) -> None: ...
    def set_idempotency_key(self, key: str | None) -> None: ...
    @property
    def level(self) -> int: return 1
    @property
    def is_active(self) -> bool: return self._active


class FakeUoWManager(IUoWManager):
    async def enter(self, *, mode=UoWMode.REUSE, read_only=False):
        uow = FakeUoW()
        yield uow
```

### Example Test

```python
async def test_get_configs_returns_items():
    mgr = FakeUoWManager()
    async with mgr.enter() as uow:
        uow.repos.config.items.append(ConfigItem(group="general", key="x", value="1"))

        service = ConfigService(uow.repos.config)
        result = await service.get_configurations_by_group_name("general")

        assert result.is_ok()
        assert len(result.unwrap()) == 1
```

This way:

* No database required
* No transactions opened
* Service logic is testable in isolation


---

## ⚡ Performance Notes

One subtle advantage of this design is **control over transaction cost**.

* **`REUSE` mode** is the fastest:

  * No new transaction, no savepoint, no roundtrips.
  * Best when nested operations are *always safe* to run under the parent scope.

* **`SAVEPOINT` mode** adds overhead:

  * Each savepoint requires SQL roundtrips (`SAVEPOINT`, `ROLLBACK TO`, `RELEASE`).
  * Use it when you need to **isolate risky steps** without aborting the entire parent transaction.

Example rule of thumb:

* Use **`REUSE`** for 95% of nested calls (normal service-to-service collaboration).
* Use **`SAVEPOINT`** sparingly (e.g., optional third-party calls, best-effort cleanups).

This balance keeps transactions **fast, predictable, and resilient**.

---

## 🏁 Closing Thoughts

The **Unit of Work pattern** is not just a persistence abstraction.
It’s a **contract**:

* *Every service runs inside a transaction.*
* *Events are only published if the transaction commits.*
* *Duplicate commits are prevented with idempotency.*

With this system, my services are safer, my tests are easier, and my architecture is cleaner.

In asynchronous apps built with SQLAlchemy + FastAPI, a combination of UoW Manager + UoW Base delivers safety, testability, and a clean architecture.

💬 Do you prefer implicit context (via contextvars) or explicit scopes like this?
Share your thoughts!

* [LinkedIn](https://www.linkedin.com/in/frederico-gago-5849281aa)
* [GitHub](https://github.com/fredericogago)