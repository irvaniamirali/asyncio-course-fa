# اجرای کدهای بلاک‌شونده در برنامه‌های Async

تصور کن یه رستوران شلوغ داری. گارسون‌ها (coroutineها) دارن سریع بین مشتری‌ها می‌چرخن، سفارش می‌گیرن و غذا سرو می‌کنن. حالا اگه یه گارسون وسط کار وایسه و شروع کنه به درست کردن یه غذای پیچیده که نیم ساعت طول می‌کشه، چی میشه؟ بقیه گارسون‌ها نمی‌تونن کارشون رو ادامه بدن، مشتری‌ها عصبانی می‌شن و کل سیستم به هم می‌ریزه!

توی **asyncio** هم همین‌طوره. اگه یه تابع بلاک‌شونده (مثل یه درخواست HTTP با کتابخونه **requests**) رو مستقیم تو یه coroutine صدا کنی، **event loop** قفل می‌کنه. یعنی تا وقتی اون تابع تموم نشه، هیچ coroutine دیگه‌ای نمی‌تونه اجرا بشه. این یعنی خداحافظ مزیت‌های برنامه‌نویسی ناهم‌زمان!

مثلاً کتابخونه **requests** بلاک‌شونده‌ست. اگه تو یه coroutine مستقیم ازش استفاده کنی، انگار کل برنامه رو روی حالت pause گذاشتی. پس باید راهی پیدا کنیم که این کارها رو بدون قفل کردن **event loop** انجام بدیم.

---

## ابزارهایی که تو جعبه‌ابزارمون داریم

برای حل این مشکل، **asyncio** و پایتون چندتا ابزار باحال بهمون می‌دن:

1. **`asyncio.get_running_loop().run_in_executor(...)`**

این روش بهت اجازه می‌ده یه تابع بلاک‌شونده رو تو یه **thread** یا **process** جداگونه اجرا کنی و **event loop** رو آزاد نگه داری.

3. **`asyncio.to_thread(...)`**

یه روش ساده‌تر و مدرن‌تر (از پایتون 3.9 به بعد) برای اجرای توابع بلاک‌شونده تو یه **thread**. برای کارهای ساده خیلی خوبه.

5. **`concurrent.futures.ThreadPoolExecutor`**

این ابزار بهت اجازه می‌ده یه مجموعه از **threadها** رو مدیریت کنی و تعدادشون رو کنترل کنی. برای کارهای **I/O-bound** مثل درخواست‌های شبکه‌ای عالیه.

7. **`concurrent.futures.ProcessPoolExecutor`**

برای کارهای سنگین **CPU-bound** (مثل پردازش تصویر یا محاسبات پیچیده) که به خاطر **GIL** (Global Interpreter Lock) نمی‌تونن تو **thread** خوب کار کنن، از این استفاده می‌کنیم.

حالا بیایم هر کدوم رو با جزئیات و مثال‌های واقعی بررسی کنیم.

---

## الگوی پایه: استفاده از `run_in_executor`

این روش مثل اینه که یه کار سنگین رو به یه کارگر دیگه بسپری و خودت بری بقیه کارات رو انجام بدی. با `run_in_executor` می‌تونی یه تابع بلاک‌شونده رو تو یه **thread** یا **process** جداگونه اجرا کنی و **event loop** رو آزاد نگه داری.

### یه مثال ساده
فرض کن می‌خوای از چندتا وبسایت با کتابخونه **requests** داده بگیری. چون **requests** بلاک‌شونده‌ست، باید از `run_in_executor` استفاده کنیم.

```python
import asyncio
import requests

def blocking_http_get(url):
    print(f"Fetching {url}")
    resp = requests.get(url)
    return resp.text

async def fetch(url):
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, blocking_http_get, url)
    return result

async def main():
    urls = ["https://example.com" for _ in range(5)]
    results = await asyncio.gather(*(fetch(u) for u in urls))
    print(f"Fetched {len(results)} pages")

if __name__ == "__main__":
    asyncio.run(main())
```

### این کد چیکار می‌کنه؟
1. تابع `blocking_http_get` یه درخواست HTTP بلاک‌شونده با **requests** انجام می‌ده.
2. توی تابع `fetch`، از `loop.run_in_executor` استفاده می‌کنیم تا این تابع رو تو یه **thread** جداگونه اجرا کنیم.
3. آرگومان `None` یعنی از **ThreadPoolExecutor** پیش‌فرض پایتون استفاده کن.
4. چون از `await` استفاده کردیم، **event loop** منتظر می‌مونه تا نتیجه برگرده، ولی قفل نمی‌شه و بقیه coroutineها می‌تونن اجرا بشن.
5. توی `main`، با `asyncio.gather` چندتا درخواست همزمان می‌فرستیم.

