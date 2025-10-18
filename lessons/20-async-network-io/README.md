# درس بیستم: ورودی و خروجی شبکه‌ای به‌صورت ناهمگام (Async Network I/O)

شبکه یعنی ارتباط بین دو برنامه، مثل وقتی مرورگرت یه صفحه وب رو از سرور لود می‌کنه یا یه بات تلگرام با API ارتباط برقرار می‌کنه. تو این موقعیت‌ها، اگه بخوای تعداد زیادی اتصال رو همزمان و با کارایی بالا مدیریت کنی، asyncio ابزار اصلیته. تو این درس، با مثال‌های واقعی و روشن، مفاهیم رو قدم به قدم توضیح می‌دم تا کاملاً جا بیفته.

---

## چرا Async Network I/O مهمه؟

تصور کن یه کافی‌شاپ شلوغ داری. اگه گارسون تو برای هر مشتری وایسه تا سفارشش کامل آماده بشه و بعد بره سراغ مشتری بعدی، همه مشتری‌ها عصبانی می‌شن و صف طولانی می‌شه. حالا اگه گارسون بتونه سفارش یه مشتری رو بگیره، بذاره تو صف آشپزخونه، و بره سراغ مشتری بعدی، چی؟ کل سیستم سریع‌تر و کارآمدتر می‌شه!

توی برنامه‌نویسی شبکه‌ای هم همین‌طوره. تو روش سنتی (synchronous I/O)، وقتی یه برنامه داره داده از شبکه می‌خونه یا می‌نویسه، تا وقتی اون کار تموم نشه، کل برنامه منتظر می‌مونه. این یعنی اگه یه کلاینت کند باشه، همه‌چیز قفل می‌کنه! اما تو مدل ناهمگام (**asyncio**)، عملیات شبکه‌ای فقط یه تسک (task) می‌شن که کنترل رو به **event loop** برمی‌گردونن. اینطوری می‌تونی صدها یا حتی هزارها اتصال رو همزمان مدیریت کنی، بدون اینکه چیزی بلاک بشه.

---

## ابزارهای اصلی: StreamReader و StreamWriter

قبل از اینکه بریم سراغ مثال‌ها، بیایم با دو تا ابزار کلیدی تو **asyncio** آشنا بشیم: **StreamReader** و **StreamWriter**. اینا مثل دو تا دستیار باحالن که کار با سوکت‌های شبکه‌ای رو ساده‌تر می‌کنن.

‏- **StreamReader**: برای خوندن داده از شبکه استفاده می‌شه. مثلاً می‌تونی بگی "تا 100 بایت داده بخون" یا "یه خط کامل بخون".

‏- **StreamWriter**: برای نوشتن داده تو شبکه استفاده می‌شه. داده رو می‌نویسی، ولی تا وقتی نگی "ارسال کن"، چیزی نمی‌ره.

چندتا متد مهم:

‏- `reader.read(n)`: تا حداکثر `n` بایت داده از اتصال می‌خونه.

‏- `reader.readline()`: یه خط کامل (تا `\n`) می‌خونه.

‏- `writer.write(data)`: داده رو آماده ارسال می‌کنه، ولی هنوز نمی‌فرسته.

‏- `writer.drain()`: صبر می‌کنه تا داده واقعاً از بافر ارسال بشه.
‏- `writer.close()`: اتصال رو می‌بنده.

این ابزارها باعث می‌شن کار با شبکه خیلی تمیز و ساده بشه. حالا بیایم با یه مثال عملی شروع کنیم.

---

## ساخت یه Echo Server ساده

اولین چیزی که قراره بسازیم یه **echo server** ساده‌ست. این سرور هر پیامی که از کلاینت بگیره، همونو عیناً برمی‌گردونه. مثل یه آینه که هر چی بهش بگی، تکرار می‌کنه!

