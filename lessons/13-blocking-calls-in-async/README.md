# اجرای کدهای بلاک‌شونده در برنامه‌های Async

خیلی خوب. حالا که با مفاهیم اصلی asyncio و ابزارهای هم‌زمانی آشنا شدی، وقتشه بفهمیم وقتی باید با کتابخانه‌ها یا تابع‌های **بلاک‌شونده (blocking)** کار کنیم، چطور جلوی قفل شدن event loop رو بگیریم. این درس مفصل دربارهٔ راهکارها، نکات، و الگوهای عملی ترکیب کد هم‌زمان و غیرهم‌زمان است.

---

## چرا باید به این موضوع اهمیت بدیم؟

اگه تابعی بلاک‌شونده اجرا بشه و اون رو مستقیماً داخل یک coroutine فراخوانی کنی، event loop کاملًا متوقف میشه و بقیه coroutineها منتظر می‌مونن. نتیجهٔ نهایی اینه که مزیت اصلی async از بین میره.

برای مثال، کتابخانهٔ معروف `requests` بلاک‌شونده است. اگه مستقیم ازش در یک coroutine استفاده کنی، تمام مزایای async از بین میره.

---

## ابزارهایی که داریم

‏- `asyncio.get_running_loop().run_in_executor(...)` — اجرای تابع blocking در thread یا process جداگانه.

‏- `asyncio.to_thread(...)` — راه‌حل کوتاه‌تر برای اجرای تابع در ThreadPool (Python 3.9+).

‏- `concurrent.futures.ThreadPoolExecutor` — کنترل تعداد threads و مدیریت آن‌ها.

‏- `concurrent.futures.ProcessPoolExecutor` — برای کارهای CPU-bound که نیاز به دور زدن GIL دارند.

---

## الگوی ساده: استفاده از `run_in_executor`

در ساده‌ترین حالت می‌تونیم یک تابع بلاک‌شونده رو داخل یک thread اجرا کنیم و از await برای انتظار نتایج استفاده کنیم:

```python
import asyncio
import requests

def blocking_http_get(url):
    resp = requests.get(url)
    return resp.text

async def fetch(url):
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, blocking_http_get, url)
    return result

async def main():
    urls = ["https://example.com" for _ in range(5)]
    results = await asyncio.gather(*(fetch(u) for u in urls))
    print("fetched", len(results))

if __name__ == "__main__":
    asyncio.run(main())
```

در اینجا `None` به معنی استفاده از ThreadPoolExecutor پیش‌فرض است. هر کدام از فراخوانی‌ها در thread جدا اجرا میشه و event loop بلاک نمیشه.

---

## استفاده از `asyncio.to_thread` (پیشنهاد مدرن)

برای اجرای توابع سادهٔ بلاک‌شونده در thread، `to_thread` کوتاه‌تر و خواناتر است:

```python
import asyncio
import requests

def blocking_http_get(url):
    resp = requests.get(url)
    return resp.text

async def fetch(url):
    result = await asyncio.to_thread(blocking_http_get, url)
    return result

async def main():
    urls = ["https://example.com" for _ in range(5)]
    results = await asyncio.gather(*(fetch(u) for u in urls))
    print("fetched", len(results))

asyncio.run(main())
```

این راهکار برای بیشتر موارد شبکه‌ای که فقط کتابخانهٔ sync در دسترسه مناسب و ساده‌ست.

---

## وقتی تعداد threadها مهم میشه — ThreadPoolExecutor

برای کنترل بهتر تعداد threadها و مدیریت منابع، می‌تونی یک `ThreadPoolExecutor` بسازی و اون رو به `run_in_executor` پاس بدی:

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import requests

executor = ThreadPoolExecutor(max_workers=10)

def blocking_http_get(url):
    return requests.get(url).text

async def fetch(url):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(executor, blocking_http_get, url)

async def main():
    urls = ["https://example.com" for _ in range(50)]
    results = await asyncio.gather(*(fetch(u) for u in urls))
    print("fetched", len(results))

