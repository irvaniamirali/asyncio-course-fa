# درس یازدهم: ساخت یک downloader هم‌زمان با asyncio

تا الان یاد گرفتیم چطور کدهای async بنویسیم، تسک‌ها رو مدیریت کنیم، خطاها و timeoutها رو کنترل کنیم و برنامه‌های ساده بسازیم. حالا وقتشه یاد بگیریم چطور یک پروژه‌ی واقعی بسازیم که همه‌ی این مفاهیم رو در عمل به کار ببره. در این درس یک downloader هم‌زمان می‌سازیم که می‌تونه چند فایل رو هم‌زمان دانلود کنه و همزمان منابع رو ایمن مدیریت کنه.

---

## طراحی پروژه

اولین قدم اینه که پروژه رو ساختار بدیم. یادمون باشه هر بخش باید یک مسئولیت مشخص داشته باشه:

* `fetcher.py`: مسئول دانلود فایل‌ها از اینترنت
* `saver.py`: مسئول ذخیره فایل‌ها روی دیسک
* `orchestrator.py`: مسئول هماهنگ کردن جریان دانلود و مدیریت تعداد تسک‌ها
* `main.py`: نقطه ورود برنامه

ساختار پوشه‌ای پیشنهادی:

```
async_downloader/
├── main.py
├── services/
│   ├── fetcher.py
│   ├── saver.py
│   └── orchestrator.py
└── utils/
    └── logger.py
```

با این ساختار هر بخش وظیفه خودش رو داره و نگهداری و توسعه پروژه راحت‌تره.

---

## نوشتن fetcher

وظیفه fetcher اینه که فایل‌ها رو از اینترنت دانلود کنه و نتیجه رو برگردونه. از aiohttp استفاده می‌کنیم چون کاملاً async است.

```python
import aiohttp
import asyncio

async def fetch(session, url):
    try:
        async with session.get(url) as response:
            if response.status != 200:
                raise Exception(f"Failed to download {url}")
            return await response.read()
    except Exception as e:
        print(f"Error fetching {url}: {e}")
        return None

async def fetch_all(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        return results
```

در این کد، `fetch` مسئول دانلود یک فایل است و تمام خطاها را مدیریت می‌کند. `fetch_all` چند URL را هم‌زمان دانلود می‌کند. `asyncio.gather` کمک می‌کند همه تسک‌ها هم‌زمان اجرا شوند.

---

## نوشتن saver

بعد از دانلود، باید فایل‌ها روی دیسک ذخیره شوند. برای این کار از aiofiles استفاده می‌کنیم:

```python
import aiofiles
import asyncio

async def save_file(filename, content):
    if content is None:
        print(f"No content to save for {filename}")
        return
    async with aiofiles.open(filename, 'wb') as f:
        await f.write(content)
        print(f"Saved {filename}")

async def save_all(files):
    tasks = [save_file(name, content) for name, content in files]
    await asyncio.gather(*tasks)
```

این کد تضمین می‌کنه که تمام فایل‌ها به‌صورت async ذخیره بشن و ذخیره شدن هر فایل اعلام بشه.

---

## نوشتن orchestrator

orchestrator مسئول اینه که fetcher و saver رو هماهنگ کنه و تعداد تسک‌ها رو کنترل کنه. با asyncio.Semaphore می‌توانیم تعداد دانلود هم‌زمان رو محدود کنیم:

```python
import asyncio
from services.fetcher import fetch
from services.saver import save_file

async def download_manager(urls, max_concurrent=3):
    semaphore = asyncio.Semaphore(max_concurrent)

    async def sem_fetch(url):
        async with semaphore:
            async with aiohttp.ClientSession() as session:
                return await fetch(session, url)

    tasks = [sem_fetch(url) for url in urls]
    results = await asyncio.gather(*tasks)

    save_tasks = [save_file(f"file_{i}.bin", content) for i, content in enumerate(results)]
    await asyncio.gather(*save_tasks)
```

در این بخش، هر fetch تنها وقتی اجرا می‌شود که semaphore اجازه بده، بنابراین منابع سیستم تحت فشار زیاد قرار نمی‌گیرن. بعد از دانلود همه، فایل‌ها ذخیره می‌شوند.

---

## نوشتن main

در main برنامه رو اجرا می‌کنیم و URLها رو تعریف می‌کنیم:

```python
import asyncio
from services.orchestrator import download_manager

urls = [
    "https://example.com/file1",
    "https://example.com/file2",
    "https://example.com/file3",
]

asyncio.run(download_manager(urls, max_concurrent=2))
```

با این کار برنامه همه فایل‌ها را هم‌زمان تا حد مشخص دانلود و ذخیره می‌کنه.

---

## مدیریت خطاها و retry

در پروژه‌های واقعی ممکنه دانلود به خاطر مشکل شبکه شکست بخوره. می‌توانیم retry اضافه کنیم:

```python
async def fetch_with_retry(session, url, retries=3):
    for attempt in range(retries):
        result = await fetch(session, url)
        if result is not None:
            return result
        await asyncio.sleep(1)
    print(f"Failed to download {url} after {retries} retries")
    return None
```

با این روش، فایل‌هایی که در ابتدا دانلود نمی‌شن، چند بار تلاش می‌شن تا موفقیت‌آمیز بشن.

---

## اضافه کردن log و پیشرفت دانلود

برای پروژه‌های بزرگ، خوبه که وضعیت دانلود رو log کنیم و پیشرفت نمایش بدیم:

```python
import logging
logging.basicConfig(level=logging.INFO)

async def save_file(filename, content):
    if content is None:
        logging.warning(f"No content to save for {filename}")
        return
    async with aiofiles.open(filename, 'wb') as f:
        await f.write(content)
    logging.info(f"Saved {filename}")
```

با log، می‌تونیم وضعیت همه تسک‌ها و خطاها رو راحت ببینیم.

---

## تست downloader

برای اطمینان از عملکرد downloader، باید تست بنویسیم:

* تست fetcher با URLهای واقعی و mock شده
* تست saver با tmp_path
* تست کل pipeline با asyncio.gather و بررسی تعداد فایل‌ها

این کار باعث می‌شه برنامه مطمئن و پایدار باشه.

---

## جمع‌بندی

در این درس یاد گرفتیم:

* چطور یک پروژه واقعی async بسازیم
* ساختار پوشه‌ای مناسب و تقسیم مسئولیت‌ها
* مدیریت concurrency با Semaphore
* ذخیره async فایل‌ها با aiofiles
* مدیریت خطاها و retry
* اضافه کردن log برای مشاهده وضعیت
* نوشتن تست برای downloader

---
