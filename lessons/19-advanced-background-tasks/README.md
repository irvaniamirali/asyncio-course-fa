# طراحی Background Task در asyncio

تصور کن یه برنامه داری که یه سری کار اصلی انجام می‌ده، مثلاً یه وب‌سرور که به درخواست‌های کاربرا جواب می‌ده. حالا می‌خوای یه سری کار دیگه هم تو پس‌زمینه انجام بشه، بدون اینکه کار اصلی برنامه (مثل پاسخ به کاربرا) متوقف بشه. مثلاً می‌خوای هر چند ثانیه یه بار دیتابیس رو چک کنی، یه ایمیل بفرستی، یا یه فایل رو پردازش کنی. اینجاست که **background task** توی **asyncio** به کار میاد.

‏**Background task** یه تسک (یا coroutine) هست که تو پس‌زمینه اجرا می‌شه و روی **event loop** اصلی برنامه تأثیر منفی نمی‌ذاره. این تسک‌ها معمولاً کارهایی هستن که باید به‌صورت دوره‌ای (periodic) یا مداوم اجرا بشن، بدون اینکه جریان اصلی برنامه رو مختل کنن.

---

## چرا به Background Task نیاز داریم؟

فرض کن یه وبسایت داری که کاربرا می‌تونن توش پیام بفرستن. می‌خوای هر 5 دقیقه یه بار پیام‌های قدیمی رو از دیتابیس پاک کنی. اگه این کار رو تو همون **event loop** اصلی انجام بدی و یه دفعه دیتابیس کند بشه، کل وبسایتت ممکنه قفل کنه! **Background task** بهت کمک می‌کنه اینجور کارها رو جداگونه و بدون مزاحمت برای کارهای اصلی انجام بدی.

چندتا سناریوی رایج که نیاز به **background task** دارن:
- **پاکسازی دوره‌ای**: مثلاً حذف فایل‌های موقت یا لاگ‌های قدیمی.
- **نوتیفیکیشن**: ارسال ایمیل یا اعلان به کاربرا تو زمان‌های مشخص.
- **به‌روزرسانی داده**: مثلاً گرفتن قیمت سهام یا اطلاعات آب‌وهوا هر چند دقیقه.
- **پردازش داده**: مثل پردازش فایل‌های آپلودشده توسط کاربرا تو پس‌زمینه.

---

## چطور Background Task بسازیم؟

توی **asyncio** چند روش اصلی برای ساخت **background task** وجود داره:

1. **استفاده از `asyncio.create_task`**: این روش ساده‌ترین راهه برای اجرای یه coroutine تو پس‌زمینه.
2. **حلقه‌های دوره‌ای با `asyncio.sleep`**: برای کارهایی که باید مداوم یا با فاصله زمانی اجرا بشن.
3. **ترکیب با `asyncio.Queue`**: اگه بخوای یه سری کار رو به ترتیب تو پس‌زمینه پردازش کنی.
4. **مدیریت لغو (cancellation)**: برای اینکه بتونی تسک‌های پس‌زمینه رو درست مدیریت کنی و اگه لازم شد متوقفشون کنی.

بیایم هر کدوم رو با مثال‌های واقعی بررسی کنیم.

---

### 1. استفاده از `asyncio.create_task`

ساده‌ترین راه برای ساخت یه **background task** اینه که یه coroutine رو با `asyncio.create_task` اجرا کنی. این روش تسک رو به **event loop** می‌سپره و خودش تو پس‌زمینه اجرا می‌شه، بدون اینکه بقیه برنامه رو متوقف کنه.

#### یه مثال ساده
فرض کن یه برنامه داری که باید یه پیام ساده رو هر چند ثانیه تو پس‌زمینه چاپ کنه، در حالی که برنامه اصلی داره کارای دیگه انجام می‌ده.

```python
import asyncio

async def background_task():
    while True:
        print("Background task is running...")
        await asyncio.sleep(2)  # هر 2 ثانیه اجرا می‌شه

async def main():
    # ساخت تسک پس‌زمینه
    asyncio.create_task(background_task())
    
    # کار اصلی برنامه
    for i in range(5):
        print(f"Main task is doing work {i}")
        await asyncio.sleep(1)

if __name__ == "__main__":
    asyncio.run(main())
```

#### این کد چیکار می‌کنه؟
1. تابع `background_task` یه حلقه بی‌نهایت داره که هر 2 ثانیه یه پیام چاپ می‌کنه.
2. با `asyncio.create_task` این تابع رو تو پس‌زمینه اجرا می‌کنیم.
3. تابع `main` کار اصلی برنامه رو انجام می‌ده (چاپ پیام هر 1 ثانیه).
4. چون تسک پس‌زمینه با `create_task` اجرا شده، همزمان با کار اصلی پیش می‌ره.

