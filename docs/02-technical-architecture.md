# ۲. معماری فنی

## ۲.۱ نمای کلی

```text
Scheduler / Manual Trigger
          |
          v
     Pipeline Runner
          |
          +--> Source Registry
          |       |
          |       v
          |    RSS Fetcher
          |
          +--> Article Extractor
          |
          +--> Relevance Filter
          |
          +--> Duplicate Grouper
          |
          +--> GeminiProvider
          |
          +--> Ranker & Digest Builder
          |
          v
       SQLite
          |
          v
    FastAPI + Jinja UI
```

## ۲.۲ اصول معماری

1. منطق domain از FastAPI، SQLAlchemy و Gemini جدا بماند.
2. pipeline مجموعه‌ای از stageهای مشخص و قابل اجرای مجدد باشد.
3. ورودی و خروجی هر stage typed و قابل تست باشد.
4. دسترسی شبکه پشت interface قرار گیرد تا تست‌ها offline باشند.
5. provider هوش مصنوعی interface داشته باشد، اما در MVP فقط `GeminiProvider` پیاده‌سازی شود.
6. تغییر مدل Gemini فقط از configuration انجام شود.
7. هیچ تصمیم مهمی فقط به prompt واگذار نشود؛ قواعد قطعی در کد حفظ شوند.

## ۲.۳ ساختار پیشنهادی پوشه‌ها

```text
app/
  main.py
  api/
    routes/
      home.py
      archive.py
      events.py
      sources.py
      runs.py
  core/
    config.py
    logging.py
    time.py
  db/
    base.py
    session.py
    models/
      source.py
      article.py
      news_event.py
      digest.py
      pipeline_run.py
    repositories/
  domain/
    enums.py
    schemas.py
    scoring.py
    url_normalization.py
    deduplication.py
  services/
    feeds/
      base.py
      rss.py
      registry.py
    extraction/
      base.py
      trafilatura_extractor.py
    llm/
      base.py
      gemini.py
      prompts.py
      schemas.py
    pipeline/
      runner.py
      discover.py
      extract.py
      classify.py
      group.py
      summarize.py
      digest.py
  templates/
  static/
alembic/
tests/
  unit/
  integration/
  fixtures/
scripts/
docs/
```

ساختار می‌تواند در پیاده‌سازی کمی تغییر کند، ولی مرزهای مسئولیت باید حفظ شوند.

## ۲.۴ تنظیمات

تنظیمات با `pydantic-settings` از environment خوانده شوند.

```env
APP_ENV=development
APP_HOST=127.0.0.1
APP_PORT=8000
DATABASE_URL=sqlite:///./data/ai_news.db
APP_TIMEZONE=Asia/Tehran
DAILY_DIGEST_LIMIT=10
ARTICLE_LOOKBACK_HOURS=24

GEMINI_API_KEY=
GEMINI_MODEL=gemini-3.5-flash-lite
GEMINI_MAX_OUTPUT_TOKENS=1200
GEMINI_REQUEST_TIMEOUT_SECONDS=60
GEMINI_MAX_INPUT_CHARS=18000

HTTP_TIMEOUT_SECONDS=20
HTTP_MAX_RETRIES=3
USER_AGENT=RussellAINews/0.1 (+local personal news reader)
LOG_LEVEL=INFO
```

نکات:

- مقدار مدل نمونه است و باید قبل از اجرا با مدل‌های فعال و سهمیه حساب Google AI Studio کنترل شود.
- `.env` در `.gitignore` باشد.
- `.env.example` بدون secret commit شود.
- startup باید نبودن کلید Gemini را واضح گزارش کند، اما صفحات read-only بتوانند بدون اجرای pipeline بالا بیایند.

## ۲.۵ مدل داده

### `sources`

| ستون | نوع | توضیح |
|---|---|---|
| id | integer PK | شناسه |
| key | varchar unique | کلید پایدار |
| name | varchar | نام نمایشی |
| source_type | enum | `media` یا `official` |
| discovery_type | enum | فعلاً `rss` |
| feed_url | text | آدرس Feed |
| site_url | text | دامنه اصلی |
| enabled | boolean | فعال بودن |
| trust_score | integer | ۰ تا ۱۰۰ |
| last_checked_at | datetime nullable | آخرین بررسی |
| last_success_at | datetime nullable | آخرین موفقیت |
| last_error | text nullable | خطای sanitize‌شده |
| created_at/updated_at | datetime | UTC |