```python
import asyncio

async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    print(f"کلاینت {addr} وصل شد")

    while True:
        data = await reader.read(100)
        if not data:
            print(f"اتصال توسط {addr} بسته شد")
            break
        message = data.decode()
        print(f"دریافت از {addr}: {message}")

        writer.write(data)
        await writer.drain()

    writer.close()
    await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 8888)
    addr = server.sockets[0].getsockname()
    print(f"سرور روی {addr} شروع به کار کرد")

    async with server:
        await server.serve_forever()

if __name__ == "__main__":
    asyncio.run(main())
```

### این کد چیکار می‌کنه؟
1. تابع `handle_client` برای هر کلاینت که وصل می‌شه اجرا می‌شه. آدرس کلاینت رو با `get_extra_info('peername')` می‌گیره.
2. تو یه حلقه، داده رو با `reader.read(100)` می‌خونه (حداکثر 100 بایت).
3. اگه داده‌ای نبود (`not data`)، یعنی کلاینت اتصال رو بسته و حلقه تموم می‌شه.
4. داده رو با `decode()` به رشته تبدیل می‌کنه، چاپ می‌کنه، و همونو با `writer.write` برمی‌گردونه.
5. با `writer.drain()` مطمئن می‌شه داده واقعاً ارسال شده.
6. آخرش اتصال رو با `writer.close()` و `wait_closed()` می‌بنده.
7. تابع `main` سرور رو روی آدرس `127.0.0.1` و پورت `8888` راه‌اندازی می‌کنه و با `serve_forever()` منتظر کلاینت‌ها می‌مونه.

### چطور تستش کنیم؟
برای تست، می‌تونی از ابزارهایی مثل `telnet` یا `nc` (netcat) استفاده کنی. تو ترمینال بزن:

```bash
telnet 127.0.0.1 8888
```

بعد هر چی تایپ کنی، سرور همونو برمی‌گردونه. مثلاً اگه بنویسی `Hello`， سرور می‌گه `Hello`. حالا چندتا ترمینال باز کن و چندتا کلاینت وصل کن. می‌بینی که سرور همه‌شون رو همزمان مدیریت می‌کنه، بدون اینکه قفل کنه!

#### خروجی نمونه:
اگه تو ترمینال با `telnet` وصل شی و بنویسی `Hello World`، خروجی سرور چیزی شبیه اینه:
```
سرور روی ('127.0.0.1', 8888) شروع به کار کرد
کلاینت ('127.0.0.1', 12345) وصل شد
دریافت از ('127.0.0.1', 12345): Hello World
اتصال توسط ('127.0.0.1', 12345) بسته شد
```

---

## ساخت یه کلاینت ناهمگام

حالا که سرور داریم، بیایم یه کلاینت بسازیم که بتونه با این سرور گپ بزنه. کلاینت قراره چندتا پیام بفرسته و جواب سرور رو بگیره.

```python
import asyncio

async def tcp_client():
    reader, writer = await asyncio.open_connection('127.0.0.1', 8888)
    print("به سرور وصل شدیم")

    messages = ["Hello", "How are you?", "Bye"]
    for msg in messages:
        writer.write(msg.encode())
        await writer.drain()
        print(f"ارسال: {msg}")

        data = await reader.read(100)
        print(f"دریافت از سرور: {data.decode()}")

    writer.close()
    await writer.wait_closed()
    print("اتصال بسته شد")

if __name__ == "__main__":
    asyncio.run(tcp_client())
```

### این کد چیکار می‌کنه؟
1. با `asyncio.open_connection` یه اتصال به سرور روی `127.0.0.1:8888` باز می‌کنه و یه **StreamReader** و **StreamWriter** برمی‌گردونه.
2. چندتا پیام (`Hello`, `How are you?`, `Bye`) رو یکی‌یکی encode می‌کنه و با `writer.write` می‌فرسته.
3. با `writer.drain()` مطمئن می‌شه داده‌ها واقعاً ارسال شدن.
4. جواب سرور رو با `reader.read(100)` می‌خونه و چاپ می‌کنه.
5. آخرش اتصال رو می‌بنده.