#### خروجی چطوره؟
خروجی چیزی شبیه اینه:
```
Main task is doing work 0
Background task is running...
Main task is doing work 1
Main task is doing work 2
Background task is running...
Main task is doing work 3
Main task is doing work 4
Background task is running...
```

#### یه مثال واقعی‌تر
فرض کن یه وب‌سرور داری که با **FastAPI** کار می‌کنه. می‌خوای هر 10 ثانیه یه بار وضعیت سرور (مثلاً مصرف CPU) رو چک کنی و تو یه فایل لاگ کنی.

```python
import asyncio
import psutil

async def log_cpu_usage():
    while True:
        cpu_percent = psutil.cpu_percent()
        with open("cpu_usage.log", "a") as f:
            f.write(f"CPU Usage: {cpu_percent}%\n")
        print(f"Logged CPU usage: {cpu_percent}%")
        await asyncio.sleep(10)

async def main():
    # راه‌اندازی تسک پس‌زمینه
    asyncio.create_task(log_cpu_usage())
    
    # شبیه‌سازی کار وب‌سرور
    print("Web server is running...")
    await asyncio.sleep(30)  # وب‌سرور برای 30 ثانیه اجرا می‌شه

if __name__ == "__main__":
    asyncio.run(main())
```

این کد مصرف CPU رو هر 10 ثانیه تو یه فایل لاگ می‌کنه، در حالی که وب‌سرور (یا هر کار اصلی دیگه) بدون وقفه ادامه می‌ده.

---

### 2. حلقه‌های دوره‌ای با `asyncio.sleep`

اگه بخوای یه کار به‌صورت دوره‌ای (مثلاً هر 5 دقیقه) اجرا بشه، می‌تونی یه حلقه با `asyncio.sleep` بسازی. این روش برای کارهایی مثل به‌روزرسانی داده یا ارسال نوتیفیکیشن خیلی مناسبه.

#### یه مثال ساده
فرض کن می‌خوای هر 3 ثانیه یه پیام به یه API بفرستی تا وضعیت یه دستگاه رو چک کنی.

```python
import asyncio
import aiohttp

async def check_device_status():
    async with aiohttp.ClientSession() as session:
        while True:
            async with session.get("http://device-api.example.com/status") as resp:
                status = await resp.json()
                print(f"Device status: {status}")
            await asyncio.sleep(3)

async def main():
    asyncio.create_task(check_device_status())
    
    # کار اصلی
    print("Main program is running...")
    await asyncio.sleep(10)

if __name__ == "__main__":
    asyncio.run(main())
```

#### این کد چیکار می‌کنه؟
1. تابع `check_device_status` هر 3 ثانیه یه درخواست HTTP به API می‌فرسته و وضعیت دستگاه رو چک می‌کنه.
2. با `create_task` این تابع تو پس‌زمینه اجرا می‌شه.
3. تابع `main` کار اصلی برنامه رو انجام می‌ده (اینجا فقط یه شبیه‌سازی ساده‌ست).

#### یه مثال واقعی‌تر
فرض کن یه برنامه داری که باید هر 5 دقیقه قیمت بیت‌کوین رو از یه API بگیره و اگه تغییر زیادی داشت، به کاربرا ایمیل بفرسته.

```python
import asyncio
import aiohttp

async def monitor_bitcoin_price():
    last_price = None
    async with aiohttp.ClientSession() as session:
        while True:
            async with session.get("http://api.example.com/bitcoin/price") as resp:
                data = await resp.json()
                price = data["price"]
                print(f"Bitcoin price: ${price}")
                if last_price and abs(price - last_price) > 100:
                    print("Price changed significantly! Sending email...")
                    # اینجا کد ارسال ایمیل میاد
                last_price = price
            await asyncio.sleep(300)  # هر 5 دقیقه

async def main():
    asyncio.create_task(monitor_bitcoin_price())
    print("Main program is running...")
    await asyncio.sleep(1000)  # برنامه برای مدتی اجرا می‌شه

if __name__ == "__main__":
    asyncio.run(main())
```

این کد قیمت بیت‌کوین رو هر 5 دقیقه چک می‌کنه و اگه تغییر زیادی داشته باشه، اعلان می‌فرسته.

---

### 3. ترکیب با `asyncio.Queue`

