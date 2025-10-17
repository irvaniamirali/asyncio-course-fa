# درس هفتم: کار با ورودی و خروجی غیرهمزمان (Async I/O در عمل)

توی درس‌های قبلی یاد گرفتیم async و await چطور کار می‌کنن و حلقهٔ رویداد پشت صحنه چه جوری همه‌چی رو مدیریت می‌کنه. حالا وقتشه از این مفاهیم توی دنیای واقعی استفاده کنیم.

---

## هدف درس

هدف این درسه که بفهمیم برنامه‌نویسی غیرهمزمان چه کمکی به کار با **ورودی و خروجی (I/O)** می‌کنه؛ یعنی خوندن و نوشتن فایل‌ها، یا ارسال و دریافت داده از اینترنت.

---

## مفهوم Blocking I/O در مقابل Non-blocking I/O

وقتی با کد معمولی (synchronous) یه فایل می‌خونی یا از یه API داده می‌گیری، برنامه باید منتظر بمونه تا اون عملیات تموم بشه. به این حالت می‌گیم **Blocking I/O**.

اما در async، عملیات I/O متوقف‌کننده نیست. یعنی برنامه می‌تونه هم‌زمان چندتا کار انجام بده، بدون اینکه منتظر بمونه.

---

## کار با فایل‌ها به صورت async

برای اینکه بتونیم فایل‌هامون رو به شکل async بخونیم یا بنویسیم، از کتابخونه‌ی [aiofiles](https://github.com/Tinche/aiofiles) استفاده می‌کنیم.

```python
import asyncio
import aiofiles

async def read_file(path: str):
    async with aiofiles.open(path, mode='r') as f:
        content = await f.read()
        print(f"{path} → {len(content)} bytes")

async def main():
    tasks = [
        read_file('file1.txt'),
        read_file('file2.txt'),
        read_file('file3.txt'),
    ]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

در این مثال، سه فایل هم‌زمان خونده می‌شن، بدون اینکه برنامه منتظر تموم شدن یکی بمونه.

---

## درخواست‌های شبکه‌ای async

برای ارتباط با APIها می‌تونیم از کتابخونهٔ [aiohttp](https://docs.aiohttp.org) استفاده کنیم:

```python
import asyncio
import aiohttp

URLS = [
    'https://jsonplaceholder.typicode.com/todos/1',
    'https://jsonplaceholder.typicode.com/todos/2',
    'https://jsonplaceholder.typicode.com/todos/3',
]

async def fetch(session, url):
    async with session.get(url) as response:
        data = await response.json()
        print(data['title'])

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in URLS]
        await asyncio.gather(*tasks)

asyncio.run(main())
```

در اینجا، سه درخواست HTTP هم‌زمان انجام می‌شن، و نتیجه سریع‌تر از نسخهٔ سنتی برمی‌گرده.

---

## نکتهٔ مهم

‏async همیشه بهتر نیست. اگه برنامه‌ت کار CPU زیادی انجام می‌ده (مثلاً پردازش تصویر)، async کمکی نمی‌کنه.
اما برای کارهای I/O مثل خوندن فایل، درخواست شبکه یا کار با دیتابیس‌های async عالیه.



<p align="center">
<a href="../06-event-loop-deepdive/README.md">درس قبلی</a>
&nbsp; | &nbsp;
<a href="../08-task-management-and-errors/README.md">درس بعدی</a>
