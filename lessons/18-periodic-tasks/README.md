# اجرای دوره‌ای تسک‌ها و زمان‌بندی (Periodic Tasks & Scheduling)

گاهی در برنامه‌هامون نیاز داریم کاری رو **به صورت مکرر و در فواصل زمانی مشخص** انجام بدیم. مثل جمع‌آوری دیتا هر ۵ ثانیه، ارسال heartbeat به سرور، یا بررسی وضعیت یک سرویس. اینجا مفهوم **periodic tasks** و زمان‌بندی تسک‌ها اهمیت پیدا می‌کنه.

در asyncio، چند روش داریم برای اجرای دوره‌ای تسک‌ها و کنترل دقیق زمان‌بندی، که تو این درس مفصل بررسی می‌کنیم.

---

## مفهوم Periodic Task

یک تسک دوره‌ای، یه coroutine هست که بارها اجرا میشه و بین اجرای هر بار، یک فاصله زمانی مشخص داره. تفاوت اصلی با loop معمولی اینه که این فاصله دقیقاً قابل کنترل هست و می‌تونیم رفتارهای پیچیده‌تری بسازیم.

مثال ساده: اجرای یک coroutine هر ۲ ثانیه

```python
import asyncio

async def periodic_task():
    while True:
        print("Running periodic task")
        await asyncio.sleep(2)

asyncio.run(periodic_task())
```

در این مثال، تسک هر ۲ ثانیه اجرا میشه، ولی هنوز کنترل دقیقی روی زمان شروع و پایان هر iteration نداریم.

---

## حفظ فاصله دقیق بین اجراها

گاهی مهمه که تسک دقیقا هر n ثانیه اجرا بشه، بدون اینکه زمان اجرای خودش، فاصله رو جابه‌جا کنه. برای این کار می‌تونیم زمان بعدی اجرا رو محاسبه کنیم:

```python
import asyncio, time

async def precise_periodic_task(interval):
    next_time = time.time() + interval
    while True:
        print(f"Task running at {time.strftime('%X')}")
        await asyncio.sleep(max(0, next_time - time.time()))
        next_time += interval

asyncio.run(precise_periodic_task(3))
```

با این روش، حتی اگه اجرای task طول بکشه، فاصله‌ی شروع هر iteration ثابت باقی می‌مونه.

---

## اجرای چند تسک دوره‌ای همزمان

در برنامه‌های واقعی، ممکنه چند تسک دوره‌ای داشته باشیم که هم‌زمان اجرا می‌شن. با `asyncio.create_task` می‌تونیم هر تسک رو مستقل اجرا کنیم:

```python
import asyncio, time

async def task(name, interval):
    next_time = time.time() + interval
    while True:
        print(f"{name} running at {time.strftime('%X')}")
        await asyncio.sleep(max(0, next_time - time.time()))
        next_time += interval

async def main():
    tasks = [
        asyncio.create_task(task("Task-A", 2)),
        asyncio.create_task(task("Task-B", 3))
    ]
    await asyncio.sleep(10)  # run for 10 seconds
    for t in tasks:
        t.cancel()

asyncio.run(main())
```

در این مثال، Task-A هر ۲ ثانیه و Task-B هر ۳ ثانیه اجرا می‌شه، بدون اینکه روی هم تاثیر بذارن.

---

## ترکیب با صف‌ها و زمان‌بندی پیشرفته

می‌تونیم تسک‌های دوره‌ای رو با صف‌ها ترکیب کنیم و یه **scheduler ساده** بسازیم. فرض کن می‌خوای چند worker رو به صورت دوره‌ای فعال کنی تا آیتم‌ها از صف برداشته و پردازش بشن:

```python
import asyncio, time

async def worker(queue, name):
    while True:
        item = await queue.get()
        print(f"{name} processing {item} at {time.strftime('%X')}")
        await asyncio.sleep(1)
        queue.task_done()

async def scheduler(queue, interval, name):
    next_time = time.time() + interval
    count = 0
    while True:
        await asyncio.sleep(max(0, next_time - time.time()))
        count += 1
        await queue.put(f"job-{count}")
        print(f"{name} added job-{count} to queue")
        next_time += interval

async def main():
    queue = asyncio.Queue()

    workers = [asyncio.create_task(worker(queue, f"Worker-{i}")) for i in range(2)]
    schedulers = [asyncio.create_task(scheduler(queue, i+2, f"Scheduler-{i}")) for i in range(2)]

    await asyncio.sleep(12)  # run for 12 seconds

    for t in workers + schedulers:
        t.cancel()

asyncio.run(main())
```

اینجا هر scheduler آیتم‌های جدید رو به صف اضافه می‌کنه و workerها همزمان اون‌ها رو پردازش می‌کنن. این ترکیب، پایه‌ی خیلی از سیستم‌های واقعی مثل crawlerها، downloaderها و job queueهاست.

---

## نکات کاربردی و حرفه‌ای

* همیشه از روش محاسبه زمان دقیق شروع بعدی استفاده کن تا drift زمانی ایجاد نشه.
* تسک‌های دوره‌ای ممکنه با لغو یا خطا مواجه بشن، حتما `try/except asyncio.CancelledError` و cleanup رو رعایت کن.
* اگه چند تسک دوره‌ای داری، بهتره مستقل باشن و با create_task اجرا بشن تا همزمانی کامل داشته باشیم.
* می‌تونی interval‌ها رو پویا تغییر بدی، مثلا براساس load سیستم یا تعداد آیتم‌های صف.
* ترکیب با PriorityQueue می‌تونه اولویت و زمان‌بندی رو با هم مدیریت کنه.

---

## جمع‌بندی

اجرای دوره‌ای تسک‌ها و زمان‌بندی دقیقشون بخش مهمی از سیستم‌های async پیشرفته است.
با تکنیک‌های این درس می‌تونی:

* تسک‌ها رو در فواصل ثابت اجرا کنی
* چند تسک دوره‌ای همزمان مدیریت کنی
* با صف‌ها و workerها ترکیب کنی
* سیستم‌های واقعی مثل crawler، downloader و job scheduler بسازی

---
