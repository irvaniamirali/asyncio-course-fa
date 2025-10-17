## درس بیستم: ورودی و خروجی شبکه‌ای به‌صورت ناهمگام (Async Network I/O)

در این درس قراره با یکی از جذاب‌ترین بخش‌های AsyncIO آشنا بشیم: کار با شبکه به‌صورت ناهمگام. یعنی بتونیم سرور و کلاینت‌هایی بسازیم که بدون قفل شدن، هم‌زمان چندین اتصال شبکه‌ای رو مدیریت کنن.

وقتی صحبت از شبکه میشه، معمولاً منظورمون ارتباط بین دو برنامه‌ست که از طریق اینترنت یا شبکه محلی داده رد و بدل می‌کنن. مثلاً وقتی مرورگر از سرور درخواست صفحه می‌کنه یا وقتی بات تلگرام با API در ارتباطه. اگه بخوایم این ارتباط‌ها رو به‌صورت هم‌زمان و کارآمد پیاده‌سازی کنیم، AsyncIO ابزار اصلی ماست.

---

### مقدمه‌ای بر async network I/O

در مدل سنتی (synchronous I/O)، وقتی برنامه در حال خواندن یا نوشتن داده از شبکه‌ست، تا زمانی که اون عملیات تموم نشه، اجرای کل برنامه متوقف میشه. اما در مدل ناهمگام، عملیات شبکه‌ای فقط یه *تسک* میشه که تا زمان دریافت داده، کنترل رو به حلقه‌ی رویداد (event loop) برمی‌گردونه. این باعث میشه بتونیم صدها اتصال رو بدون بلاک شدن مدیریت کنیم.

---

### ساخت یک echo server ساده

اولین مثال ما یه سرور ساده‌ست که هر چی از کاربر دریافت کنه، همونو برمی‌گردونه.

```python
import asyncio

async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    print(f"Connected with {addr}")

    while True:
        data = await reader.read(100)
        if not data:
            print(f"Connection closed by {addr}")
            break
        message = data.decode()
        print(f"Received from {addr}: {message}")

        writer.write(data)
        await writer.drain()

    writer.close()
    await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 8888)

    addr = server.sockets[0].getsockname()
    print(f"Server started on {addr}")

    async with server:
        await server.serve_forever()

asyncio.run(main())
```

این سرور روی پورت ۸۸۸۸ منتظره و هر پیامی که از کلاینت بگیره، عین همونو برمی‌گردونه.

---

### تست کردن سرور با telnet یا netcat

برای تست می‌تونی از ابزارهایی مثل `telnet` یا `nc` استفاده کنی:

```bash
telnet 127.0.0.1 8888
```

هر چیزی که تایپ کنی، سرور همونو برمی‌گردونه. اگه چند تا تب ترمینال باز کنی و چند اتصال هم‌زمان برقرار کنی، متوجه می‌شی که سرور بدون بلاک شدن، همه رو مدیریت می‌کنه.

---

### ساخت یک کلاینت ناهمگام

حالا بیایم یه کلاینت بسازیم که با همین سرور ارتباط بگیره.

```python
import asyncio

async def tcp_client():
    reader, writer = await asyncio.open_connection('127.0.0.1', 8888)

    for msg in ["Hello", "How are you?", "Bye"]:
        writer.write(msg.encode())
        await writer.drain()

        data = await reader.read(100)
        print(f"Server replied: {data.decode()}")

    writer.close()
    await writer.wait_closed()

asyncio.run(tcp_client())
```

اینجا هم عملیات ارسال و دریافت به‌صورت ناهمگام انجام میشه و حلقه رویداد می‌تونه چندین کلاینت رو هم‌زمان کنترل کنه.

---

### مفهوم stream در asyncio

`StreamReader` و `StreamWriter` در واقع دو abstraction ساده برای کار با داده‌های شبکه هستن. به‌جای اینکه مستقیم با سوکت کار کنیم، asyncio این دو رو در اختیارمون می‌ذاره تا به شکل ساده‌تری بتونیم داده بخونیم و بنویسیم.

* **reader.read(n)**: تا حداکثر n بایت داده رو از اتصال می‌خونه.
* **writer.write(data)**: داده رو می‌نویسه، ولی فوراً ارسال نمی‌کنه.
* **writer.drain()**: صبر می‌کنه تا داده واقعاً ارسال بشه.

---

### ساخت یک سرور هم‌زمان با چند کلاینت

فرض کن می‌خوای یه چت سرور ساده بسازی. در این حالت باید پیام‌های هر کاربر برای بقیه کاربران ارسال بشه.

```python
import asyncio

clients = []

async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    clients.append(writer)
    print(f"{addr} connected")

    try:
        while True:
            data = await reader.readline()
            if not data:
                break
            message = f"{addr}: {data.decode()}"
            print(message.strip())
            for client in clients:
                if client is not writer:
                    client.write(message.encode())
                    await client.drain()
    finally:
        print(f"{addr} disconnected")
        clients.remove(writer)
        writer.close()
        await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 9000)
    print("Chat server started on port 9000")

    async with server:
        await server.serve_forever()

asyncio.run(main())
```

حالا چند ترمینال باز کن و با ابزار `nc` به سرور وصل شو:

```bash
nc 127.0.0.1 9000
```

هر پیامی که بنویسی برای بقیه‌ی کاربران هم ارسال میشه.

---

### بهینه‌سازی و مدیریت خطاها

در دنیای واقعی، شبکه پر از خطاهای غیرمنتظره‌ست: اتصال قطع میشه، داده نصفه میاد، یا طرف مقابل پاسخی نمی‌ده. بنابراین باید همیشه از `try/except` برای مدیریت خطاها استفاده کنی.

مثلاً:

```python
try:
    data = await reader.read(100)
except ConnectionResetError:
    print("Client disconnected unexpectedly")
```

همچنین بهتره برای timeout گذاشتن از `asyncio.wait_for` استفاده کنی:

```python
try:
    data = await asyncio.wait_for(reader.read(100), timeout=5)
except asyncio.TimeoutError:
    print("Timeout waiting for data")
```

---

### جمع‌بندی

در این درس یاد گرفتیم چطور از `asyncio` برای ساخت سرور و کلاینت‌های شبکه‌ای استفاده کنیم. فهمیدیم که:

* عملیات I/O ناهمگام چطور باعث کارایی بالا میشه.
* با `StreamReader` و `StreamWriter` میشه به‌سادگی با سوکت‌ها کار کرد.
* مدیریت چند اتصال هم‌زمان در asyncio بسیار راحت‌تر از مدل سنتیه.
---