### خروجی چطوره؟
خروجی چیزی شبیه اینه:
```
Fetching https://example.com
Fetching https://example.com
Fetching https://example.com
Fetching https://example.com
Fetching https://example.com
Fetched 5 pages
```

همه درخواست‌ها به صورت موازی تو **threadهای** جدا اجرا می‌شن و **event loop** آزاد می‌مونه.

### یه مثال واقعی‌تر
فرض کن یه برنامه داری که قراره از چندتا API مختلف اطلاعات آب‌وهوا بگیره و نتایج رو نمایش بده. چون APIها با **requests** کار می‌کنن، باید از `run_in_executor` استفاده کنیم.

```python
import asyncio
import requests

def get_weather(city):
    print(f"Getting weather for {city}")
    url = f"http://api.example.com/weather?city={city}"
    resp = requests.get(url)
    return resp.json()

async def fetch_weather(city):
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, get_weather, city)
    return result

async def main():
    cities = ["Tehran", "Shiraz", "Mashhad", "Isfahan"]
    results = await asyncio.gather(*(fetch_weather(city) for city in cities))
    for city, data in zip(cities, results):
        print(f"Weather in {city}: {data}")

if __name__ == "__main__":
    asyncio.run(main())
```

این کد اطلاعات آب‌وهوا رو به صورت موازی می‌گیره بدون اینکه **event loop** قفل بشه.

---

## روش مدرن: استفاده از `asyncio.to_thread`

اگه از پایتون 3.9 یا جدیدتر استفاده می‌کنی، یه روش ساده‌تر به اسم `asyncio.to_thread` داری که کار `run_in_executor` رو برای **threadها** راحت‌تر می‌کنه. این روش برای کارهای **I/O-bound** که نیازی به کنترل پیچیده **ThreadPool** ندارن عالیه.

### یه مثال ساده
بیایم همون مثال قبلی رو با `to_thread` بازنویسی کنیم:

```python
import asyncio
import requests

def blocking_http_get(url):
    print(f"Fetching {url}")
    resp = requests.get(url)
    return resp.text

async def fetch(url):
    result = await asyncio.to_thread(blocking_http_get, url)
    return result

async def main():
    urls = ["https://example.com" for _ in range(5)]
    results = await asyncio.gather(*(fetch(u) for u in urls))
    print(f"Fetched {len(results)} pages")

if __name__ == "__main__":
    asyncio.run(main())
```

### این کد چیکار می‌کنه؟
1. به جای `run_in_executor`، از `asyncio.to_thread` استفاده کردیم که ساده‌تر و خواناترِ.
2. این روش خودش تابع بلاک‌شونده رو تو یه **thread** جدا اجرا می‌کنه.
3. بقیه داستان مثل قبلِ: **event loop** آزاد می‌مونه و درخواست‌ها موازی انجام می‌شن.

### چرا این روش بهتره؟
- کد تمیزتر و کوتاه‌تره.
- نیازی به دستی گرفتن **loop** نیست.
- برای کارهای ساده **I/O-bound** مثل درخواست‌های شبکه‌ای خیلی مناسبه.

### یه مثال واقعی
فرض کن یه برنامه داری که قراره از یه API قیمت سهام چند شرکت رو بگیره و نمایش بده. چون API با **requests** کار می‌کنه، از `to_thread` استفاده می‌کنیم.

```python
import asyncio
import requests

def get_stock_price(symbol):
    print(f"Fetching price for {symbol}")
    url = f"http://api.example.com/stocks?symbol={symbol}"
    resp = requests.get(url)
    return resp.json()

async def fetch_stock(symbol):
    result = await asyncio.to_thread(get_stock_price, symbol)
    return result

async def main():
    stocks = ["AAPL", "GOOGL", "TSLA", "MSFT"]
    results = await asyncio.gather(*(fetch_stock(s) for s in stocks))
    for stock, data in zip(stocks, results):
        print(f"Price of {stock}: {data}")

if __name__ == "__main__":
    asyncio.run(main())
```

این کد قیمت سهام رو به صورت موازی می‌گیره و **event loop** رو آزاد نگه می‌داره.

---

## کنترل تعداد Threadها با `ThreadPoolExecutor`

