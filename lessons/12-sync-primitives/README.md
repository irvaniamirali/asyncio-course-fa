# مدیریت هم‌زمانی با Queue، Lock و Semaphore در asyncio

تصور کن چند تا کارمند تو یه شرکت دارن همزمان روی یه فایل اکسل مشترک کار می‌کنن. اگه همه با هم شروع کنن به ویرایش، چی میشه؟ یه آشوب کامل! یکی سلول A1 رو تغییر میده، یکی دیگه همون موقع می‌نویسه روش، آخرش معلوم نیست چی به چیه! تو برنامه‌نویسی هم همین‌طوره. وقتی چند تا coroutine بخوان روی یه منبع مشترک (مثل یه متغیر، فایل یا دیتابیس) کار کنن، باید یه جوری هماهنگشون کنیم که به همدیگه گند نزنن!

اینجاست که ابزارهای **Lock**، **Semaphore** و **Queue** تو **asyncio** به دادمون می‌رسن. تو این درس، می‌خوام این ابزارها رو با جزئیات و مثال‌های ملموس و جذاب براتون توضیح بدم. آماده‌این؟ بریم!

---

## قفل (Lock): نگهبان منابع مشترک

فرض کن یه آشپزخونه داری که فقط یه اجاق گاز داره. حالا سه تا آشپز می‌خوان همزمان غذا درست کنن، ولی اگه همه با هم بیان سراغ اجاق، ممکنه یکی سوپش رو بریزه تو قابلمه‌ی کباب اون یکی! راه‌حل چیه؟ یه قانون می‌ذاری: فقط یه نفر می‌تونه از اجاق استفاده کنه. بقیه باید منتظر بمونن تا نوبتشون بشه. تو **asyncio**، این قانون رو با **Lock** پیاده می‌کنیم.

‏**Lock** مثل یه قفل واقعی عمل می‌کنه. وقتی یه coroutine قفل رو می‌گیره، بقیه coroutineها باید صبر کنن تا قفل آزاد بشه. اینجوری مطمئن می‌شیم که فقط یه coroutine تو یه لحظه داره روی منبع مشترک کار می‌کنه.

### یه مثال واقعی
فرض کن یه برنامه داری که یه متغیر مشترک به اسم `counter` داره و چند تا coroutine می‌خوان این متغیر رو افزایش بدن. اگه همشون با هم بیان و بخونن و بنویسن، ممکنه به مشکل بربخوری. مثلاً یکی `counter` رو می‌خونه (مثلاً 5)، بعد قبل از اینکه بنویسه 6، یه coroutine دیگه می‌خونه (دوباره 5) و می‌نویسه 6. آخرش به جای اینکه `counter` بشه 7، می‌مونه روی 6! اینجاست که **Lock** میاد وسط.

```python
import asyncio

counter = 0
lock = asyncio.Lock()

async def increase(name):
    global counter
    async with lock:
        print(f"{name} entered the critical section")
        temp = counter
        await asyncio.sleep(0.1)  # شبیه‌سازی یه کار زمان‌بر
        counter = temp + 1
        print(f"{name} changed counter to {counter}")

async def main():
    tasks = [
        increase("Task-1"),
        increase("Task-2"),
        increase("Task-3"),
        increase("Task-4")
    ]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

### این کد چیکار می‌کنه؟
1. یه متغیر مشترک به اسم `counter` داریم که از صفر شروع می‌شه.
2. یه **Lock** تعریف کردیم که قراره از دسترسی همزمان جلوگیری کنه.
3. توی تابع `increase`، با استفاده از `async with lock` مطمئن می‌شیم فقط یه coroutine تو یه لحظه می‌تونه `counter` رو تغییر بده.
4. هر coroutine اول `counter` رو می‌خونه، یه کم صبر می‌کنه (با `sleep`)، بعد مقدارش رو افزایش میده.
5. چون از **Lock** استفاده کردیم، coroutineها یکی‌یکی وارد بخش بحرانی (critical section) می‌شن و نتیجه همیشه درست خواهد بود.

### خروجی این کد چطوره؟
خروجی چیزی شبیه اینه:
```
Task-1 entered the critical section
Task-1 changed counter to 1
Task-2 entered the critical section
Task-2 changed counter to 2
Task-3 entered the critical section
Task-3 changed counter to 3
Task-4 entered the critical section
Task-4 changed counter to 4
```

بدون **Lock**، ممکن بود خروجی به هم بریزه، مثلاً دو تا coroutine همزمان `counter` رو بخونن و مقدارش اشتباه بشه.

### یه مثال ملموس‌تر
تصور کن یه وبسایت داری که کاربرها می‌تونن بلیط کنسرت بخرن. فقط 100 تا بلیط موجوده. اگه 200 نفر همزمان درخواست بدن، ممکنه به جای 100 تا بلیط، 150 تا بفروشی! اینجا می‌تونی از **Lock** استفاده کنی تا مطمئن شی فقط یه coroutine تو یه لحظه تعداد بلیط‌ها رو کم می‌کنه.

```python
import asyncio