### خروجی نمونه:
اگه سرور بالا باشه، خروجی کلاینت چیزی شبیه اینه:
```
به سرور وصل شدیم
ارسال: Hello
دریافت از سرور: Hello
ارسال: How are you?
دریافت از سرور: How are you?
ارسال: Bye
دریافت از سرور: Bye
اتصال بسته شد
```

### یه مثال واقعی‌تر
فرض کن یه بات تلگرام داری که باید با یه سرور داخلی گپ بزنه و دستورات کاربرا رو بفرسته. کلاینت می‌تونه دستورات رو به سرور بفرسته و جوابش رو بگیره.

```python
import asyncio

async def telegram_bot_client():
    reader, writer = await asyncio.open_connection('127.0.0.1', 8888)
    print("بات به سرور وصل شد")

    commands = ["/start", "/help", "/status"]
    for cmd in commands:
        writer.write(cmd.encode())
        await writer.drain()
        print(f"ارسال دستور: {cmd}")

        data = await reader.read(100)
        print(f"جواب سرور: {data.decode()}")

    writer.close()
    await writer.wait_closed()
    print("اتصال بات بسته شد")

if __name__ == "__main__":
    asyncio.run(telegram_bot_client())
```

این کد یه بات ساده رو شبیه‌سازی می‌کنه که دستورات تلگرامی رو به سرور می‌فرسته و جواب می‌گیره.

---

## ساخت یه چت سرور با چند کلاینت

حالا بیایم یه پروژه جذاب‌تر بسازیم: یه **چت سرور** که چندتا کلاینت بتونن باهاش وصل بشن و پیام‌هاشون برای همه کلاینت‌های دیگه پخش بشه. مثل یه گروه چت ساده!

```python
import asyncio

clients = []

async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    clients.append(writer)
    print(f"کلاینت {addr} وصل شد")

    try:
        while True:
            data = await reader.readline()
            if not data:
                break
            message = f"{addr}: {data.decode().strip()}"
            print(message)
            for client in clients:
                if client is not writer:
                    client.write(message.encode())
                    await client.drain()
    except Exception as e:
        print(f"خطا برای {addr}: {e}")
    finally:
        print(f"کلاینت {addr} قطع شد")
        clients.remove(writer)
        writer.close()
        await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 9000)
    print("چت سرور روی پورت 9000 شروع شد")

    async with server:
        await server.serve_forever()

if __name__ == "__main__":
    asyncio.run(main())
```

### این کد چیکار می‌کنه؟
1. یه لیست `clients` داریم که همه **StreamWriter**های کلاینت‌ها رو نگه می‌داره.
2. تابع `handle_client` برای هر کلاینت اجرا می‌شه:
   - آدرس کلاینت رو می‌گیره و به لیست `clients` اضافه می‌کنه.
   - تو یه حلقه، پیام‌های کلاینت رو با `reader.readline()` می‌خونه.
   - پیام رو برای همه کلاینت‌های دیگه (به جز خودش) می‌فرسته.
   - اگه کلاینت قطع بشه یا خطایی پیش بیاد، با `finally` مطمئن می‌شیم که کلاینت از لیست حذف بشه و اتصالش بسته بشه.
3. تابع `main` سرور رو روی پورت `9000` راه‌اندازی می‌کنه.

### چطور تستش کنیم؟
چندتا ترمینال باز کن و تو هر کدوم بزن:

```bash
nc 127.0.0.1 9000
```

حالا تو هر ترمینال پیامی بنویس. می‌بینی که پیامت برای همه کلاینت‌های دیگه هم می‌ره!

#### خروجی نمونه:
اگه دو تا کلاینت با `nc` وصل بشن و یکی بنویسه `Hi everyone!`، خروجی سرور چیزی شبیه اینه:
```
چت سرور روی پورت 9000 شروع شد
کلاینت ('127.0.0.1', 12345) وصل شد
کلاینت ('127.0.0.1', 12346) وصل شد
('127.0.0.1', 12345): Hi everyone!
کلاینت ('127.0.0.1', 12345) قطع شد
```