if __name__ == "__main__":
    asyncio.run(main())
    executor.shutdown()
```

قابلیت `max_workers` کمک می‌کنه هم بار روی سرور و هم مصرف منابع محلی رو کنترل کنی.

---

## کارهای CPU-bound و ProcessPoolExecutor

برای کارهایی که نیاز به پردازش سنگین CPU دارن (مثل فشرده‌سازی، رمزنگاری، پردازش تصویر)، استفاده از thread‌ کفایت نمی‌کنه به‌خاطر GIL. در این موارد `ProcessPoolExecutor` مناسب‌تره:

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def heavy_compute(x):
    # محاسبه سنگین فرضی
    s = 0
    for i in range(10_000_000):
        s += (i * x) % 7
    return s

async def main():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        tasks = [loop.run_in_executor(pool, heavy_compute, i) for i in range(4)]
        results = await asyncio.gather(*tasks)
        print(results)

if __name__ == "__main__":
    asyncio.run(main())
```

در نظر داشته باش که ارسال داده بین پروسس‌ها هزینه داره؛ پس برایِ کارهای کوچک این راهکار می‌تونه کم‌سود باشه.

---

## نکات مهم در مورد cancellation و استثناها

‏- وقتی `run_in_executor` یا `to_thread` رو await می‌کنی و سپس task کانسل میشه، انتظار داری که اجرای زیرین هم متوقف بشه؛ اما حقیقت اینه که توی thread یا process اجرا شده، تابع بلاک‌شونده همچنان اجرا میشه. کانسلیشن فقط باعث میشه await برگشت بده `CancelledError`، ولی تابع در پس‌زمینه ادامه خواهد داشت مگر خودِ تابع بشه متوقف.

‏- برای توابع حساس به کانسلیشن می‌تونی مکانیسم‌های بیرونی (مانند پرچم‌های مشترک یا قرار دادن timeout در خودِ تابع) استفاده کنی.

‏- استثناهای داخلی تابع بلاک‌شونده به شکل استثناهای awaitable بالا میاد؛ بنابراین باید مثل همیشه با try/except مدیریت‌شون کنی.

---

## مثال عملی: ترکیب Queue و ThreadPool برای پردازش I/O بلاک‌شونده

این الگو وقتی مفیده که چند producer async داری و پردازش هر آیتم با یک کتابخانهٔ sync انجام میشه. از Queue برای همگرا کردن و از ThreadPoolExecutor برای اجرای تابع sync استفاده می‌کنیم.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import requests

executor = ThreadPoolExecutor(max_workers=5)

def blocking_process(url):
    r = requests.get(url)
    return len(r.content)

async def producer(queue):
    for i in range(20):
        await queue.put(f"https://example.com?i={i}")
    for _ in range(5):
        await queue.put(None)

async def consumer(queue):
    loop = asyncio.get_running_loop()
    while True:
        url = await queue.get()
        if url is None:
            break
        size = await loop.run_in_executor(executor, blocking_process, url)
        print("processed", url, "size", size)

async def main():
    queue = asyncio.Queue()
    producers = [producer(queue)]
    consumers = [consumer(queue) for _ in range(5)]
    await asyncio.gather(*(producers + consumers))

if __name__ == "__main__":
    asyncio.run(main())
    executor.shutdown()
```

---

## توصیه‌ها و بهترین شیوه‌ها

* اگر امکان‌پذیره، به‌جای wrap کردن نسخه‌های sync، از نسخهٔ async کتابخانه‌ها استفاده کن (مثلاً از `aiohttp` به‌جای `requests`).
* برای کارهای کوتاه I/O که زیاد هم تکرار می‌شن، `to_thread` ساده و خوبه.
* برای پردازش‌های سنگین CPU از `ProcessPoolExecutor` استفاده کن.
* همواره محدودیت تعداد threadها/processها رو در نظر بگیر تا منابع سیستمی رو خالی نکنی.
* به رفتار کانسلیشن توجه کن و اگر لازم شد، خود تابع sync طوری طراحی کن که قابل توقف باشه.

---