tickets = 100
lock = asyncio.Lock()

async def buy_ticket(user):
    global tickets
    async with lock:
        if tickets > 0:
            print(f"{user} is trying to buy a ticket")
            await asyncio.sleep(0.1)  # شبیه‌سازی بررسی پرداخت
            tickets -= 1
            print(f"{user} bought a ticket. Tickets left: {tickets}")
        else:
            print(f"{user} failed. No tickets left!")

async def main():
    tasks = [buy_ticket(f"User-{i}") for i in range(120)]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

این کد مطمئن می‌شه که بیشتر از 100 تا بلیط فروخته نمی‌شه.

---

## صف (Queue): یه خط تولید منظم

حالا فرض کن یه کارخانه داری. یه عده کارگر قطعات رو تولید می‌کنن (تولیدکننده‌ها) و یه عده دیگه این قطعات رو مونتاژ می‌کنن (مصرف‌کننده‌ها). اگه تولیدکننده‌ها قطعات رو بندازن وسط سالن و مصرف‌کننده‌ها هر کدوم یه گوشه دنبال قطعات بگردن، چی میشه؟ یه هرج‌ومرج کامل! راه‌حل چیه؟ یه صف درست می‌کنی که تولیدکننده‌ها قطعات رو تو صف بذارن و مصرف‌کننده‌ها یکی‌یکی ازش بردارن.

توی **asyncio**، این صف رو با **Queue** پیاده می‌کنیم. **Queue** یه راه عالیه برای مدیریت ارتباط بین تولیدکننده‌ها و مصرف‌کننده‌ها تو برنامه‌های ناهم‌زمان.

### یه مثال ساده
فرض کن یه برنامه داری که یه سری داده (مثلاً اعداد) تولید می‌کنه و یه سری coroutine دیگه این داده‌ها رو پردازش می‌کنن. با **Queue** می‌تونی این ارتباط رو خیلی تمیز مدیریت کنی.

```python
import asyncio

async def producer(queue):
    for i in range(5):
        print(f"Producer is creating item {i}")
        await queue.put(i)  # اضافه کردن به صف
        await asyncio.sleep(0.2)  # شبیه‌سازی تولید زمان‌بر
    await queue.put(None)  # علامت پایان

async def consumer(queue):
    while True:
        item = await queue.get()  # گرفتن از صف
        if item is None:  # اگه علامت پایان بود، تموم کن
            queue.task_done()
            break
        print(f"Consumer is processing item {item}")
        await asyncio.sleep(0.3)  # شبیه‌سازی پردازش زمان‌بر
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    await asyncio.gather(producer(queue), consumer(queue))

asyncio.run(main())
```

### این کد چیکار می‌کنه؟
1. یه **Queue** درست کردیم که مثل یه خط تولیده.
2. تابع `producer` اعداد 0 تا 4 رو تولید می‌کنه و می‌ذاره تو صف.
3. تابع `consumer` از صف داده‌ها رو برمی‌داره و پردازش می‌کنه.
4. وقتی `producer` کارش تموم شد، یه `None` می‌ذاره تو صف تا به `consumer` بگه دیگه داده‌ای نیست.
5. متد `task_done()` به صف می‌گه که این آیتم پردازش شده و می‌تونه از حافظه آزاد بشه.

### خروجی این کد چطوره؟
خروجی چیزی شبیه اینه:
```
Producer is creating item 0
Producer is creating item 1
Consumer is processing item 0
Producer is creating item 2
Consumer is processing item 1
Producer is creating item 3
Consumer is processing item 2
Producer is creating item 4
Consumer is processing item 3
Consumer is processing item 4
```

### یه مثال واقعی‌تر
تصور کن یه برنامه داری که قراره یه سری فایل رو از اینترنت دانلود کنه و بعد هر کدوم رو پردازش کنه (مثلاً ازشون متن استخراج کنه). می‌تونی یه **Queue** درست کنی که لینک‌های دانلود رو توش بذاری و یه سری coroutine دیگه این لینک‌ها رو بگیرن و پردازش کنن.

