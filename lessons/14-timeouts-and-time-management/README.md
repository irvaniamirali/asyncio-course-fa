# مدیریت زمان و تایم‌اوت در asyncio


## چرا تایم‌اوت مهمه؟

قبل از هرچیز لازمه بدونی هدف اصلی تایم‌اوت اینه که برنامه‌ات گیر نکنه. وقتی با شبکه یا هر IO بیرونی کار می‌کنی، ممکنه یک‌درخواست بی‌پاسخ بمونه یا خیلی طول بکشه؛ تایم‌اوت بهت کمک می‌کنه مرز زمانی بذاری و منابع رو نجات بدی.

اگه تایم‌اوت نزاری، ممکنه event loop مشغول بمونه و بقیه وظایف هم دیر یا هیچ‌وقت اجرا نشن — مخصوصاً برای سرویس‌هایی که باید پاسخگو بمونن، این غیرقابل‌قبوله.

---

## ابزارهایی که باید بشناسی

این ابزارها بیشترین کاربرد رو دارن و مهمه طرز کار هر کدوم رو بدونی:

‏- `asyncio.wait_for(coro, timeout)` — ساده و مستقیم برای محدود کردن زمان اجرای یک coroutine.

‏- `asyncio.timeout(seconds)` — مدیریت‌کنندهٔ مبتنی‌بر context که از پایتون ۳.۱۱ اومده و خواناتر شده.

‏- `asyncio.wait(tasks, timeout=...)` — وقتی چند تسک هم‌زمان داری و می‌خوای روی گروه تسک‌ها timeout بذاری یا وقتی یکی کامل شد واکنش بدی.

‏- `asyncio.shield(task)` — وقتی نمی‌خوای تسکی از بیرون کانسل بشه و می‌خوای حتما کامل اجرا بشه (مثلاً برای cleanup).

حالا هر کدوم رو با مثال و نکته با هم مرور می‌کنیم.

---

## محدود کردن زمان اجرای یک coroutine با `wait_for`

قابلیت اصلی `wait_for` اینه که یک coroutine رو اجرا می‌کنه و اگه بیش از زمان مشخص طول بکشه، اون coroutine رو کانسل می‌کنه و `asyncio.TimeoutError` پرتاب می‌شه. این روش برای مواردی که یک عملیات مشخص داری، خیلی ساده و کاربردیه.

قبل از دیدن کد، این نکته رو در نظر داشته باش: وقتی `wait_for` کانسل می‌کنه، داخل آن coroutine یک `CancelledError` رخ می‌ده. پس حتما داخل coroutine از `try/finally` یا مدیریت مناسب استفاده کن تا منابع آزاد بشن.

مثال ساده:

```python
import asyncio

async def long_task():
    await asyncio.sleep(5)
    return "done"

async def main():
    try:
        result = await asyncio.wait_for(long_task(), timeout=2)
        print(result)
    except asyncio.TimeoutError:
        print("Task timed out")

asyncio.run(main())
```

در این مثال بعد از دو ثانیه `wait_for` تایم‌اوت می‌زنه و خطای `TimeoutError` رو بیرون می‌ده.

---

## استفاده از context manager خواناتر: `asyncio.timeout` (پایتون ۳.۱۱+)

وقتی چند عملیات پشت‌سرهم داری که مجموعا نباید از زمان مشخصی بگذره، استفاده از `asyncio.timeout` خیلی مناسب و خواناتره. این ساختار کد رو تمیزتر می‌کنه.

مثال:

```python
import asyncio

async def long_task():
    await asyncio.sleep(5)
    return "done"

async def main():
    try:
        async with asyncio.timeout(2):
            result = await long_task()
            print(result)
    except asyncio.TimeoutError:
        print("Timed out inside context")

asyncio.run(main())
```

این الگو خوبه وقتی می‌خوای چند await داخل یک بلوک رو با هم محدود کنی.

---

## مدیریت چند تسک با `asyncio.wait`

گاهی لازم داری چند تسک رو هم‌زمان اجرا کنی و بسته به اینکه کدوم زودتر تموم شه یا اگه timeout پیش بیاد، تصمیم بگیری. `asyncio.wait` این انعطاف رو میده.

مثال به‌صورت خلاصه:

```python
import asyncio

async def t(i, delay):
    await asyncio.sleep(delay)
    return i

async def main():
    tasks = {asyncio.create_task(t(1, 3)), asyncio.create_task(t(2, 1))}
    done, pending = await asyncio.wait(tasks, timeout=2, return_when=asyncio.FIRST_COMPLETED)
    print('done:', done)
    print('pending:', pending)

asyncio.run(main())
```

در این حالت اگر timeout رخ بده، مجموعهٔ `pending` شامل تسک‌هایی میشه که هنوز اجرا نشدن؛ تو باید تصمیم بگیری اون‌ها رو cancel کنی یا بذاری ادامه پیدا کنن.

---

## محافظت از تسک‌ها با `shield`

