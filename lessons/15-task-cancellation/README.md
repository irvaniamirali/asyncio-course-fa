# لغو تسک‌ها و پاکسازی منابع در asyncio

تا اینجا یاد گرفتیم چطور با زمان کار کنیم و حتی برای عملیات‌هامون تایم‌اوت بذاریم. اما یه سوال مهم پیش میاد: اگه بخوایم یه تسک رو خودمون لغو کنیم چی؟ مثلاً کاربر وسط دانلود پشیمون بشه، یا اتصال قطع بشه؟ اینجاست که مفهوم لغو (Cancellation) وارد میشه.

---

## مفهوم لغو در asyncio

وقتی یه تسک در حال اجراست، می‌تونیم با صدا زدن `cancel()` اون رو متوقف کنیم. اما این توقف فوری نیست! در واقع asyncio یه درخواست لغو برای تسک می‌فرسته و تسک باید خودش به این لغو واکنش نشون بده.

> یعنی اگه تسک در جایی منتظر چیزی باشه (مثل `await asyncio.sleep()` یا `await session.get()`)، اون‌وقت لغو اعمال میشه. ولی اگه در یه حلقه‌ی معمولی گیر کرده باشه، لغو تا زمانی که کنترل به event loop برنگرده، انجام نمیشه.

---

## یه مثال ساده از لغو

فرض کن یه برنامه دانلود فایل داریم و می‌خوایم وسط کار لغوش کنیم:

```python
import asyncio

async def fake_download():
    print("Start downloading...")
    try:
        for i in range(5):
            await asyncio.sleep(1)
            print(f"Downloaded {20 * (i + 1)}%")
    except asyncio.CancelledError:
        print("Download cancelled!")
        raise
    finally:
        print("Cleaning up resources...")

async def main():
    task = asyncio.create_task(fake_download())
    await asyncio.sleep(2.5)
    print("User requested cancel!")
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("Task was cancelled successfully.")

asyncio.run(main())
```

در این مثال، بعد از حدود دو و نیم ثانیه، تسک لغو میشه. تسک وقتی لغو میشه، `CancelledError` دریافت می‌کنه و در `finally` می‌تونیم کارهای تمیزکاری رو انجام بدیم، مثل بستن فایل یا حذف فایل ناقص.

---

## فرق بین تایم‌اوت و لغو دستی

* در تایم‌اوت (مثلاً با `asyncio.wait_for`) تسک به صورت خودکار بعد از مدت مشخص لغو میشه.
* در لغو دستی، خودمون تصمیم می‌گیریم چه زمانی عملیات متوقف بشه.

در هر دو حالت، تسک در نهایت یه `CancelledError` می‌گیره، پس باید همیشه توی کدها آماده‌ی این اتفاق باشیم.

---

## لغو چند تسک همزمان

اگه چند تسک در حال اجرا باشن و بخوایم همشون رو لغو کنیم، می‌تونیم خیلی راحت با یه حلقه این کار رو انجام بدیم:

```python
import asyncio

async def worker(name):
    try:
        while True:
            print(f"{name} working...")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print(f"{name} got cancelled.")
        raise

async def main():
    tasks = [asyncio.create_task(worker(f"Worker-{i}")) for i in range(3)]
    await asyncio.sleep(3)
    print("Cancelling all workers...")
    for t in tasks:
        t.cancel()
    await asyncio.gather(*tasks, return_exceptions=True)
    print("All workers stopped.")

asyncio.run(main())
```

در این مثال هر worker به صورت جداگانه لغو میشه و با `return_exceptions=True` جلوی خطاهای ناخواسته رو می‌گیریم.

---

## نکات مهم هنگام لغو

* همیشه از `try / except asyncio.CancelledError` استفاده کن تا لغوها رو درست مدیریت کنی.
* از `finally` برای بستن فایل، connection یا cleanup استفاده کن.
* اگه تسک‌های دیگه به تسک لغوشده وابسته‌ان، حواست باشه که خطاها رو propagate کنی.



<p align="center">
<a href="../14-timeouts-and-time-management/README.md">درس قبلی</a>
&nbsp; | &nbsp;
<a href="../16-async-queues/README.md">درس بعدی</a>
