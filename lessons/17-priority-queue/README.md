# صف‌های اولویت‌دار (PriorityQueue) و زمان‌بندی کارها

گاهی وقت‌ها همه‌ی کارها اهمیت یکسانی ندارن. بعضی باید سریع‌تر انجام بشن، بعضی می‌تونن منتظر بمونن. مثلاً تو یه سیستم دانلود، ممکنه بخوای فایل‌های کوچیک یا حیاتی زودتر دانلود شن. اینجاست که صف‌های اولویت‌دار به کمک میان.

---

## مفهوم کلی صف اولویت‌دار

صف معمولی یعنی هر کاری که زودتر وارد صف شده، زودتر هم اجرا می‌شه. ولی صف اولویت‌دار بر اساس **درجه‌ی اهمیت** تصمیم می‌گیره کدوم تسک زودتر اجرا بشه.

تو `asyncio` یه کلاس آماده برای این کار وجود داره به اسم `PriorityQueue`. این صف دقیقاً مثل `Queue` معمولی عمل می‌کنه، فقط یه تفاوت داره: هر آیتم یه عدد اولویت داره. عدد کوچیک‌تر یعنی مهم‌تر.

---

## یه مثال ساده

فرض کن می‌خوای یه سیستم پردازش سفارش بنویسی که سفارش‌های VIP رو سریع‌تر رسیدگی کنه:

```python
import asyncio

async def worker(queue):
    while True:
        priority, order = await queue.get()
        print(f"Processing order: {order} (priority: {priority})")
        await asyncio.sleep(1)
        queue.task_done()

async def main():
    queue = asyncio.PriorityQueue()

    await queue.put((3, 'regular order #1'))
    await queue.put((1, 'VIP order'))
    await queue.put((2, 'regular order #2'))

    task = asyncio.create_task(worker(queue))

    await queue.join()
    task.cancel()

asyncio.run(main())
```

تو این مثال، با اینکه سفارش VIP آخرین موردیه که وارد صف شده، ولی چون اولویتش عدد ۱ هست، زودتر از بقیه پردازش می‌شه.

---

## زمان‌بندی کارها با صف اولویت‌دار

یه استفاده‌ی جالب دیگه از صف اولویت‌دار، زمان‌بندی اجرای کارهاست. مثلاً بخوای یه تسک در آینده اجرا بشه، می‌تونی از «زمان اجرای مورد نظر» به عنوان اولویت استفاده کنی.

```python
import asyncio, time

async def scheduler(queue):
    while True:
        execute_at, task_name = await queue.get()
        now = time.time()
        delay = execute_at - now
        if delay > 0:
            await asyncio.sleep(delay)
        print(f"Running task: {task_name}")
        queue.task_done()

async def main():
    queue = asyncio.PriorityQueue()
    now = time.time()

    await queue.put((now + 5, 'Send email'))
    await queue.put((now + 2, 'Clean cache'))
    await queue.put((now + 8, 'Backup database'))

    worker = asyncio.create_task(scheduler(queue))
    await queue.join()
    worker.cancel()

asyncio.run(main())
```

اینجا ترتیب اجرای کارها بر اساس زمانی‌ه که باید اجرا بشن. در واقع از `PriorityQueue` برای ساخت یه «زمان‌بند ساده» استفاده کردیم.

---

## چند نکته کاربردی

* همیشه عدد کوچیک‌تر یعنی اولویت بالاتر. اگه خواستی برعکسش باشه (مثلاً عدد بزرگ‌تر مهم‌تر باشه)، می‌تونی عددها رو منفی بذاری.
* آیتم‌های صف باید قابل مقایسه باشن، یعنی Python بتونه ترتیبشون رو بفهمه.
* برای ساخت ساختارهای پیچیده‌تر (مثلاً اولویت و زمان با هم)، می‌تونی یه tuple سه‌تایی بذاری، مثل `(priority, timestamp, task)` تا ترتیب قابل پیش‌بینی‌تر بشه.

---

## جمع‌بندی

صف‌های اولویت‌دار ابزار فوق‌العاده‌ای هستن برای کنترل جریان کارها. وقتی چندین تسک داری که همه‌شون مهم نیستن، با این روش می‌تونی اولویت بدهی، صف‌ها رو منظم‌تر کنی و حتی یه سیستم زمان‌بندی ساده بسازی.

---
