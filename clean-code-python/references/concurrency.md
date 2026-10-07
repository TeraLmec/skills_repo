## **Concurrency**

## Table of Contents

- [Concurrency](#concurrency)
- [Know which problem you have](#know-which-problem-you-have)
- [Keep concurrency code separate](#keep-concurrency-code-separate)
- [Limit the scope of shared data](#limit-the-scope-of-shared-data)
- [Async: do not block the event loop](#async-do-not-block-the-event-loop)
- [Practical rules](#practical-rules)


---

Concurrency decouples *what* gets done from *when* it gets done, and that decoupling can improve both
throughput and structure. It also introduces a whole class of bugs that appear once a month, in production,
and never in your test suite.

### Know which problem you have

Python offers three tools, and picking the wrong one is the most common mistake:

| Workload | Tool | Why |
| --- | --- | --- |
| **I/O bound** (network, disk, database) | `asyncio` or `threading` | The task is waiting, not computing; the GIL is released during I/O |
| **CPU bound** (parsing, maths, image work) | `multiprocessing` or `concurrent.futures.ProcessPoolExecutor` | Threads cannot use more than one core for Python bytecode |
| **Neither is fast enough** | A queue and separate workers (Celery, RQ, a message broker) | The unit of concurrency is a process on another machine |

Reaching for threads to speed up a CPU-bound loop makes it *slower*, because the threads fight over the GIL.

### Keep concurrency code separate

Concurrency is a responsibility of its own (see the
[Single Responsibility Principle](single-responsibility.md#single-responsibility-principle)). Do not scatter locks and thread
creation through your business logic.

**Bad** :angry:

```python
class ReportBuilder:
    def build(self, accounts: list[Account]) -> Report:
        results, threads = [], []
        lock = threading.Lock()

        def work(account: Account) -> None:
            value = self._summarise(account)      # business logic
            with lock:                            # ...tangled with threading
                results.append(value)

        for account in accounts:
            thread = threading.Thread(target=work, args=(account,))
            thread.start()
            threads.append(thread)
        for thread in threads:
            thread.join()
        return Report(results)
```

**Good** :smiley:

```python
class ReportBuilder:
    def summarise(self, account: Account) -> Summary:      # pure, single-threaded, testable
        ...


def build_report(accounts: list[Account], builder: ReportBuilder) -> Report:
    with ThreadPoolExecutor(max_workers=8) as pool:        # concurrency lives here, alone
        summaries = list(pool.map(builder.summarise, accounts))
    return Report(summaries)
```

`summarise` can now be tested and reasoned about without a single thought about threads, and the concurrency
strategy can be swapped for a process pool by changing one line.

### Limit the scope of shared data

Every piece of mutable data shared between threads is a place a race condition can live. Two defences, in
order of preference:

**1. Do not share.** Give each task its own data and combine the results at the end. Pure functions
(see [Pure Functions](functions.md#pure-functions)) are automatically thread safe.

**2. If you must share, guard it — and keep the critical section tiny.**

**Bad** :angry:

```python
balance = 0

def deposit(amount: int) -> None:
    global balance
    balance = balance + amount      # read, add, write: not atomic
```

Run that in ten threads and the final balance will be wrong, because another thread can run between the read
and the write.

**Good** :smiley:

```python
from threading import Lock


class Balance:
    def __init__(self) -> None:
        self._lock = Lock()
        self._amount = 0

    def deposit(self, amount: int) -> None:
        with self._lock:            # the lock is owned by the data it protects
            self._amount += amount

    @property
    def amount(self) -> int:
        with self._lock:
            return self._amount
```

Better still, let a `queue.Queue` do the synchronising for you — it is thread-safe by design, and a pipeline
of queues has no locks to forget.

### Async: do not block the event loop

With `asyncio`, a single blocking call stalls every other task in the process.

**Bad** :angry:

```python
async def fetch_all(urls: list[str]) -> list[str]:
    return [requests.get(url).text for url in urls]     # blocking, and sequential
```

**Good** :smiley:

```python
import asyncio
import httpx


async def fetch_all(urls: list[str]) -> list[str]:
    async with httpx.AsyncClient() as client:
        responses = await asyncio.gather(*(client.get(url) for url in urls))
    return [response.text for response in responses]
```

If you genuinely must call blocking code, push it off the loop:

```python
result = await asyncio.to_thread(legacy_blocking_call, argument)
```

### Practical rules

- **Write correct single-threaded code first,** and only then make it concurrent. Debugging a design flaw
  and a race at the same time is misery.
- **Never assume a test failure is a fluke.** "It passes if I run it again" is the signature of a race
  condition, not of a bad machine. Do not mark it flaky; find it.
- **Make your threaded code tunable and runnable on more threads than you have cores.** Bugs hide at low
  concurrency.
- **Acquire locks in a consistent order** everywhere, or you will deadlock.
- **Prefer immutable objects** across thread boundaries; `@dataclass(frozen=True)` costs nothing.
- **Use `concurrent.futures` before `threading`.** The pool handles lifecycle, exceptions and results;
  raw threads make you handle all three by hand.

**[⬆ back to top](#table-of-contents)**