### `articles`

| ستون | نوع | توضیح |
|---|---|---|
| id | integer PK | شناسه |
| source_id | FK | منبع |
| external_id | varchar nullable | GUID از Feed |
| original_url | text | URL اولیه |
| canonical_url | text unique | URL استانداردشده |
| original_title | text | عنوان اصلی |
| feed_summary | text nullable | خلاصه Feed |
| author | varchar nullable | نویسنده |
| published_at | datetime nullable | UTC |
| discovered_at | datetime | UTC |
| raw_content_hash | varchar nullable | hash متن |
| extracted_text | text nullable | متن پاک‌شده |
| extraction_status | enum | وضعیت استخراج |
| processing_status | enum | وضعیت pipeline |
| is_ai_related | boolean nullable | نتیجه طبقه‌بندی |
| relevance_score | integer nullable | ۰ تا ۱۰۰ |
| category | enum nullable | دسته |
| llm_payload_version | varchar nullable | نسخه prompt/schema |
| last_error | text nullable | خطا |

Index روی `published_at`، `source_id`، `processing_status` و `raw_content_hash` لازم است.

### `news_events`

| ستون | نوع | توضیح |
|---|---|---|
| id | integer PK | شناسه رویداد |
| primary_article_id | FK | مقاله اصلی |
| title_fa | text | عنوان فارسی |
| summary_fa | text | خلاصه فارسی |
| why_it_matters_fa | text | دلیل اهمیت |
| category | enum | دسته |
| relevance_score | integer | ۰ تا ۱۰۰ |
| impact_score | integer | امتیاز Gemini |
| final_score | integer | امتیاز قطعی رتبه‌بندی |
| event_fingerprint | varchar | fingerprint رویداد |
| first_published_at | datetime nullable | UTC |
| created_at/updated_at | datetime | UTC |

### `article_event_links`

- `article_id`
- `event_id`
- `similarity_score`
- unique constraint روی `(article_id, event_id)`

### `daily_digests`

- `id`
- `digest_date` بر اساس Asia/Tehran، unique
- `generated_at` به UTC
- `limit_used`
- `status`

### `daily_digest_items`

- `digest_id`
- `event_id`
- `position`
- `score_snapshot`
- unique روی `(digest_id, event_id)` و `(digest_id, position)`

### `pipeline_runs`

- `id`
- `trigger_type`: manual یا scheduled
- `status`: running، succeeded، partial، failed
- `started_at` و `finished_at`
- شمارنده‌های مرحله‌ای
- `error_summary` بدون secret و متن کامل مقاله

## ۲.۶ وضعیت‌های pipeline

```text
discovered
  -> extracted
  -> classified
  -> rejected_not_ai | ready_for_grouping
  -> grouped
  -> summarized
  -> ranked
  -> included_in_digest
```

وضعیت‌های خطا باید مرحله را مشخص کنند:

- `extraction_failed`
- `classification_failed`
- `summarization_failed`

Retry باید رکورد موجود را ادامه دهد، نه اینکه مقاله جدید بسازد.

## ۲.۷ دریافت RSS

الگوریتم هر منبع:

1. منبع فعال را از registry دریافت کن.
2. Feed را با timeout و User-Agent مشخص دانلود کن.
3. HTTP status و parse errors را کنترل کن.
4. هر item را به DTO داخلی تبدیل کن.
5. URL را canonicalize کن.
6. بر اساس canonical URL و سپس GUID deduplicate کن.
7. فقط رکورد جدید بساز.
8. وضعیت و زمان source را به‌روز کن.

قواعد canonical URL:

- scheme و host lowercase
- حذف fragment
- حذف trailing slash غیرضروری
- حذف پارامترهای tracking مثل `utm_*`, `fbclid`, `gclid`
- مرتب‌سازی query parameterهای باقی‌مانده
- عدم دنبال‌کردن redirect در تابع pure؛ redirect نهایی در مرحله HTTP ثبت شود

## ۲.۸ استخراج مقاله

ورودی: URL و HTML.

خروجی typed:

- title
- text
- author
- published_at
- canonical_url_from_page
- extraction_method
- warnings

قواعد:

- `trafilatura` ابزار اصلی باشد.
- BeautifulSoup فقط برای metadata و fallback محدود استفاده شود.
- حداقل طول متن قابل قبول configuration داشته باشد؛ مقدار اولیه ۵۰۰ کاراکتر.
- HTML خام به‌طور پیش‌فرض ذخیره نشود.
- paywall یا صفحه login دور زده نشود.
- محتوای JavaScript-heavy در MVP شکست کنترل‌شده محسوب شود؛ مرورگر headless اضافه نشود.

## ۲.۹ قرارداد LLM

### interface

```python
class LLMProvider(Protocol):
    async def analyze_article(self, request: ArticleAnalysisRequest) -> ArticleAnalysisResult:
        ...
```

در MVP فقط کلاس زیر وجود دارد:

```python
class GeminiProvider(LLMProvider):
    ...
```

### SDK

- package رسمی: `google-genai`
- از package قدیمی `google-generativeai` استفاده نشود.
- کلاینت یک‌بار ساخته و reuse شود.
- نام مدل از `GEMINI_MODEL` خوانده شود.
- خروجی با JSON schema/Pydantic محدود شود.

مستندات مرجع رسمی:

- https://ai.google.dev/gemini-api/docs/get-started
- https://ai.google.dev/gemini-api/docs/structured-output
- https://ai.google.dev/gemini-api/docs/rate-limits
- https://ai.google.dev/gemini-api/docs/troubleshooting

### schema ورودی

```python
class ArticleAnalysisRequest(BaseModel):
    source_name: str
    original_title: str
    published_at: datetime | None
    article_text: str
```

URL برای خلاصه‌سازی لازم نیست و بهتر است به مدل داده نشود. مدل نباید وب را جست‌وجو کند؛ متن استخراج‌شده تنها منبع حقیقت است.

### schema خروجی

```python
class ArticleAnalysisResult(BaseModel):
    is_ai_related: bool
    relevance_score: int  # 0..100
    category: NewsCategory
    title_fa: str
    summary_fa: str
    why_it_matters_fa: str
    impact_score: int  # 0..100
    entities: list[str]
    event_key_terms: list[str]
```

اعتبارسنجی تکمیلی در کد:

- scoreها بین ۰ و ۱۰۰ باشند.
- عنوان فارسی خالی نباشد اگر خبر مرتبط است.
- summary برای خبر مرتبط حداقل و حداکثر طول داشته باشد.
- برای خبر نامرتبط، متن‌های تولیدی می‌توانند خالی باشند.
- category فقط از enum پذیرفته شود.

### وظیفه prompt

System instruction باید به مدل بگوید:

1. فقط بر اساس متن ورودی پاسخ بده.
2. اگر اطلاعات کافی نیست، عدم قطعیت را بیان کن.
3. نام مدل، شرکت، عدد، تاریخ و ادعا را جعل نکن.
4. خلاصه فارسی روان و خبری باشد.
5. خلاصه حدود ۸۰ تا ۱۵۰ کلمه باشد.
6. «چرا مهم است» یک یا دو جمله و جدا از خلاصه باشد.
7. صرف ذکر AI به معنای مرتبط بودن نیست.
8. خروجی دقیقاً مطابق schema باشد.

Promptها در فایل جدا و versioned نگهداری شوند. مثال: `ARTICLE_ANALYSIS_PROMPT_VERSION = "v1"`.

### خطا و retry

- خطاهای ۴۲۹، timeout و 5xx موقتی‌اند.
- SDK رسمی retry خودکار دارد؛ wrapper نباید retry تو در تو و انفجاری ایجاد کند.
- برای 400 و 403 retry انجام نشود.
- پس از پایان retry، وضعیت مقاله شکست‌خورده ثبت شود.
- اجرای بعدی بتواند آن را دوباره پردازش کند.
- متن مقاله یا API key در log نوشته نشود.

### کنترل سهمیه

- قبل از LLM، dedup دقیق انجام شود.
- طول ورودی با `GEMINI_MAX_INPUT_CHARS` محدود شود.
- برش متن باید شروع مقاله را حفظ کند و در صورت امکان پاراگراف‌بندی را نشکند.
- یک مقاله موفق با همان `content_hash + prompt_version + model` دوباره تحلیل نشود.
- شمار درخواست موفق/ناموفق و مدل ثبت شود.
- مقدار سهمیه واقعی از Google AI Studio کنترل می‌شود و در کد hard-code نمی‌شود.