تا حالا دیدیم که می‌تونیم از **ThreadPoolExecutor** پیش‌فرض پایتون استفاده کنیم (با `None` تو `run_in_executor`). اما اگه بخوای تعداد **threadها** رو کنترل کنی چی؟ مثلاً نمی‌خوای سرور یا سیستم خودت رو با 100 تا درخواست همزمان نابود کنی! اینجا می‌تونی یه **ThreadPoolExecutor** با تعداد **worker** محدود درست کنی.

### یه مثال ساده
فرض کن می‌خوای 50 تا وبسایت رو بخونی، ولی نمی‌خوای بیشتر از 10 تا **thread** همزمان اجرا بشه.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import requests

executor = ThreadPoolExecutor(max_workers=10)

def blocking_http_get(url):
    print(f"Fetching {url}")
    return requests.get(url).text

async def fetch(url):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(executor, blocking_http_get, url)

async def main():
    urls = [f"https://example.com?page={i}" for i in range(50)]
    results = await asyncio.gather(*(fetch(u) for u in urls))
    print(f"Fetched {len(results)} pages")

if __name__ == "__main__":
    asyncio.run(main())
    executor.shutdown()
```

### این کد چیکار می‌کنه؟
1. یه **ThreadPoolExecutor** با حداکثر 10 تا **worker** درست کردیم.
2. تابع `blocking_http_get` تو یه **thread** از این **executor** اجرا می‌شه.
3. با `max_workers=10` مطمئن می‌شیم که بیشتر از 10 تا درخواست همزمان به سرور نمی‌ره.
4. بعد از تموم شدن کار، با `executor.shutdown()` منابع رو آزاد می‌کنیم.

### یه مثال واقعی
فرض کن یه وبسایت داری که قراره نظرات کاربرها رو از یه API بگیره و تو دیتابیس ذخیره کنه. چون API با **requests** کار می‌کنه و نمی‌خوای سرور API رو تحت فشار بذاری، تعداد **threadها** رو محدود می‌کنی.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import requests

executor = ThreadPoolExecutor(max_workers=5)

def fetch_comments(post_id):
    print(f"Fetching comments for post {post_id}")
    url = f"http://api.example.com/comments?post_id={post_id}"
    return requests.get(url).json()

async def process_comments(post_id):
    loop = asyncio.get_running_loop()
    comments = await loop.run_in_executor(executor, fetch_comments, post_id)
    print(f"Processed {len(comments)} comments for post {post_id}")
    return comments

async def main():
    post_ids = list(range(1, 21))
    results = await asyncio.gather(*(process_comments(pid) for pid in post_ids))
    print(f"Total posts processed: {len(results)}")

if __name__ == "__main__":
    asyncio.run(main())
    executor.shutdown()
```

این کد مطمئن می‌شه که بیشتر از 5 تا درخواست همزمان به API نمی‌ره و سیستم رو سبک نگه می‌داره.

---

## کارهای سنگین CPU با `ProcessPoolExecutor`

تا حالا بیشتر درباره کارهای **I/O-bound** (مثل درخواست‌های شبکه‌ای) حرف زدیم. اما اگه بخوای یه کار سنگین **CPU-bound** انجام بدی چی؟ مثلاً پردازش تصویر، محاسبات ریاضی پیچیده یا رمزنگاری. اینجا **threadها** به خاطر **GIL** (Global Interpreter Lock) توی پایتون خوب کار نمی‌کنن. پس باید از **ProcessPoolExecutor** استفاده کنیم که کارها رو تو **processهای** جداگونه اجرا می‌کنه.

### یه مثال ساده
فرض کن یه تابع داری که یه محاسبه سنگین ریاضی انجام می‌ده. می‌خوای چندتا از این محاسبات رو همزمان انجام بدی.

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def heavy_compute(x):
    print(f"Computing for {x}")
    s = 0
    for i in range(10_000_000):
        s += (i * x) % 7
    return s

async def main():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        tasks = [loop.run_in_executor(pool, heavy_compute, i) for i in range(4)]
        results = await asyncio.gather(*tasks)
        print(f"Results: {results}")

if __name__ == "__main__":
    asyncio.run(main())
