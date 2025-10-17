# طراحی سیستم‌های پیشرفته background tasks در asyncio

خب، بیایید راحت و دوستانه درباره‌ی background tasks صحبت کنیم. وقتی پروژه‌ای داریم که تعداد زیادی پردازش همزمان داره یا کارهای زمان‌بر و حساسی انجام می‌ده، یک background task ساده واقعا کافی نیست. تو این درس می‌خوایم با هم یاد بگیریم چطور یک سیستم پیشرفته بسازیم که tasks تو پس‌زمینه درست و بهینه اجرا بشن و چطور همزمانی، صف‌ها، اولویت‌ها، retry، زمان‌بندی و مدیریت خطاها رو کنترل کنیم.

## چرا background tasks پیشرفته لازم داریم؟

فرض کن یک سیستم داریم که کاربر فایل آپلود می‌کنه و باید پردازش ویدئو روش انجام بشه. اگر همه‌ی این کارها رو مستقیم و بدون کنترل اجرا کنیم:

* سرور خیلی سریع منابعش پر می‌شه.
* احتمال خطا یا crash خیلی بالاست.
* تجربه کاربری خیلی کند می‌شه و سیستم قابل اعتماد نیست.

راه حل؟ طراحی یک سیستم background tasks با صف، worker، زمان‌بندی و مدیریت خطا.

## الگوی پایه: Queue و Worker

ساده‌ترین و در عین حال قدرتمندترین روش استفاده از صف‌ها و workerهاست. فکر کن یه صف داریم و چند worker که از اون صف کار می‌گیرن و پردازش می‌کنن. مثال:

```python
import asyncio
import random

async def worker(queue, name):
    while True:
        task_item = await queue.get()
        try:
            print(f"{name} processing {task_item}")
            await asyncio.sleep(random.uniform(1, 3))
        except Exception as e:
            print(f"{name} encountered an error: {e}")
        finally:
            queue.task_done()

async def main():
    queue = asyncio.Queue()

    for i in range(3):
        asyncio.create_task(worker(queue, f"Worker-{i}"))

    for j in range(10):
        await queue.put(f"Job-{j}")

    await queue.join()

asyncio.run(main())
```

## مدیریت اولویت و retry

حالا فرض کن بعضی کارها مهم‌ترن یا ممکنه خطا داشته باشن و نیاز به retry داشته باشن. می‌تونیم از PriorityQueue و retry استفاده کنیم:

```python
import asyncio
import random

class Job:
    def __init__(self, name, priority=5, retries=3):
        self.name = name
        self.priority = priority
        self.retries = retries

async def worker(queue, name):
    while True:
        priority, job = await queue.get()
        attempt = 0
        while attempt < job.retries:
            try:
                print(f"{name} processing {job.name}, attempt {attempt+1}")
                if random.random() < 0.3:
                    raise ValueError("Simulated error")
                break
            except Exception as e:
                print(f"{job.name} failed: {e}")
                attempt += 1
        queue.task_done()

async def main():
    queue = asyncio.PriorityQueue()

    jobs = [Job(f"Job-{i}", priority=i%3) for i in range(10)]
    for job in jobs:
        await queue.put((job.priority, job))

    for i in range(2):
        asyncio.create_task(worker(queue, f"Worker-{i}"))

    await queue.join()

asyncio.run(main())
```

## زمان‌بندی background tasks

گاهی لازم داریم taskها نه فقط تو پس‌زمینه اجرا بشن، بلکه در زمان مشخص یا دوره‌ای هم اجرا بشن:

```python
import asyncio, time

async def scheduled_task(interval, name):
    next_time = time.time() + interval
    count = 0
    while True:
        await asyncio.sleep(max(0, next_time - time.time()))
        print(f"{name} execution {count} at {time.strftime('%X')}")
        count += 1
        next_time += interval

async def main():
    asyncio.create_task(scheduled_task(3, "Task-A"))
    asyncio.create_task(scheduled_task(5, "Task-B"))

    await asyncio.sleep(15)

asyncio.run(main())
```

## نکات حرفه‌ای

* همیشه exceptionها رو مدیریت کن و منابع task رو درست پاکسازی کن.
* تعداد workerها و حجم صف رو مطابق منابع سرور تنظیم کن.
* ترکیب priority queue و scheduled tasks سیستم‌های پیچیده مثل crawler یا pipeline رو قابل اعتماد می‌کنه.
* برای taskهای طولانی یا وابسته به IO، صف و retry بهترین روشه تا از crash و deadlock جلوگیری بشه.

## جمع‌بندی

با مفاهیم این درس می‌تونی:

* یک سیستم background tasks مقیاس‌پذیر و مقاوم بسازی
* همزمانی taskها رو کنترل کنی
* اولویت، retry و زمان‌بندی رو مدیریت کنی
* نمونه‌های واقعی مثل downloader، پردازش ویدئو، crawler و pipeline بسازی