کلاینت دوم این پیام رو می‌بینه: `('127.0.0.1', 12345): Hi everyone!`

### یه مثال واقعی‌تر
فرض کن یه سرور برای یه اپلیکیشن چت گروهی داری. می‌خوای کاربرا بتونن تو گروه‌ها پیام بفرستن و پیام‌ها برای همه اعضای گروه پخش بشه. کد بالا یه نسخه ساده از این ایده‌ست. برای یه اپ واقعی، می‌تونی گروه‌ها رو با یه دیکشنری مدیریت کنی:

```python
import asyncio

groups = {}  # {group_name: [writers]}

async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    print(f"کلاینت {addr} وصل شد")
    
    # از کلاینت بخواه اسم گروه رو بفرسته
    group_name = (await reader.readline()).decode().strip()
    if group_name not in groups:
        groups[group_name] = []
    groups[group_name].append(writer)
    print(f"کلاینت {addr} به گروه {group_name} پیوست")

    try:
        while True:
            data = await reader.readline()
            if not data:
                break
            message = f"{addr}: {data.decode().strip()}"
            print(message)
            for client in groups[group_name]:
                if client is not writer:
                    client.write(message.encode())
                    await client.drain()
    except Exception as e:
        print(f"خطا برای {addr}: {e}")
    finally:
        print(f"کلاینت {addr} از گروه {group_name} قطع شد")
        groups[group_name].remove(writer)
        writer.close()
        await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 9000)
    print("چت سرور گروهی روی پورت 9000 شروع شد")

    async with server:
        await server.serve_forever()

if __name__ == "__main__":
    asyncio.run(main())
```

اینجا هر کلاینت اول اسم گروهش رو می‌فرسته (مثلاً `room1`) و بعد پیام‌هاش فقط برای همون گروه پخش می‌شه.

---

## مدیریت خطاها و بهینه‌سازی

شبکه دنیای پرخطریه! اتصال ممکنه قطع بشه، کلاینت ممکنه بدون خداحافظی بره، یا داده‌ها نصفه برسن. برای همین باید همیشه آماده خطاها باشی.

### مدیریت خطاهای اتصال
یه نمونه کد که خطاها رو مدیریت می‌کنه:

```python
import asyncio

async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    print(f"کلاینت {addr} وصل شد")

    try:
        while True:
            data = await reader.read(100)
            if not data:
                print(f"کلاینت {addr} قطع شد")
                break
            message = data.decode()
            print(f"دریافت از {addr}: {message}")

            writer.write(data)
            await writer.drain()
    except ConnectionResetError:
        print(f"کلاینت {addr} به‌صورت غیرمنتظره قطع شد")
    except Exception as e:
        print(f"خطا برای {addr}: {e}")
    finally:
        writer.close()
        await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 8888)
    print("سرور روی پورت 8888 شروع شد")

    async with server:
        await server.serve_forever()

if __name__ == "__main__":
    asyncio.run(main())
```

اینجا با `try/except` خطاهایی مثل `ConnectionResetError` رو مدیریت کردیم تا سرور به خاطر یه کلاینت خراب از کار نیفته.

### اضافه کردن Timeout
اگه بخوای مطمئن شی که سرور منتظر کلاینت‌های کند نمی‌مونه، می‌تونی از `asyncio.wait_for` استفاده کنی:

```python
import asyncio

async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    print(f"کلاینت {addr} وصل شد")

    try:
        while True:
            data = await asyncio.wait_for(reader.read(100), timeout=5)
            if not data:
                print(f"کلاینت {addr} قطع شد")
                break
            message = data.decode()
            print(f"دریافت از {addr}: {message}")

            writer.write(data)
            await writer.drain()
    except asyncio.TimeoutError:
        print(f"تایم‌اوت برای {addr}")
    except Exception as e:
        print(f"خطا برای {addr}: {e}")
    finally:
        writer.close()
        await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 8888)
    print("سرور روی پورت 8888 شروع شد")

    async with server:
        await server.serve_forever()

if __name__ == "__main__":
    asyncio.run(main())
```

