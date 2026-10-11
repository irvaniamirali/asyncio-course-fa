# دورهٔ AsyncIO به فارسی

این یه دوره‌ی کوتاه و ساده‌ست برای آشنایی با برنامه‌نویسی غیرهمزمان در پایتونه.
هدفش اینه که مفهوم async و await رو بدون پیچوندن یاد بگیری و بتونی توی پروژه‌های واقعی استفاده‌شون کنی.

همه‌چیز به زبان ساده گفته شده، با مثال‌های کم‌حرف و پرمعنی.

---
## پیش‌نیازها

برای شروع این دوره، فقط باید با **پایتون مقدماتی** راحت باشی. یعنی:

- بتونی تابع بنویسی و ازش استفاده کنی.
- با مفهوم تابع بازگشتی، متغیرها، حلقه‌ها و شرط‌ها آشنا باشی.
- بدونی ماژول چیه و چطور `import` می‌کنن.
- با ساختار فایل‌های `.py` و اجرای برنامه‌ها از ترمینال یا VS Code آشنا باشی.

اگر پایتون رو قبلاً تا سطح مقدماتی یاد گرفتی، همین کافیه. نیازی به تجربهٔ پیشرفته یا آشنایی با threading، شبکه یا async نداری. اون‌ها رو توی همین دوره قدم‌به‌قدم یاد می‌گیری.

---
## شروع دوره

- [درس اول: مقدمه‌ای بر برنامه‌نویسی غیرهمزمان](./lessons/01-intro/README.md)

---

## فهرست درس‌ها

1. [مقدمه‌ای بر برنامه‌نویسی غیرهمزمان](./lessons/01-intro/README.md)
2. [مفهوم انتظار و زمان بیکاری در برنامه‌نویسی](./lessons/02-idle-time/README.md)
3. [مفهوم وظیفه (Task) و حلقه‌ی رویداد (Event Loop)](./lessons/03-tasks-and-event-loop/README.md)
4. [async و await دقیقاً چطور کار می‌کنن](./lessons/04-async-await/README.md)
5. [دانلود هم‌زمان چند فایل با asyncio](./lessons/05-download-files/README.md)
6. [ماجرای event loop و چطور کار می‌کنه](./lessons/06-event-loop-deepdive/README.md)
7. [کار با ورودی و خروجی غیرهمزمان (Async I/O در عمل)](./lessons/07-async-io-in-practice/README.md)
8. [مدیریت تسک‌ها و خطاها در asyncio](./lessons/08-task-management-and-errors/README.md)
9. [طراحی و ساختاردهی برنامه‌های async در پایتون](./lessons/09-structuring-async-programs/README.md)
10. [تست‌نویسی برای کدهای async در پایتون](./lessons/10-testing-async-code/README.md)
11. [ساخت یک downloader هم‌زمان با asyncio](./lessons/11-async-downloader/README.md)
12. [مدیریت هم‌زمانی با Queue، Lock و Semaphore در asyncio](./lessons/12-sync-primitives/README.md)
13. [اجرای کدهای بلاک‌شونده در برنامه‌های Async](./lessons/13-blocking-calls-in-async/README.md)
14. [مدیریت زمان و تایم‌اوت در asyncio](./lessons/14-timeouts-and-time-management/README.md)
15. [لغو تسک‌ها و پاکسازی منابع در asyncio](./lessons/15-task-cancellation/README.md)
16. [صف‌های همزمان (Async Queues) و الگوی Producer–Consumer](./lessons/16-async-queues/README.md)
17. [صف‌های اولویت‌دار (PriorityQueue) و زمان‌بندی کارها](./lessons/17-priority-queue/README.md)
18. [اجرای دوره‌ای تسک‌ها و زمان‌بندی (Periodic Tasks & Scheduling)](./lessons/18-periodic-tasks/README.md)
19. [طراحی سیستم‌های پیشرفته background tasks در asyncio](./lessons/19-advanced-background-tasks/README.md)
20. [ورودی و خروجی شبکه‌ای به‌صورت ناهمگام (Async Network I/O)](./lessons/20-async-network-io/README.md)
21. [الگوهای پیشرفته هم‌زمانی — Fan-in / Fan-out، Pipeline، Queue و Event](./lessons/21-advanced-concurrency-patterns/README.md)


---

## مشارکت

این پروژه بازه و مشارکت توش آزاده. اگر ایرادی دیدی یا پیشنهادی داشتی، خوشحال می‌شم که پول‌ریکوئست بدی یا توی Issues بنویسی.

---

## نکته

کدها و مثال‌ها با **Python 3.12+** تست شدن، پس قبل از اجرا مطمئن شو نسخه‌ت به‌روز باشه.
