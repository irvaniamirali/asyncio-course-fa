## درس بیست‌ویکم: الگوهای پیشرفته هم‌زمانی — Fan-in / Fan-out، Pipeline، Queue و Event

تا اینجا یاد گرفتیم چطور تسک‌ها رو به‌صورت ناهمگام اجرا کنیم، داده‌ها رو بخونیم و بنویسیم و با شبکه کار کنیم. اما دنیای واقعی خیلی وقت‌ها پیچیده‌تره. وقتی چندین منبع داده داری، چندین کارگر (worker) برای پردازش، و نیاز به هماهنگی بینشون، اون‌وقته که باید سراغ *الگوهای پیشرفته هم‌زمانی* بریم.

در این درس قراره چند تا از مهم‌ترین الگوهای حرفه‌ای دنیای async رو یاد بگیریم:

* الگوی **fan-in / fan-out** برای ترکیب و توزیع کارها
* ساخت **pipeline** یا زنجیره پردازش داده
* استفاده هم‌زمان از **Queue**، **PriorityQueue** و **Event** برای کنترل جریان داده

---

### مفهوم fan-out

وقتی یه کار بزرگ داری که میشه اون رو بین چند وظیفه (task) تقسیم کرد، از fan-out استفاده می‌کنی. مثلاً فرض کن می‌خوای ۱۰۰۰ آدرس وب رو بخونی و تحلیل کنی. به‌جای اینکه یکی‌یکی انجامش بدی، می‌تونی چند کارگر هم‌زمان بسازی که هر کدوم بخشی از کار رو انجام بدن.

```python
import asyncio
import random

async def worker(name, queue):
    while True:
        url = await queue.get()
        print(f"{name} started {url}")
        await asyncio.sleep(random.uniform(0.5, 2))  # simulate I/O
        print(f"{name} finished {url}")
        queue.task_done()

async def main():
    queue = asyncio.Queue()

    for i in range(10):
        await queue.put(f"https://example.com/page{i}")

    workers = [asyncio.create_task(worker(f"Worker-{i}", queue)) for i in range(3)]

    await queue.join()

    for w in workers:
        w.cancel()

asyncio.run(main())
```

در این مثال، ۱۰ لینک بین ۳ کارگر تقسیم می‌شن. این یعنی *fan-out*، چون یه منبع داده، بین چند تسک پخش شده.

---

### مفهوم fan-in

در fan-in، چندین منبع مختلف داده وجود دارن که باید نتایجشون رو به یک نقطه جمع کنیم. مثلاً فرض کن چند کارگر داری که داده تولید می‌کنن و یه تابع باید همه نتایج رو بخونه.

```python
import asyncio
import random

async def producer(name, queue):
    for i in range(3):
        await asyncio.sleep(random.uniform(0.5, 1.5))
        item = f"{name}-data-{i}"
        await queue.put(item)
        print(f"{name} produced {item}")

async def consumer(queue):
    while True:
        item = await queue.get()
        print(f"Consumer got {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    producers = [asyncio.create_task(producer(f"Producer-{i}", queue)) for i in range(3)]
    consumer_task = asyncio.create_task(consumer(queue))

    await asyncio.gather(*producers)
    await queue.join()
    consumer_task.cancel()

asyncio.run(main())
```

در اینجا سه تولیدکننده داده (fan-in) دارن داده تولید می‌کنن و یه مصرف‌کننده اون‌ها رو جمع می‌کنه.

---

### ساخت pipeline (زنجیره پردازش)

الگوی pipeline برای زمانی مناسبه که داده‌هامون باید در چند مرحله پردازش بشن. هر مرحله خروجی مرحله‌ی قبل رو می‌گیره و بعد از پردازش، اون رو به مرحله‌ی بعدی می‌فرسته.

مثلاً فرض کن می‌خوای داده‌ها رو از یه API بگیری، پردازش کنی و بعد ذخیره‌شون کنی.