اگه بخوای یه سری کار رو به ترتیب تو پس‌زمینه پردازش کنی، **Queue** خیلی به کار میاد. می‌تونی کارها رو تو یه صف بذاری و یه **background task** اونا رو یکی‌یکی پردازش کنه.

#### یه مثال ساده
فرض کن یه برنامه داری که کاربرا فایل آپلود می‌کنن و باید تو پس‌زمینه پردازش بشن (مثلاً تبدیل به PDF).

```python
import asyncio

async def process_file(file_name):
    print(f"Processing {file_name}...")
    await asyncio.sleep(2)  # شبیه‌سازی پردازش
    print(f"Finished processing {file_name}")

async def background_processor(queue):
    while True:
        file_name = await queue.get()
        if file_name is None:
            queue.task_done()
            break
        await process_file(file_name)
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    # راه‌اندازی پردازشگر پس‌زمینه
    asyncio.create_task(background_processor(queue))
    
    # شبیه‌سازی آپلود فایل توسط کاربرا
    files = ["file1.txt", "file2.txt", "file3.txt"]
    for file in files:
        print(f"User uploaded {file}")
        await queue.put(file)
    await queue.put(None)  # علامت پایان

if __name__ == "__main__":
    asyncio.run(main())
```

#### این کد چیکار می‌کنه؟
1. یه **Queue** درست کردیم که فایل‌ها رو نگه می‌داره.
2. تابع `background_processor` فایل‌ها رو از **Queue** می‌گیره و پردازش می‌کنه.
3. تابع `main` شبیه‌سازی می‌کنه که کاربرا فایل آپلود می‌کنن و تو **Queue** می‌ذارن.
4. با `create_task`، پردازشگر تو پس‌زمینه اجرا می‌شه.

#### خروجی چطوره؟
خروجی چیزی شبیه اینه:
```
User uploaded file1.txt
User uploaded file2.txt
User uploaded file3.txt
Processing file1.txt...
Finished processing file1.txt
Processing file2.txt...
Finished processing file2.txt
Processing file3.txt...
Finished processing file3.txt
```

#### یه مثال واقعی‌تر
فرض کن یه اپلیکیشن چت داری که باید پیام‌های کاربرا رو تو پس‌زمینه ذخیره کنه تو دیتابیس.

```python
import asyncio
import sqlite3

async def save_message(queue):
    conn = sqlite3.connect("messages.db")
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE IF NOT EXISTS messages (id INTEGER PRIMARY KEY, content TEXT)")
    
    while True:
        message = await queue.get()
        if message is None:
            queue.task_done()
            break
        print(f"Saving message: {message}")
        cursor.execute("INSERT INTO messages (content) VALUES (?)", (message,))
        conn.commit()
        queue.task_done()
    
    conn.close()

async def main():
    queue = asyncio.Queue()
    asyncio.create_task(save_message(queue))
    
    # شبیه‌سازی ارسال پیام توسط کاربرا
    messages = ["Hi!", "How are you?", "See you later!"]
    for msg in messages:
        print(f"User sent: {msg}")
        await queue.put(msg)
    await queue.put(None)

if __name__ == "__main__":
    asyncio.run(main())
```

این کد پیام‌ها رو تو یه دیتابیس SQLite ذخیره می‌کنه، بدون اینکه جریان اصلی برنامه (مثلاً چت) مختل بشه.

---

### 4. مدیریت لغو (Cancellation) تسک‌های پس‌زمینه

یه نکته مهم درباره **background task** اینه که باید بتونی اونا رو درست مدیریت کنی، مخصوصاً اگه بخوای برنامه رو ببندی یا تسک رو متوقف کنی. اگه تسک‌های پس‌زمینه رو درست لغو نکنی، ممکنه منابع سیستم (مثل حافظه یا اتصال به دیتابیس) آزاد نشن.

#### یه مثال ساده
فرض کن یه تسک پس‌زمینه داری که باید بتونی لغوش کنی.

```python
import asyncio

async def background_task():
    try:
        while True:
            print("Background task is running...")
            await asyncio.sleep(2)
    except asyncio.CancelledError:
        print("Background task was cancelled!")
        raise

async def main():
    task = asyncio.create_task(background_task())
    
    # چند ثانیه صبر کن
    await asyncio.sleep(5)
    
    # لغو تسک
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("Main: Task was cancelled")

if __name__ == "__main__":
    asyncio.run(main())
```

#### این کد چیکار می‌کنه؟
1. تابع `background_task` یه حلقه بی‌نهایت داره که هر 2 ثانیه پیام چاپ می‌کنه.
2. با `create_task` این تسک تو پس‌زمینه اجرا می‌شه.
3. بعد از 5 ثانیه، تسک رو با `task.cancel()` لغو می‌کنیم.
4. با `except asyncio.CancelledError` مطمئن می‌شیم که لغو شدن تسک درست مدیریت می‌شه.