```python
import asyncio

async def downloader(queue):
    urls = [
        "http://example.com/file1.pdf",
        "http://example.com/file2.pdf",
        "http://example.com/file3.pdf"
    ]
    for url in urls:
        print(f"Downloader is adding {url} to queue")
        await queue.put(url)
        await asyncio.sleep(0.5)  # شبیه‌سازی دانلود
    await queue.put(None)

async def processor(queue):
    while True:
        url = await queue.get()
        if url is None:
            queue.task_done()
            break
        print(f"Processor is extracting text from {url}")
        await asyncio.sleep(1)  # شبیه‌سازی پردازش
        queue.task_done()

async def main():
    queue = asyncio.Queue()
    await asyncio.gather(downloader(queue), processor(queue))

asyncio.run(main())
```

این کد یه خط تولید تمیز درست می‌کنه که دانلود و پردازش فایل‌ها رو منظم انجام میده.

---

## شمارنده هم‌زمانی (Semaphore): کنترل تعداد کارگرها

حالا فرض کن تو همون آشپزخونه، به جای یه اجاق گاز، سه تا اجاق داری. یعنی می‌تونی به سه تا آشپز اجازه بدی همزمان کار کنن، ولی اگه بیشتر از سه تا باشن، آشپزخونه به هم می‌ریزه. اینجا **Semaphore** به کار میاد. **Semaphore** مثل یه نگهبان باهوشه که به تعداد مشخصی coroutine اجازه می‌ده همزمان وارد یه بخش بشن.

‏**Lock** فقط به یه coroutine اجازه می‌داد، ولی **Semaphore** می‌تونه به چند تا (مثلاً 3 تا) اجازه بده.

### یه مثال ساده
فرض کن یه برنامه داری که قراره یه سری فایل رو دانلود کنه، ولی نمی‌خوای سرور بیش از حد تحت فشار بره. پس تصمیم می‌گیری حداکثر سه تا دانلود همزمان انجام بشه.

```python
import asyncio

semaphore = asyncio.Semaphore(3)

async def download_file(file_id):
    async with semaphore:
        print(f"Starting download for file {file_id}")
        await asyncio.sleep(1)  # شبیه‌سازی دانلود
        print(f"Finished downloading file {file_id}")

async def main():
    tasks = [download_file(i) for i in range(10)]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

### این کد چیکار می‌کنه؟
۱. یه **Semaphore** تعریف کردیم که حداکثر به 3 تا coroutine اجازه می‌ده همزمان اجرا بشن.

۲. تابع `download_file` برای هر فایل یه دانلود شبیه‌سازی می‌کنه.

۳. با `async with semaphore` مطمئن می‌شیم که تو هر لحظه حداکثر 3 تا دانلود در حال اجرا باشن.

۴. بقیه coroutineها منتظر می‌مونن تا یکی از دانلودها تموم بشه.

### خروجی این کد چطوره؟
خروجی چیزی شبیه اینه:
```
Starting download for file 0
Starting download for file 1
Starting download for file 2
Finished downloading file 0
Starting download for file 3
Finished downloading file 1
Starting download for file 4
Finished downloading file 2
Starting download for file 5
...
```

### یه مثال واقعی‌تر
فرض کن یه وبسایت داری که کاربرها می‌تونن آپلود فایل کنن، ولی سرورت فقط می‌تونه همزمان 5 تا آپلود رو پردازش کنه. اینجا **Semaphore** بهت کمک می‌کنه.

```python
import asyncio

semaphore = asyncio.Semaphore(5)

async def upload_file(user_id):
    async with semaphore:
        print(f"User {user_id} is uploading file")
        await asyncio.sleep(2)  # شبیه‌سازی آپلود
        print(f"User {user_id} finished uploading")

async def main():
    tasks = [upload_file(i) for i in range(20)]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

این کد مطمئن می‌شه که سرور بیشتر از 5 تا آپلود همزمان پردازش نمی‌کنه.

---

## نکته‌های کلیدی و ترفندها

۱. **کی از Lock استفاده کنیم؟**
   وقتی یه منبع مشترک داری (مثل یه فایل، متغیر یا دیتابیس) و فقط یه coroutine باید بتونه باهاش کار کنه. اگه چند تا coroutine همزمان بهش دسترسی پیدا کنن، ممکنه داده‌ها خراب بشن.

۲. **کی از Queue استفاده کنیم؟**
   وقتی می‌خوای یه خط تولید و مصرف داده درست کنی. مثلاً یه سری داده تولید می‌کنی (مثل دانلود فایل) و یه سری دیگه باید پردازشش کنن (مثل استخراج متن). **Queue** این ارتباط رو خیلی تمیز مدیریت می‌کنه.

