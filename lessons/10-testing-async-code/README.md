# درس دهم: تست‌نویسی برای کدهای async در پایتون

بعد از اینکه یاد گرفتیم چطور برنامه‌های async رو ساختار بدیم و تسک‌ها و منابع رو مدیریت کنیم، مرحله‌ی بعد اینه که مطمئن بشیم همه چیز واقعاً درست کار می‌کنه. برنامه‌های async به دلیل اجرای هم‌زمان چند تسک و وابستگی به منابع خارجی مثل شبکه یا فایل، بیشتر از برنامه‌های معمولی مستعد خطا هستن. بنابراین یادگیری تست‌نویسی async برای هر توسعه‌دهنده پایتون حیاتی است.

---

## چرا تست async متفاوت از sync است؟

تفاوت اصلی اینه که در برنامه‌های async توابع به صورت coroutine اجرا می‌شن و به event loop وابسته هستن. در تست معمولی، اجرای یک تابع ساده sync بلافاصله نتیجه می‌ده، اما در async باید منتظر بمونیم coroutine اجرا بشه و نتیجه رو بده. بدون مدیریت صحیح event loop، تست‌ها یا اجرا نمی‌شن، یا به صورت ناگهانی خطا می‌دن.

چالش‌های اصلی:

* تست تابعی که از network یا فایل استفاده می‌کنه.
* مدیریت exceptionها و timeoutها.
* تست هم‌زمانی و concurrency.

---

## ابزارهای تست async

یکی از محبوب‌ترین ابزارها برای تست async در پایتون، `pytest-asyncio` است. این ابزار اجازه می‌ده که تست‌ها به صورت طبیعی coroutine باشن و pytest event loop مناسب رو مدیریت کنه.

برای نصب:

```
pip install pytest-asyncio
```

نمونه استفاده:

```python
import pytest

@pytest.mark.asyncio
async def test_example():
    result = await some_async_function()
    assert result == expected_value
```

با این کار، نیازی نیست خودمون event loop بسازیم و مدیریت کنیم.

---

## تست coroutineهای ساده

فرض کن یک تابع async داریم که داده برمی‌گردونه یا ممکنه Exception بده:

```python
async def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    await asyncio.sleep(0.1)
    return a / b
```

برای تستش می‌توانیم بنویسیم:

```python
import pytest

@pytest.mark.asyncio
async def test_divide_success():
    result = await divide(10, 2)
    assert result == 5

@pytest.mark.asyncio
async def test_divide_exception():
    with pytest.raises(ValueError):
        await divide(10, 0)
```

این مثال نشون می‌ده که می‌تونیم هم موفقیت و هم خطاها رو به سادگی تست کنیم.

---

## تست I/O غیرهمزمان

برای تست توابعی که با شبکه یا فایل کار می‌کنن، باید منابع خارجی رو شبیه‌سازی (mock) کنیم تا تست‌ها سریع و پایدار باشن.

نمونه با aiohttp:

```python
from aiohttp import web
from aiohttp.test_utils import AioHTTPTestCase, unittest_run_loop

class MyTestCase(AioHTTPTestCase):
    async def get_application(self):
        app = web.Application()
        return app

    @unittest_run_loop
    async def test_fetch(self):
        resp = await self.client.request("GET", "/")
        assert resp.status == 200
```

نمونه با aiofiles:

```python
import aiofiles
import pytest

@pytest.mark.asyncio
async def test_write_file(tmp_path):
    file_path = tmp_path / "test.txt"
    async with aiofiles.open(file_path, 'w') as f:
        await f.write('hello async')

    async with aiofiles.open(file_path, 'r') as f:
        content = await f.read()

    assert content == 'hello async'
```

استفاده از `tmp_path` باعث می‌شه که فایل‌ها به صورت موقت ساخته و بعد از تست پاک بشن.

---

## تست هم‌زمانی و concurrency

زمانی که چند coroutine هم‌زمان اجرا می‌شن، ممکنه ترتیب اجرا مهم باشه یا منابع مشترک تحت فشار قرار بگیرن. تست باید اطمینان بده که همه چیز درست کار می‌کنه.

نمونه:

```python
import asyncio
import pytest

async def increment(counter):
    counter['value'] += 1

@pytest.mark.asyncio
async def test_concurrent_increment():
    counter = {'value': 0}
    tasks = [increment(counter) for _ in range(10)]
    await asyncio.gather(*tasks)
    assert counter['value'] == 10
```

در این مثال، هر coroutine یک واحد به مقدار counter اضافه می‌کنه. با gather مطمئن می‌شیم همه اجرا شدن.

---

## بهترین تمرین‌ها در تست async

* هر تست باید مستقل باشه و خودش event loop داشته باشه.
* منابع خارجی مثل session، connection یا فایل‌ها باید mock یا به صورت temp استفاده بشن.
* از assert دقیق و واضح برای اطمینان از رفتار درست استفاده کن.
* خطاها و timeoutها باید در تست‌ها شبیه‌سازی بشن.
* تست‌ها باید سریع باشن، نباید برای هر تست چند ثانیه منتظر بمونیم.


---

## جمع‌بندی

تست‌نویسی برای async، مهارت حیاتی برای هر توسعه‌دهنده پایتونه. با یادگیری pytest-asyncio و رعایت اصول طراحی تست:

* مطمئن می‌شیم coroutineها درست کار می‌کنن
* خطاها و timeoutها تحت کنترل هستن
* برنامه قابل نگهداری، قابل اعتماد و آماده توسعه است

---
