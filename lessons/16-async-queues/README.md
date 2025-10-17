# صف‌های همزمان (Async Queues) و الگوی Producer–Consumer

تا حالا بیشتر روی اجرای همزمان تسک‌ها تمرکز کردیم، ولی وقتشه یاد بگیریم چطور بین چند تسک هماهنگی ایجاد کنیم. گاهی چند coroutine باید با هم کار کنن — یکی داده تولید کنه (producer) و دیگری اون داده رو مصرف کنه (consumer). برای این کار، `asyncio.Queue` ابزار جادویی ماست!

---

## چرا به صف نیاز داریم؟

تصور کن یه سایت دانلود داری که آدرس لینک‌ها از یه منبع جمع‌آوری میشه و چند worker باید اون لینک‌ها رو دانلود کنن. اگه همه‌ی workerها مستقیم سراغ یه لیست مشترک برن، ممکنه با مشکل تداخل یا race condition مواجه بشی.

اینجاست که صف به درد می‌خوره. صف مثل یه خط انتظار رفتار می‌کنه — producer داده رو می‌ذاره داخل صف، و consumerها یکی‌یکی از صف برمی‌دارن.

---

## مثال ساده از کار با صف در asyncio

```python
import asyncio

async def producer(queue):
    for i in range(5):
        await asyncio.sleep(1)
        item = f"item-{i}"
        await queue.put(item)
        print(f"Produced {item}")
    print("Producer done!")

async def consumer(queue):
    while True:
        item = await queue.get()
        print(f"Consumed {item}")
        await asyncio.sleep(2)
        queue.task_done()

async def main():
    queue = asyncio.Queue()

    producers = [asyncio.create_task(producer(queue))]
    consumers = [asyncio.create_task(consumer(queue)) for _ in range(2)]

    await asyncio.gather(*producers)
    await queue.join()  # wait until all items are processed

    for c in consumers:
        c.cancel()

asyncio.run(main())
```

در این مثال، یه producer داریم که هر ثانیه یه آیتم تولید می‌کنه و می‌ذاره داخل صف، و دو تا consumer که هرکدوم جداگانه آیتم‌ها رو از صف برمی‌دارن و پردازش می‌کنن. در نهایت، بعد از اینکه همه‌ی آیتم‌ها مصرف شدن، consumerها لغو می‌شن.

---

## توضیح قدم‌به‌قدم

* متد `put()` آیتم رو وارد صف می‌کنه، ولی اگه صف پر باشه (limit داشته باشه)، منتظر می‌مونه تا جا باز بشه.
* متد `get()` آیتم بعدی رو از صف برمی‌داره، و اگه صف خالی باشه، منتظر می‌مونه تا چیزی وارد بشه.
* متد `task_done()` به صف می‌گه که آیتم مربوطه پردازش شده.
* متد `join()` منتظر می‌مونه تا همه‌ی آیتم‌ها مصرف و تأیید بشن.

---

## کنترل سرعت و محدود کردن ظرفیت صف

می‌تونیم برای صف یه اندازه‌ی حداکثر مشخص کنیم تا از تولید بیش از حد داده جلوگیری بشه:

```python
queue = asyncio.Queue(maxsize=10)
```

با این کار، اگه producer سریع‌تر از consumer کار کنه، متوقف میشه تا فضای خالی در صف ایجاد بشه. این رفتار باعث میشه حافظه کنترل‌شده و برنامه پایدارتر بمونه.

---

## مثال واقعی‌تر: شبیه‌سازی یک سیستم دانلودر

فرض کن چند URL داری که باید دانلود بشن، ولی فقط ۳ تا worker می‌تونن هم‌زمان کار کنن:

```python
import asyncio
import random

async def producer(queue, urls):
    for url in urls:
        await queue.put(url)
        print(f"Added {url} to queue")

async def consumer(queue, worker_id):
    while True:
        url = await queue.get()
        print(f"Worker-{worker_id} downloading {url}")
        await asyncio.sleep(random.uniform(1, 3))  # simulate network delay
        print(f"Worker-{worker_id} finished {url}")
        queue.task_done()

async def main():
    urls = [f"https://example.com/file{i}.txt" for i in range(10)]
    queue = asyncio.Queue()

    prod = asyncio.create_task(producer(queue, urls))
    consumers = [asyncio.create_task(consumer(queue, i)) for i in range(3)]

    await prod
    await queue.join()

    for c in consumers:
        c.cancel()

asyncio.run(main())
```

اینجا یه producer داریم که لیست URLها رو به صف اضافه می‌کنه، و سه consumer که هم‌زمان در حال دانلود هستن. این دقیقاً الگویی‌ه که در پروژه‌های واقعی مثل crawler یا downloader استفاده میشه.

---

## نکات مهم

* صف‌ها بهترین راه برای هماهنگی بین coroutineها هستن.
* استفاده از صف باعث میشه از قفل‌های پیچیده (`Lock`) بی‌نیاز بشی.
* همیشه بعد از `get()`، وقتی کار تموم شد، `task_done()` رو فراموش نکن.
* اگه برنامه‌ت چند worker داره، حتماً بعد از تموم‌شدن کارها اونا رو `cancel()` کن.

---

## جمع‌بندی

صف‌های async یکی از مهم‌ترین ابزارهای همزمانی در asyncio هستن. با اون‌ها می‌تونی داده‌هات رو بین تسک‌ها جابه‌جا کنی، سرعت تولید و مصرف رو کنترل کنی، و بدون درگیر شدن با قفل‌ها، برنامه‌ای تمیز و قابل‌اعتماد بسازی.

در درس بعدی، می‌تونیم یه قدم جلوتر بریم و این الگو رو در قالب یه سیستم کامل مثل crawler واقعی ترکیب کنیم.

---