۳. **کی از Semaphore استفاده کنیم؟**
   وقتی می‌خوای تعداد مشخصی coroutine همزمان بتونن یه کار رو انجام بدن، ولی بیشتر از اون نه. مثلاً وقتی می‌خوای تعداد درخواست‌ها به سرور رو محدود کنی.

۴. **مراقب قفل‌های زیاد باش!**
   اگه بیش از حد از **Lock** یا **Semaphore** استفاده کنی، برنامه‌ات کند می‌شه. چون coroutineها باید منتظر بمونن. فقط جایی که واقعاً لازم داری ازشون استفاده کن.

۵. **ترکیب ابزارها**
   گاهی می‌تونی **Lock** و **Queue** یا **Semaphore** رو با هم ترکیب کنی. مثلاً یه **Queue** برای مدیریت داده‌ها و یه **Semaphore** برای کنترل تعداد پردازش‌ها.

---

## سوالات رایج و جواب‌هاشون

**1. اگه از Lock استفاده نکنم چی میشه؟**
بدون **Lock**، اگه چند تا coroutine همزمان روی یه منبع مشترک کار کنن، ممکنه داده‌ها خراب بشن. مثلاً تو مثال `counter`، ممکنه چند تا coroutine همزمان مقدار `counter` رو بخونن و مقدار اشتباه بنویسن. این مشکل به اسم **race condition** شناخته می‌شه.

**2. تفاوت Lock و Semaphore چیه؟**
**Lock** فقط به یه coroutine اجازه می‌ده وارد بخش بحرانی بشه، ولی **Semaphore** می‌تونه به تعداد مشخصی (مثلاً 3 تا) اجازه بده. اگه فقط یه نفر باید کار کنه، **Lock** کافیه. اگه چند نفر می‌تونن همزمان کار کنن، **Semaphore** بهتره.

**3. چرا از Queue استفاده کنیم؟**
**Queue** برای مدیریت ارتباط بین تولیدکننده‌ها و مصرف‌کننده‌ها عالیه. بدون **Queue**، باید خودت هماهنگی بین coroutineها رو مدیریت کنی که خیلی پیچیده و پرخطاست.

**4. می‌تونم بدون این ابزارها کار کنم؟**
بله، ولی فقط اگه مطمئنی که coroutineها به منابع مشترک دسترسی ندارن یا کارشون به هم وابسته نیست. تو اکثر برنامه‌های واقعی، این ابزارها لازم می‌شن.

**5. اگه یه coroutine قفل رو آزاد نکنه چی؟**
اگه از `async with lock` استفاده کنی، **asyncio** خودش قفل رو آزاد می‌کنه وقتی کار تموم شد. ولی اگه دستی با `lock.acquire()` و `lock.release()` کار کنی و یادت بره آزاد کنی، برنامه‌ات قفل می‌کنه (deadlock) و بقیه coroutineها منتظر می‌مونن.

---

## یه مثال ترکیبی برای جمع‌بندی

بیا یه مثال بزنیم که همه این ابزارها رو با هم ترکیب کنه. فرض کن یه سیستم داری که قراره یه سری داده از یه API بگیره، پردازش کنه و توی یه فایل مشترک ذخیره کنه. می‌خوای:
- حداکثر 3 تا درخواست به API همزمان بره (**Semaphore**).
- داده‌ها توی یه صف جمع بشن تا پردازش بشن (**Queue**).
- ذخیره داده‌ها توی فایل با قفل انجام بشه که خراب نشه (**Lock**).

```python
import asyncio

semaphore = asyncio.Semaphore(3)
lock = asyncio.Lock()
queue = asyncio.Queue()

async def fetch_data(task_id):
    async with semaphore:
        print(f"Task {task_id} is fetching data")
        await asyncio.sleep(1)  # شبیه‌سازی درخواست API
        data = f"Data from task {task_id}"
        await queue.put(data)

async def process_data():
    while True:
        data = await queue.get()
        if data is None:
            queue.task_done()
            break
        print(f"Processing {data}")
        async with lock:
            print(f"Writing {data} to file")
            await asyncio.sleep(0.5)  # شبیه‌سازی نوشتن تو فایل
        queue.task_done()

async def main():
    tasks = [fetch_data(i) for i in range(10)] + [process_data()]
    await asyncio.gather(*tasks)
    await queue.put(None)  # علامت پایان برای پردازشگر

asyncio.run(main())
```

این کد یه سیستم کامل رو شبیه‌سازی می‌کنه که داده‌ها رو با نظم و امنیت مدیریت می‌کنه.


<p align="center">
<a href="../11-async-downloader/README.md">درس قبلی</a>
&nbsp; | &nbsp;
<a href="../13-blocking-calls-in-async/README.md">درس بعدی</a>