## ۲.۱۰ تشخیص ارتباط

MVP دو مرحله دارد:

1. prefilter قطعی برای حذف موارد کاملاً واضح یا اولویت‌بندی درخواست‌ها.
2. تصمیم نهایی `GeminiProvider` برای موارد پردازش‌شده.

Prefilter نباید خبر را فقط به دلیل نبود keyword رد کند. بهتر است یکی از خروجی‌های `likely_related`, `uncertain`, `likely_unrelated` تولید کند. در MVP هر سه می‌توانند به Gemini ارسال شوند تا داده ارزیابی جمع شود؛ بعداً `likely_unrelated` با confidence بالا قابل حذف است.

## ۲.۱۱ گروه‌بندی تکراری‌ها

ترتیب بررسی:

1. canonical URL برابر: مقاله دقیقاً تکراری.
2. content hash برابر: مقاله دقیقاً تکراری یا syndication.
3. عنوان normalize‌شده بسیار مشابه: کاندید رویداد یکسان.
4. entityها و `event_key_terms` مشترک به همراه بازه زمانی نزدیک: تقویت شباهت.

برای شباهت عنوان می‌توان از RapidFuzz استفاده کرد. threshold باید تنظیم‌پذیر و با fixtureهای واقعی تست شود؛ مقدار شروع ۸۸ از ۱۰۰ است.

انتخاب مقاله اصلی:

1. منبع رسمی بر رسانه مقدم است.
2. سپس trust score بالاتر.
3. سپس متن کامل‌تر.
4. سپس زمان انتشار زودتر.

## ۲.۱۲ رتبه‌بندی

رتبه‌بندی نهایی deterministic باشد. Gemini فقط `impact_score` را ارائه می‌دهد.

فرمول اولیه:

```text
final_score =
  0.25 * relevance_score
  + 0.25 * impact_score
  + 0.15 * freshness_score
  + 0.15 * source_trust_score
  + 0.10 * coverage_score
  + 0.10 * preference_score
```

تمام componentها ۰ تا ۱۰۰ هستند. قواعد componentها باید در کد و تست‌ها مستند شوند.

## ۲.۱۳ زمان‌بندی

در MVP:

- pipeline با command دستی قابل اجرا باشد.
- scheduler داخل process وب ساخته نشود.
- برای لوکال از Windows Task Scheduler یا یک command روزانه استفاده شود.
- برای سرور آینده cron/container scheduler در نظر گرفته شود.
- lock دیتابیسی یا فایل lock مانع اجرای هم‌زمان شود.

## ۲.۱۴ امنیت و حریم خصوصی

- `GEMINI_API_KEY` فقط از environment.
- secret هرگز در URL، exception response یا template قرار نگیرد.
- URL خارجی قبل از fetch از نظر scheme بررسی شود؛ فقط HTTP/HTTPS.
- redirectها محدود شوند.
- درخواست به localhost، private network و metadata endpointها مسدود شود تا SSRF ایجاد نشود.
- HTML به شکل متن نمایش داده شود و render مستقیم نشود.
- لینک‌ها در UI با `rel="noopener noreferrer"` باز شوند.
- استفاده از Gemini Free Tier ممکن است تابع سیاست پردازش داده Google باشد؛ فقط محتوای عمومی خبر به API ارسال شود.

## ۲.۱۵ observability

هر log باید شامل context زیر باشد، بدون اطلاعات حساس:

- `run_id`
- `source_key`
- `article_id`
- `stage`
- `duration_ms`
- `status`

متریک‌های MVP در دیتابیس:

- feed items discovered
- new articles
- extraction successes/failures
- AI-related/rejected
- Gemini successes/failures
- grouped events
- digest items

## ۲.۱۶ استراتژی تست

### Unit

- URL normalization
- scoring
- title similarity
- time conversion
- Pydantic schemas
- prompt input truncation

### Integration با fixture

- parse کردن RSS ذخیره‌شده
- استخراج مقاله از HTML ذخیره‌شده
- repository و SQLite موقت
- pipeline با HTTP fake و Gemini fake

### تست زنده اختیاری

- با marker جدا مثل `live`
- در CI و اجرای عادی غیرفعال
- نیازمند `GEMINI_API_KEY`
- مصرف سهمیه محدود به یک درخواست کوتاه

هیچ تست عادی نباید به اینترنت یا Gemini واقعی وابسته باشد.