وقتی می‌خوای یک تسک حتماً کامل اجرا بشه حتی اگر والد یا caller بخواد آن را cancel کند، از `asyncio.shield` استفاده کن. این برای مواقعی مفیده که cleanup یا نوشتن لاگ نباید نیمه‌کاره بمونه.

نمونه:

```python
import asyncio

async def important():
    try:
        await asyncio.sleep(2)
        print("important done")
    finally:
        print("cleanup inside important")

async def main():
    t = asyncio.create_task(asyncio.shield(important()))
    await asyncio.sleep(0.1)
    t.cancel()  # try to cancel
    try:
        await t
    except asyncio.CancelledError:
        print("main observed cancel")

asyncio.run(main())
```

نکتهٔ مهم: `shield` تضمین نمی‌کنه که تابع داخلِ shield هرگز کانسل نشه، بلکه جلوی کانسلیشنِ بیرون رو می‌گیره. اگر خودِ تابع داخلی از کانسلیشن پذیر باشه ممکنه با رفتارهای داخلی خودش مواجه بشی.

---

## پاک‌سازی منابع هنگام کانسلیشن

هر جا منبعی مثل فایل، connection یا lock باز می‌کنیم، باید برای پاک‌سازی برنامه‌ریزی کنیم. یک الگوی متداول اینه که داخل coroutine از `try/finally` استفاده کنیم تا cleanup حتما اجرا بشه:

```python
async def task_with_resource():
    resource = acquire_resource()
    try:
        await do_work(resource)
    except asyncio.CancelledError:
        # اگر لازم بود رفتار خاصی هنگام کانسلیشن انجام بده
        raise
    finally:
        resource.close()
```

اگر cleanup `async` باشه و نیاز باشه `await` بشه، گاهی باید از `shield` استفاده کنی تا از کانسلیشنِ دوبارهٔ cleanup جلوگیری کنی.

---

## نکتهٔ حساس: timeout برای توابع بلاک‌شونده

باید بدونی وقتی از `run_in_executor` یا `to_thread` استفاده می‌کنی و سپس `wait_for` روش می‌ذاری، کانسل شدن await باعث متوقف شدن اجرای تابع در thread نمی‌شه. برای بعضی پروژه‌ها این رفتار مشکل‌سازِ. پس همیشه فرض کن که اجرای پس‌زمینه ممکنه ادامه پیدا کنه و برنامه‌ات را طوری طراحی کن که این موضوع خطری ایجاد نکنه.

مثال:

```python
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor()

def blocking_sleep(sec):
    time.sleep(sec)
    return f"slept {sec}"

async def main():
    loop = asyncio.get_running_loop()
    try:
        result = await asyncio.wait_for(loop.run_in_executor(executor, blocking_sleep, 5), timeout=2)
        print(result)
    except asyncio.TimeoutError:
        print("blocking call timed out, but thread may still be running")

asyncio.run(main())
```

در این مثال بعد از دو ثانیه `TimeoutError` می‌بینی، اما تابع `blocking_sleep` احتمالاً تا پنج ثانیه اجرا میشه — چون کانسلیشن روی thread اعمال نمی‌شه.

---

## استفاده از کتابخانه‌های async-aware در لایهٔ بالا

به‌جای wrap کردن نسخه‌های synchronous، هر وقت ممکنه از کتابخانه‌های async استفاده کن. برای مثال به جای `requests` از `aiohttp` یا `httpx.AsyncClient` استفاده کن که کنترل timeout و کانسلیشن رو بهتر و ایمن‌تر انجام می‌دن.

نمونه با `aiohttp`:

```python
import asyncio
import aiohttp

async def fetch(session, url):
    try:
        async with session.get(url, timeout=5) as resp:
            return await resp.text()
    except asyncio.TimeoutError:
        return None

async def main():
    async with aiohttp.ClientSession() as session:
        html = await fetch(session, 'https://example.com')
        print(len(html) if html else 'timed out')

asyncio.run(main())
```

در بسیاری از موارد این روش بهترین و کم‌دردسرترین راهه.

---

## بهترین شیوه‌ها و قواعد سرلوحه

* همیشه برای I/O خارجی تایم‌اوت تعیین کن: شبکه، دیتابیس، فایل‌های راه دور و غیره.
* از `asyncio.timeout` در پایتون جدید استفاده کن تا کد خواناتر باشه.
* برای cleanup از `try/finally` کمک بگیر و در صورت نیاز از `shield` استفاده کن.
* تایم‌اوت‌ها رو معقول انتخاب کن؛ نه خیلی کوتاه که false-positive بشه و نه خیلی طولانی که فایده‌ای نداشته باشه.
* وقتی توابع blocking رو در thread اجرا می‌کنی، فرض کن کانسلیشن تاثیری روی آن‌ها نداره و طراحی‌ات را مطابق با این فرض انجام بده.



<p align="center">
<a href="../13-blocking-calls-in-async/README.md">درس قبلی</a>
&nbsp; | &nbsp;
<a href="../15-task-cancellation/README.md">درس بعدی</a>
