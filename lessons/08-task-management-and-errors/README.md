# درس هشتم: مدیریت تسک‌ها و خطاها در asyncio

تا اینجا یاد گرفتیم چطور چند کار رو هم‌زمان با async انجام بدیم. حالا وقتشه یاد بگیریم چطور **تسک‌ها** رو کنترل کنیم، **خطاها** رو مدیریت کنیم، و جلوی **اجرای بی‌رویه یا طولانی‌شدن برنامه** رو بگیریم.

در دنیای واقعی، همیشه همه‌چیز طبق برنامه پیش نمی‌ره — یکی از درخواست‌ها خطا می‌ده، سرور دیر جواب می‌ده، یا شاید بخوای فقط تعداد محدودی از کارها هم‌زمان اجرا بشن. این درس برای درک همین موقعیت‌هاست.

---

## مفهوم مدیریت تسک‌ها

هر بار که از `asyncio.create_task()` استفاده می‌کنی، در واقع داری یه تسک (Task) جدید می‌سازی که به صورت مستقل در حلقهٔ رویداد اجرا می‌شه. اما باید بدونی چطور اون تسک رو **نظارت** و **کنترل** کنی.

---

## مدیریت خطاها در asyncio.gather

وقتی چند تسک رو با `asyncio.gather()` اجرا می‌کنی، اگه یکی از اون‌ها خطا بده، به‌طور پیش‌فرض **کل gather متوقف می‌شه**. اما می‌تونی با `return_exceptions=True` کاری کنی که خطاها به‌صورت مقدار برگردن، نه اینکه برنامه متوقف بشه.

```python
import asyncio

async def risky_task(i):
    if i == 2:
        raise ValueError(f"Task {i} failed!")
    await asyncio.sleep(1)
    print(f"Task {i} finished successfully.")
    return i

async def main():
    tasks = [risky_task(i) for i in range(5)]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    for result in results:
        if isinstance(result, Exception):
            print(f"Got an error: {result}")
        else:
            print(f"Result: {result}")

asyncio.run(main())
```

در این مثال، اگر یکی از تسک‌ها خطا بده، بقیه همچنان اجرا می‌شن و خروجی در نهایت شامل هم نتایج موفق و هم Exceptionها می‌شه.

---

## لغو تسک‌ها (Cancel Tasks)

گاهی لازم می‌شه یه تسک رو در حین اجرا لغو کنی. مثلاً فرض کن کاربر برنامه رو بست یا دیگه نتیجهٔ اون تسک مهم نیست. در این حالت می‌تونی از `task.cancel()` استفاده کنی.

```python
import asyncio

async def long_task():
    try:
        print("Task started.")
        await asyncio.sleep(5)
        print("Task finished.")
    except asyncio.CancelledError:
        print("Task was cancelled!")
        raise

async def main():
    task = asyncio.create_task(long_task())
    await asyncio.sleep(2)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Handled task cancellation.")

asyncio.run(main())
```

در این مثال، تسک بعد از دو ثانیه لغو می‌شه. این موضوع برای جلوگیری از عملیات‌های طولانی یا بی‌فایده بسیار کاربردیه.

---

## محدود کردن زمان اجرا (Timeout)

یکی از سناریوهای متداول در دنیای واقعی، **محدودیت زمانی**ه. فرض کن چند درخواست شبکه داری که ممکنه یکی از سرورها خیلی دیر پاسخ بده. می‌تونی با `asyncio.wait_for()` برای هر تسک زمان مشخص کنی.

```python
import asyncio

async def fetch_data():
    print("Fetching data...")
    await asyncio.sleep(5)
    return "data received"

async def main():
    try:
        result = await asyncio.wait_for(fetch_data(), timeout=2)
        print(result)
    except asyncio.TimeoutError:
        print("Request timed out!")

asyncio.run(main())
```

در اینجا چون تابع `fetch_data` پنج ثانیه طول می‌کشه ولی timeout دو ثانیه‌ست، خطای `TimeoutError` اتفاق می‌افته.

---

## کنترل تعداد تسک‌های هم‌زمان (Semaphore)

در بعضی شرایط، مثل دانلود صد فایل یا ارسال درخواست به چند سرور، ممکنه نخوای همه‌چیز هم‌زمان اجرا بشه چون فشار زیادی به سیستم یا سرور میاد. برای این کار از **Semaphore** استفاده می‌کنیم تا تعداد تسک‌های فعال در هر لحظه محدود باشه.

```python
import asyncio
import random

sem = asyncio.Semaphore(3)

async def download_file(i):
    async with sem:
        print(f"Downloading file {i}...")
        await asyncio.sleep(random.uniform(1, 3))
        print(f"File {i} downloaded.")

async def main():
    tasks = [download_file(i) for i in range(10)]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

در این مثال، فقط سه تسک در آنِ واحد فعال می‌شن. این روش بسیار مهمه برای جلوگیری از overload روی سیستم یا API.

---

## جمع‌بندی

در این درس یاد گرفتی:

* چطور با `asyncio.gather` خطاها رو کنترل کنی
* چطور تسک‌ها رو لغو یا timeout تعیین کنی
* چطور تعداد تسک‌های هم‌زمان رو محدود نگه داری

همهٔ این مفاهیم پایه‌ای برای ساخت سیستم‌های واقعی و قابل اعتماد هستن.
از این به بعد، async فقط یه ابزار نیست — بلکه یه روش حرفه‌ای برای کنترل رفتار برنامه‌ست.

---