#### خروجی چطوره؟
خروجی چیزی شبیه اینه:
```
Background task is running...
Background task is running...
Background task is running...
Background task was cancelled!
Main: Task was cancelled
```

#### یه مثال واقعی‌تر
فرض کن یه برنامه داری که تو پس‌زمینه لاگ‌های سرور رو به یه سرور دیگه می‌فرسته. وقتی برنامه بسته می‌شه، باید مطمئن شی که تسک پس‌زمینه هم درست متوقف می‌شه.

```python
import asyncio
import aiohttp

async def send_logs():
    async with aiohttp.ClientSession() as session:
        try:
            while True:
                async with session.post("http://log-server.example.com", json={"log": "Server is alive"}) as resp:
                    print("Log sent!")
                await asyncio.sleep(5)
        except asyncio.CancelledError:
            print("Log sender was cancelled!")
            raise

async def main():
    task = asyncio.create_task(send_logs())
    
    print("Main program is running...")
    await asyncio.sleep(12)
    
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("Main: Log sender was cancelled")

if __name__ == "__main__":
    asyncio.run(main())
```

این کد نشون می‌ده چطور یه تسک پس‌زمینه رو درست لغو کنیم.

---

## نکات کلیدی و Best Practiceها

1. **از `create_task` برای سادگی استفاده کن**  

برای اکثر **background task**ها، `asyncio.create_task` ساده‌ترین و بهترین روشه.

2. **لغو تسک‌ها رو فراموش نکن**  

همیشه یه مکانیسم برای لغو تسک‌های پس‌زمینه بذار، مخصوصاً اگه برنامه قراره بسته بشه.

3. **منابع رو آزاد کن**  

اگه تسک پس‌زمینه از منابعی مثل دیتابیس یا اتصال شبکه استفاده می‌کنه، مطمئن شو که تو حالت لغو درست بسته می‌شن.

4. **از Queue برای کارهای پیچیده استفاده کن**  

اگه تسک‌های پس‌زمینه‌ات به ترتیب خاصی باید پردازش بشن، **Queue** خیلی به کار میاد.

6. **مراقب حلقه‌های بی‌نهایت باش**  

اگه از حلقه بی‌نهایت تو تسک پس‌زمینه استفاده می‌کنی، حتماً با `try/except` برای `CancelledError` آماده باش.

8. **از کتابخونه‌های async استفاده کن**  

اگه تسک پس‌زمینه‌ات شامل درخواست شبکه‌ایه، به جای **requests** از **aiohttp** استفاده کن تا نیازی به **thread** یا **process** نداشته باشی.

---

## سوالات رایج و جواب‌هاشون

**1. تفاوت `create_task` با `gather` چیه؟**  
`asyncio.gather` برای اجرای همزمان چندتا coroutine و جمع‌آوری نتایجشونه. `create_task` برای اجرای یه coroutine تو پس‌زمینه‌ست و نتیجه‌ش رو مستقیم نمی‌گیره.

**2. اگه تسک پس‌زمینه رو لغو نکنم چی میشه؟**  
اگه تسک لغو نشه، ممکنه تا ابد اجرا بشه و منابع سیستم (مثل حافظه یا اتصال شبکه) رو اشغال کنه. همیشه یه راه برای لغو تسک بذار.

**3. می‌تونم چندتا تسک پس‌زمینه داشته باشم؟**  
بله! می‌تونی با `create_task` چندتا تسک پس‌زمینه درست کنی. فقط مطمئن شو که منابع سیستم (مثل CPU یا حافظه) رو بیش از حد مصرف نمی‌کنن.

**4. اگه تسک پس‌زمینه به یه منبع مشترک نیاز داشت چی؟**  
از ابزارهای هم‌زمانی مثل **Lock** یا **Semaphore** استفاده کن تا از **race condition** جلوگیری کنی.

**5. برای کارهای دوره‌ای بهتره چیکار کنم؟**  
برای کارهای دوره‌ای، یه حلقه با `asyncio.sleep` تو یه تسک پس‌زمینه بساز. اگه کار پیچیده‌ست، می‌تونی از کتابخونه‌هایی مثل **APScheduler** هم استفاده کنی.



<p align="center">
<a href="../18-periodic-tasks/README.md">درس قبلی</a>
&nbsp; | &nbsp;
<a href="../20-async-network-io/README.md">درس بعدی</a>