```

### این کد چیکار می‌کنه؟
1. تابع `heavy_compute` یه محاسبه سنگین **CPU-bound** انجام می‌ده.
2. با **ProcessPoolExecutor**، این تابع تو **processهای** جداگونه اجرا می‌شه که **GIL** رو دور می‌زنن.
3. نتایج با `asyncio.gather` جمع‌آوری می‌شن و نمایش داده می‌شن.

### یه نکته مهم
ارسال داده بین **processها** (برخلاف **threadها**) هزینه داره، چون هر **process** حافظه جداگونه‌ای داره. پس برای کارهای خیلی سبک، **ProcessPoolExecutor** ممکنه بهینه نباشه.

### یه مثال واقعی
فرض کن یه برنامه داری که قراره یه سری عکس رو پردازش کنه (مثلاً اندازه‌شون رو تغییر بده). این کار **CPU-bound** هست و با **ProcessPoolExecutor** بهتر انجام می‌شه.

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from PIL import Image
import io

def resize_image(image_data):
    print("Resizing image")
    img = Image.open(io.BytesIO(image_data))
    img = img.resize((100, 100))
    buffer = io.BytesIO()
    img.save(buffer, format="PNG")
    return buffer.getvalue()

async def process_image(image_data):
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, resize_image, image_data)
    return result

async def main():
    # شبیه‌سازی داده‌های عکس
    image_data = open("sample.png", "rb").read()
    images = [image_data for _ in range(4)]
    with ProcessPoolExecutor() as pool:
        tasks = [loop.run_in_executor(pool, resize_image, img) for img in images]
        results = await asyncio.gather(*tasks)
        print(f"Processed {len(results)} images")

if __name__ == "__main__":
    loop = asyncio.get_running_loop()
    asyncio.run(main())
```

این کد چندتا عکس رو به صورت موازی پردازش می‌کنه و **event loop** رو آزاد نگه می‌داره.

---

## ترکیب Queue و ThreadPool: یه سیستم کامل

حالا بیایم یه مثال ترکیبی بزنیم که هم از **Queue** و هم از **ThreadPoolExecutor** استفاده کنه. این الگو وقتی مفیده که یه سری داده تولید می‌کنی (مثلاً لینک‌های دانلود) و می‌خوای با یه کتابخونه بلاک‌شونده (مثل **requests**) پردازششون کنی.

### مثال
فرض کن یه برنامه داری که قراره یه سری فایل PDF از اینترنت دانلود کنه و تعداد صفحات هر کدوم رو بشمره. دانلود با **requests** انجام می‌شه و تعداد صفحات با یه کتابخونه بلاک‌شونده (مثل **PyPDF2**).

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import requests
from PyPDF2 import PdfReader
import io

executor = ThreadPoolExecutor(max_workers=5)

def count_pdf_pages(url):
    print(f"Processing {url}")
    resp = requests.get(url)
    pdf = PdfReader(io.BytesIO(resp.content))
    return len(pdf.pages)

async def producer(queue):
    urls = [f"http://example.com/pdf/{i}.pdf" for i in range(20)]
    for url in urls:
        await queue.put(url)
    for _ in range(5):  # برای 5 تا مصرف‌کننده
        await queue.put(None)

async def consumer(queue):
    loop = asyncio.get_running_loop()
    while True:
        url = await queue.get()
        if url is None:
            break
        page_count = await loop.run_in_executor(executor, count_pdf_pages, url)
        print(f"{url} has {page_count} pages")
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    producers = [producer(queue)]
    consumers = [consumer(queue) for _ in range(5)]
    await asyncio.gather(*(producers + consumers))

if __name__ == "__main__":
    asyncio.run(main())
    executor.shutdown()
