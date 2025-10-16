# مدیریت هم‌زمانی با Queue، Lock و Semaphore در asyncio

تا اینجای دوره دیدیم که چطور چند کار رو هم‌زمان اجرا کنیم، خطاها رو کنترل کنیم و ساختار برنامه‌هامون رو منظم‌تر بسازیم. اما یه سوال مهم هنوز باقی مونده:

وقتی چند coroutine هم‌زمان دارن روی یه منبع مشترک کار می‌کنن، چطور باید بینشون هماهنگی برقرار کنیم؟

اینجا پای ابزارهایی مثل **Lock**، **Semaphore** و **Queue** وسط میاد.

---

## قفل (Lock)

گاهی وقتا چند coroutine ممکنه هم‌زمان بخوان روی یه منبع مشترک (مثلاً یه فایل یا یه لیست) بنویسن. اگه اجازه بدیم همه با هم بنویسن، نتیجه می‌تونه کاملاً غیرقابل‌پیش‌بینی بشه.

برای این موقعیت‌ها از `asyncio.Lock` استفاده می‌کنیم.

```python
import asyncio

counter = 0
lock = asyncio.Lock()

async def increase(name):
    global counter
    async with lock:
        print(f"{name} entered the critical section")
        temp = counter
        await asyncio.sleep(0.1)
        counter = temp + 1
        print(f"{name} changed counter to {counter}")

async def main():
    await asyncio.gather(
        increase("Task-1"),
        increase("Task-2"),
        increase("Task-3"),
    )

asyncio.run(main())
```

در این مثال، `async with lock` باعث میشه فقط یه coroutine در لحظه به بخش بحرانی دسترسی داشته باشه.

---

## صف (Queue)

صف یکی از پرکاربردترین ابزارهای هم‌زمانی توی asyncio هست. فرض کن می‌خوای یه سری داده تولید کنی و بعد به‌صورت هم‌زمان اون داده‌ها رو پردازش کنی.

با `asyncio.Queue` می‌تونی این ارتباط بین "تولیدکننده" و "مصرف‌کننده" رو مدیریت کنی.

```python
import asyncio

async def producer(queue):
    for i in range(5):
        print(f"Producing item {i}")
        await queue.put(i)
        await asyncio.sleep(0.2)
    await queue.put(None)  # end signal

async def consumer(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"Consuming item {item}")
        await asyncio.sleep(0.3)

async def main():
    queue = asyncio.Queue()
    await asyncio.gather(producer(queue), consumer(queue))

asyncio.run(main())
```

اینجا صف به‌صورت خودکار وظیفه هماهنگی بین تولید و مصرف داده رو انجام میده.

---

## شمارنده هم‌زمانی (Semaphore)

قفل فقط اجازه میده یه coroutine وارد بخش بحرانی بشه، اما اگه بخوای مثلاً **حداکثر سه coroutine** هم‌زمان وارد بشن، از `asyncio.Semaphore` استفاده می‌کنیم.

```python
import asyncio

semaphore = asyncio.Semaphore(3)

async def download_file(i):
    async with semaphore:
        print(f"Starting download for file {i}")
        await asyncio.sleep(1)
        print(f"Finished downloading file {i}")

async def main():
    await asyncio.gather(*[download_file(i) for i in range(10)])

asyncio.run(main())
```

در این مثال، در هر لحظه فقط سه تا coroutine می‌تونن هم‌زمان در حال دانلود باشن.

---

## نکته‌های پایانی

* همیشه قبل از استفاده از Lock یا Semaphore فکر کن واقعاً لازمش داری یا نه. گاهی async I/O خودش به اندازه کافی امنه.
* اگه فقط می‌خوای کارها رو صف‌بندی کنی یا یه pipeline ساده بسازی، Queue انتخاب بهتریه.
* استفاده بیش‌ازحد از قفل‌ها باعث کاهش کارایی برنامه‌ات میشه.

---
