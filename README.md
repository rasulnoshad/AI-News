# Russell AI News

این مخزن برای ساخت یک خبرخوان هوشمند و محلی در حوزه هوش مصنوعی طراحی شده است. سیستم منابع منتخب را روزانه بررسی می‌کند، مطالب جدید را استخراج می‌کند، خبرهای نامرتبط و تکراری را کنار می‌گذارد، رویدادها را رتبه‌بندی می‌کند و حداکثر ۱۰ خبر مهم را با خلاصه فارسی، نام منبع و لینک اصلی نمایش می‌دهد.

## وضعیت پروژه

پروژه در مرحله طراحی و مستندسازی است. هنوز کد محصول ایجاد نشده است.

## ترتیب مطالعه مستندات

عامل کدنویس باید فایل‌ها را دقیقاً با این ترتیب بخواند:

1. [مشخصات محصول](docs/01-product-specification.md)
2. [معماری فنی](docs/02-technical-architecture.md)
3. [برنامه پیاده‌سازی](docs/03-implementation-roadmap.md)
4. [راهنمای Codex و Cursor](docs/04-coding-agent-guide.md)

## تصمیم‌های قطعی

- زبان توسعه: Python
- بک‌اند و API: FastAPI
- رابط MVP: Jinja2 و HTML سمت سرور
- دیتابیس MVP: SQLite
- ORM و migration: SQLAlchemy 2 و Alembic
- دریافت RSS: feedparser
- دریافت HTTP: httpx
- استخراج مقاله: trafilatura با fallback محدود مبتنی بر BeautifulSoup
- سرویس هوش مصنوعی: Gemini API
- پیاده‌سازی هوش مصنوعی: `GeminiProvider` با SDK رسمی `google-genai`
- خروجی LLM: JSON ساختاریافته و اعتبارسنجی‌شده با Pydantic
- تعداد پیش‌فرض خبر روزانه: ۱۰
- زبان خلاصه‌ها: فارسی
- منطقه زمانی نمایش: Asia/Tehran
- ذخیره زمان‌ها در دیتابیس: UTC

## اصل توسعه

هر فاز باید مستقل پیاده‌سازی، تست و تأیید شود. عامل کدنویس نباید بدون تکمیل معیارهای پذیرش یک فاز، به فاز بعدی برود.

## آماده‌سازی محیط توسعه

نسخه Python پشتیبانی‌شده در `pyproject.toml` ثبت شده است. دستورات زیر برای PowerShell و از ریشه پروژه هستند.

### فعال‌کردن محیط مجازی

```powershell
.\.venv\Scripts\Activate.ps1
```

### نصب یا به‌روزرسانی وابستگی‌ها

```powershell
python -m pip install --upgrade pip
python -m pip install --upgrade -e ".[dev]"
```

### اجرای برنامه

پس از تکمیل فاز صفر و ایجاد `app/main.py`:

```powershell
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```

### اجرای migration

پس از تکمیل فاز صفر و ایجاد تنظیمات Alembic:

```powershell
python -m alembic upgrade head
```

### اجرای تست‌ها

```powershell
python -m pytest
```

### اجرای lint

```powershell
python -m ruff check .
```

### اجرای type check

پس از ایجاد package برنامه در فاز صفر:

```powershell
python -m mypy app tests
```

در آماده‌سازی اولیه عمداً هیچ اسکلت محصول، migration، RSS، scraping، دیتابیس یا `GeminiProvider` ایجاد نشده است. شروع آن‌ها متعلق به فازهای صریح roadmap است.