```python
import asyncio
import random

async def fetch_data(fetch_q):
    for i in range(5):
        data = f"raw-data-{i}"
        await asyncio.sleep(random.uniform(0.5, 1))
        await fetch_q.put(data)
        print(f"Fetched: {data}")
    await fetch_q.put(None)

async def process_data(fetch_q, process_q):
    while True:
        data = await fetch_q.get()
        if data is None:
            await process_q.put(None)
            break
        result = data.upper()
        await asyncio.sleep(random.uniform(0.5, 1))
        await process_q.put(result)
        print(f"Processed: {result}")
        fetch_q.task_done()

async def save_data(process_q):
    while True:
        data = await process_q.get()
        if data is None:
            break
        await asyncio.sleep(random.uniform(0.5, 1))
        print(f"Saved: {data}")
        process_q.task_done()

async def main():
    fetch_q = asyncio.Queue()
    process_q = asyncio.Queue()

    await asyncio.gather(
        fetch_data(fetch_q),
        process_data(fetch_q, process_q),
        save_data(process_q)
    )

asyncio.run(main())
```

اینجا سه مرحله‌ی متوالی داریم: گرفتن داده، پردازش، و ذخیره. خروجی هر مرحله ورودی بعدی میشه. این دقیقاً مفهوم pipeline در دنیای async هست.

---

### استفاده از PriorityQueue

گاهی لازمه بعضی کارها زودتر انجام بشن. مثلاً پیام‌های مهم‌تر یا تسک‌های حیاتی. برای این کار می‌تونیم از `asyncio.PriorityQueue` استفاده کنیم.

```python
import asyncio
import random

async def worker(queue):
    while True:
        priority, task = await queue.get()
        await asyncio.sleep(random.uniform(0.5, 1.5))
        print(f"Processed {task} with priority {priority}")
        queue.task_done()

async def main():
    queue = asyncio.PriorityQueue()

    await queue.put((3, "normal task"))
    await queue.put((1, "urgent task"))
    await queue.put((2, "medium task"))

    asyncio.create_task(worker(queue))
    await queue.join()

asyncio.run(main())
```

در این مثال، کار با اولویت عدد کوچکتر زودتر انجام میشه.

---

### همگام‌سازی با Event

گاهی لازمه چند تسک منتظر وقوع یه رویداد خاص بمونن. مثلاً وقتی داده آماده شد یا پردازش تموم شد. در این حالت از `asyncio.Event` استفاده می‌کنیم.

```python
import asyncio

async def waiter(event):
    print("Waiting for event...")
    await event.wait()
    print("Event triggered!")

async def setter(event):
    await asyncio.sleep(2)
    print("Setting event...")
    event.set()

async def main():
    event = asyncio.Event()
    await asyncio.gather(waiter(event), setter(event))

asyncio.run(main())
```

در اینجا، تسک `waiter` منتظر می‌مونه تا رویداد تنظیم بشه و بعد ادامه میده.

---

### ترکیب Queue و Event

حالا یه مثال واقعی‌تر بسازیم که در اون تسک‌ها داده‌ها رو از صف می‌خونن و وقتی صف خالی شد، با استفاده از Event بقیه رو متوقف می‌کنیم.

```python
import asyncio

async def producer(queue, event):
    for i in range(5):
        await asyncio.sleep(0.5)
        await queue.put(f"task-{i}")
        print(f"Produced task-{i}")
    event.set()

async def consumer(queue, event):
    while True:
        if queue.empty() and event.is_set():
            print("No more tasks. Consumer stopping.")
            break
        try:
            item = await asyncio.wait_for(queue.get(), timeout=1)
            print(f"Consumed {item}")
            queue.task_done()
        except asyncio.TimeoutError:
            pass

async def main():
    queue = asyncio.Queue()
    event = asyncio.Event()
    await asyncio.gather(producer(queue, event), consumer(queue, event))

asyncio.run(main())
```

اینجا Event به‌عنوان سیگنال پایان داده عمل می‌کنه و به مصرف‌کننده اطلاع میده که تولیدکننده تموم شده.

---

### جمع‌بندی

در این درس با چند الگوی مهم در برنامه‌نویسی ناهمگام آشنا شدیم:

* الگوی **fan-out** برای تقسیم کار بین چند تسک
* الگوی **fan-in** برای جمع‌آوری خروجی از چند منبع
* الگوی **pipeline** برای ساخت زنجیره پردازش داده
* صف‌های مختلف مثل **Queue** و **PriorityQueue** برای مدیریت جریان کار
* و **Event** برای همگام‌سازی بین تسک‌ها
---