```

### این کد چیکار می‌کنه؟
1. تابع `producer` یه سری لینک PDF تولید می‌کنه و تو **Queue** می‌ذاره.
2. تابع `consumer` لینک‌ها رو از **Queue** می‌گیره و با `run_in_executor` تابع بلاک‌شونده `count_pdf_pages` رو اجرا می‌کنه.
3. تابع `count_pdf_pages` فایل PDF رو دانلود می‌کنه و تعداد صفحاتش رو برمی‌گردونه.
4. با `max_workers=5` مطمئن می‌شیم که بیشتر از 5 تا دانلود همزمان انجام نمی‌شه.
5. **Queue** هماهنگی بین تولیدکننده و مصرف‌کننده‌ها رو مدیریت می‌کنه.

### خروجی چطوره؟
خروجی چیزی شبیه اینه:
```
Processing http://example.com/pdf/1.pdf
Processing http://example.com/pdf/2.pdf
Processing http://example.com/pdf/3.pdf
http://example.com/pdf/1.pdf has 10 pages
Processing http://example.com/pdf/4.pdf
http://example.com/pdf/2.pdf has 15 pages
...
```

---

## نکات مهم درباره Cancellation و استثناها

‏2. **Cancellation چی میشه؟**  

اگه یه task رو کنسل کنی (مثلاً با `task.cancel()`)، تابع بلاک‌شونده‌ای که تو **thread** یا **process** اجرا می‌شه متوقف نمی‌شه! فقط **await** توی **event loop** یه `CancelledError` پرت می‌کنه. برای توابع حساس، باید خودت مکانیسمی بذاری که بتونه تابع بلاک‌شونده رو متوقف کنه (مثلاً یه پرچم یا timeout).

2‏. **مدیریت استثناها**  

اگه تابع بلاک‌شونده خطایی بندازه، این خطا به شکل یه استثنا تو **event loop** بالا میاد. پس همیشه با `try/except` آماده باش.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=1)

def risky_function():
    raise ValueError("Something went wrong!")

async def main():
    loop = asyncio.get_running_loop()
    try:
        await loop.run_in_executor(executor, risky_function)
    except ValueError as e:
        print(f"Caught error: {e}")

if __name__ == "__main__":
    asyncio.run(main())
    executor.shutdown()
```

این کد نشون می‌ده چطور خطای تابع بلاک‌شونده رو مدیریت کنیم.

---

## توصیه‌ها و بهترین روش‌ها

1‏. **اولویت با کتابخونه‌های async**  
   اگه کتابخونه async برای کارت هست (مثل **aiohttp** به جای **requests**)، همیشه از اونا استفاده کن. اینطوری اصلاً نیازی به **thread** یا **process** نداری.

2‏. **برای کارهای ساده از `to_thread` استفاده کن**  
   اگه کار **I/O-bound** داری و نیازی به کنترل پیچیده **threadها** نیست، `asyncio.to_thread` ساده‌ترین و تمیزترین راهه.

3‏. **برای کارهای CPU-bound از `ProcessPoolExecutor`**  
   برای محاسبات سنگین که **GIL** اذیتت می‌کنه، **ProcessPoolExecutor** بهترین انتخابه.

4‏. **تعداد thread/process رو محدود کن**  
   همیشه با `max_workers` تعداد **threadها** یا **processها** رو کنترل کن تا سیستم یا سرور تحت فشار نره.

5‏. **منابع رو آزاد کن**  
   بعد از اتمام کار، حتماً **executor** رو با `shutdown()` یا context manager (`with`) ببند تا منابع سیستم آزاد بشن.

6‏. **به cancellation فکر کن**  
   اگه تابع بلاک‌شونده‌ات حساسه، یه مکانیسم برای توقفش طراحی کن (مثلاً یه پرچم یا timeout).

---

## سوالات رایج و جواب‌هاشون

**1. اگه از `run_in_executor` استفاده نکنم چی میشه؟**  
اگه تابع بلاک‌شونده رو مستقیم تو coroutine صدا کنی، **event loop** قفل می‌کنه و بقیه coroutineها نمی‌تونن اجرا بشن. انگار کل برنامه رو sync کردی!

**2. تفاوت `to_thread` و `run_in_executor` چیه؟**  
`to_thread` فقط برای **threadها** و ساده‌تره. `run_in_executor` می‌تونه هم برای **thread** و هم برای **process** استفاده بشه و کنترل بیشتری می‌ده.

**3. کی از `ProcessPoolExecutor` استفاده کنم؟**  
برای کارهای **CPU-bound** مثل پردازش تصویر، محاسبات سنگین یا رمزنگاری. برای کارهای **I/O-bound** مثل درخواست شبکه‌ای، **ThreadPoolExecutor** کافیه.

**4. اگه تعداد `max_workers` رو نذارم چی میشه؟**  
پایتون یه تعداد پیش‌فرض برای **threadها** یا **processها** انتخاب می‌کنه که ممکنه برای سیستمت زیاد یا کم باشه. بهتره خودت با `max_workers` کنترلش کنی.

**5. می‌تونم یه تابع async رو تو `run_in_executor` اجرا کنم؟**  
نه، `run_in_executor` فقط برای توابع معمولی (sync) طراحی شده. توابع **async** رو باید با `await` تو **event loop** اجرا کنی.



<p align="center">
<a href="../12-sync-primitives/README.md">درس قبلی</a>
&nbsp; | &nbsp;
<a href="../14-timeouts-and-time-management/README.md">درس بعدی</a>