اینجا اگه کلاینت تو 5 ثانیه داده نفرسته، سرور اتصالش رو قطع می‌کنه.

---

## نکات کلیدی و بهترین روش‌ها

1‏. **همیشه خطاها رو مدیریت کن**  
   شبکه پر از خطاهای غیرمنتظره‌ست. همیشه از `try/except` برای مدیریت خطاهایی مثل `ConnectionResetError` یا `TimeoutError` استفاده کن.

2‏. **از Timeout استفاده کن**  
   با `asyncio.wait_for` می‌تونی جلوی منتظر موندن بی‌نهایت برای کلاینت‌های کند رو بگیری.

3‏. **منابع رو آزاد کن**  
   همیشه مطمئن شو که اتصال‌ها با `writer.close()` و `await writer.wait_closed()` درست بسته می‌شن.

4‏. **از StreamReader/StreamWriter استفاده کن**  
   این ابزارها کار با سوکت‌ها رو خیلی ساده‌تر می‌کنن. مستقیم با سوکت‌های سطح پایین کار نکن، مگر اینکه واقعاً لازم باشه.

5‏. **برای کارهای پیچیده‌تر از کتابخونه‌ها استفاده کن**  
   اگه بخوای یه وب‌سرور یا API بسازی، کتابخونه‌هایی مثل **aiohttp** یا **FastAPI** خیلی از این پیچیدگی‌ها رو برات مدیریت می‌کنن.

---

## سوالات رایج و جواب‌هاشون

**1. تفاوت `reader.read` و `reader.readline` چیه؟**  
`reader.read(n)` تا `n` بایت داده می‌خونه، ولی `reader.readline()` یه خط کامل (تا `\n`) می‌خونه. برای چت‌سرورها معمولاً `readline` بهتره چون پیام‌ها معمولاً خط به خطن.

**2. اگه کلاینت قطع بشه چی میشه؟**  
اگه کلاینت بدون اطلاع قطع بشه، `reader.read` یا `reader.readline` یه داده خالی (`b''`) برمی‌گردونه یا خطای `ConnectionResetError` پرت می‌کنه. همیشه این موارد رو با `try/except` مدیریت کن.

**3. می‌تونم چندتا سرور رو تو یه برنامه اجرا کنم؟**  
بله! می‌تونی چندتا `asyncio.start_server` رو با پورت‌های مختلف تو یه **event loop** اجرا کنی. فقط مطمئن شو که پورت‌ها تداخل نداشته باشن.

**4. چرا از `drain` استفاده می‌کنیم؟**  
`writer.drain()` مطمئن می‌شه که داده‌ها واقعاً از بافر ارسال شدن. اگه اینو نذاری، ممکنه داده‌ها تو بافر بمونن و کلاینت چیزی دریافت نکنه.

**5. برای کارای پیچیده‌تر چیکار کنم؟**  
برای اپلیکیشن‌های واقعی (مثل وب‌سرور یا API)، به جای سوکت‌های خام، از **aiohttp** یا **FastAPI** استفاده کن. اینا خیلی از جزئیات شبکه‌ای رو برات مدیریت می‌کنن.

---

## جمع‌بندی

تو این درس یاد گرفتیم چطور با **asyncio** سرور و کلاینت‌های شبکه‌ای بسازیم که بتونن چندین اتصال رو همزمان مدیریت کنن. دیدیم که:
- ‏**StreamReader** و **StreamWriter** چطور کار با شبکه رو ساده می‌کنن.
- چطور یه **echo server** ساده بسازیم که پیام‌ها رو برگردونه.
- چطور یه **چت سرور** بسازیم که پیام‌ها رو بین کلاینت‌ها پخش کنه.
- چطور خطاها و تایم‌اوت‌ها رو مدیریت کنیم تا سرورمون پایدار بمونه.


<p align="center">
<a href="../19-advanced-background-tasks/README.md">درس قبلی</a>
&nbsp; | &nbsp;
<a href="../21-advanced-concurrency-patterns/README.md">درس بعدی</a>
