# راهنمای تست در امیر

این راهنما نسخه‌ای به‌روز از شیوه‌ی تست‌نویسی در پروژه امیر است. تمام مثال‌ها بر اساس کد فعلی نوشته شده‌اند و از سرویس‌های واقعی موجود در مخزن استفاده می‌کنند. اگر بخشی از قابلیت‌ها هنوز پیاده‌سازی نشده باشد (مثلاً helperهای محاسبه مانده)، در این راهنما به‌عنوان «کارهای آینده» مشخص شده‌اند تا با کد موجود اشتباه گرفته نشوند.

## مفاهیم پایه قبل از نوشتن تست

### سرویس سند (`DocumentService`)
`DocumentService` منبع اصلی منطق حسابداری در لایهٔ سرویس است. متدهای در دسترس آن عبارت‌اند از:

- `createDocument(User $user, array $data, array $transactions)` – ایجاد سند جدید و ثبت تراکنش‌ها در یک تراکنش دیتابیس. این متد شماره‌ی سند را در صورت نبودن مقدار ورودی به‌طور خودکار تولید می‌کند و خطاهای ولیدیشن را از طریق `DocumentServiceException` برمی‌گرداند.【F:app/Services/DocumentService.php†L15-L57】
- `updateDocument(Document $document, array $data)` – به‌روزرسانی فیلدهای ساده‌ی سند (شماره، تاریخ، عنوان). برای خطاهای ولیدیشن از استثناء عمومی (`\Exception`) استفاده می‌شود.【F:app/Services/DocumentService.php†L59-L92】
- `approveDocument(Document $document)` – بررسی می‌کند مجموع فیلد `value` در تمام تراکنش‌های سند صفر باشد و سپس `approved_at` را مقداردهی می‌کند. در صورت عدم موازنه، استثناء پرتاب می‌شود.【F:app/Services/DocumentService.php†L94-L117】
- `createTransaction(Document $document, array $data)` و `updateTransaction(Transaction $transaction, array $data)` – ولیدیشن تراکنش‌ها را با فیلدهای `subject_id`, `desc`, `value` انجام می‌دهند و مقدار را در جدول `transactions` ذخیره می‌کنند.【F:app/Services/DocumentService.php†L119-L161】
- `updateDocumentTransactions(int $documentId, array $transactionsData)` – داده‌های دریافتی از فرم را که شامل ستون‌های `debit` و `credit` است به مقدار واحد `value` تبدیل می‌کند و رکوردها را به‌روزرسانی می‌کند.【F:app/Services/DocumentService.php†L163-L187】
- `deleteDocument(int $documentId)` – حذف سند و تراکنش‌های مرتبط در یک تراکنش دیتابیس.【F:app/Services/DocumentService.php†L189-L198】

### ساختار تراکنش‌ها و تبدیل «بدهکار/بستانکار»
- در جدول `transactions` تنها یک ستون `value` نگه‌داری می‌شود؛ مقدار مثبت به معنای «بستانکار» و مقدار منفی به معنای «بدهکار» است. مدل `Transaction` همین منطق را برای نمایش فیلدهای مجازی `debit` و `credit` استفاده می‌کند.【F:app/Models/Transaction.php†L11-L44】
- درخواست‌های ایجاد/ویرایش سند توسط `StoreTransactionRequest` اعتبارسنجی می‌شوند. این کلاس پیش از ولیدیشن، مقادیر ورودی `debit` و `credit` را به اعداد اعشاری تبدیل می‌کند و سپس قواعدی مانند «حداقل یکی از بدهکار یا بستانکار باید پر باشد» را اعمال می‌کند.【F:app/Http/Requests/StoreTransactionRequest.php†L23-L71】
- هنگام ساخت داده‌ی تست برای سرویس‌ها، اگر قصد دارید به‌صورت مستقیم `DocumentService::createTransaction` را صدا بزنید باید خودتان مقدار `value` را محاسبه و ارسال کنید. در تست‌های لایهٔ HTTP می‌توانید آرایه‌ی `transactions` را همانند فرم با کلیدهای `debit`, `credit`, `desc`, `subject_id` بسازید تا request آن را آماده کند.

> **نکته:** وجود scope سراسری `FiscalYearScope` روی مدل‌های `Document` و `Subject` باعث می‌شود تمام کوئری‌ها به شناسهٔ شرکت فعال (`session('active-company-id')`) فیلتر شوند. پیش از اجرای تست‌هایی که این مدل‌ها را لمس می‌کنند، حتماً مقدار session را تنظیم کنید.【F:app/Models/Scopes/FiscalYearScope.php†L8-L16】

## Unit Test‌ها

نمونه‌های زیر در پوشه‌ی `tests/Unit/Services` قابل‌استفاده هستند و همگی از `RefreshDatabase` برای بازیابی دیتابیس استفاده می‌کنند.

### ایجاد سند و ذخیرهٔ تراکنش‌ها
```php
<?php

namespace Tests\Unit\Services;

use App\Exceptions\DocumentServiceException;
use App\Models\Document;
use App\Models\Company;
use App\Models\Subject;
use App\Models\Transaction;
use App\Models\User;
use App\Services\DocumentService;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class DocumentServiceTest extends TestCase
{
    use RefreshDatabase;

    public function test_create_document_persists_transactions(): void
    {
        $company = Company::factory()->create();
        session(['active-company-id' => $company->id]);

        $user = User::factory()->create();
        $cash = Subject::factory()->create(['company_id' => $company->id]);
        $revenue = Subject::factory()->create(['company_id' => $company->id]);

        $document = DocumentService::createDocument($user, [
            'title' => 'سند فروش',
            'date' => now()->toDateString(),
        ], [
            [
                'subject_id' => $cash->id,
                'value' => '-150000.00',
                'desc' => 'دریافت وجه نقد',
            ],
            [
                'subject_id' => $revenue->id,
                'value' => '150000.00',
                'desc' => 'ثبت فروش',
            ],
        ]);

        $this->assertEquals($company->id, $document->company_id);
        $this->assertCount(2, $document->transactions);
    }
}
```

### مدیریت خطاهای ولیدیشن
```php
public function test_create_document_requires_date(): void
{
    $this->expectException(DocumentServiceException::class);

    $company = Company::factory()->create();
    session(['active-company-id' => $company->id]);

    $user = User::factory()->create();

    DocumentService::createDocument($user, [
        'title' => 'بدون تاریخ',
        // تاریخ ارسال نشده است
    ], []);
}
```

### تبدیل مقادیر بدهکار/بستانکار در به‌روزرسانی
```php
public function test_update_document_transactions_converts_debit_and_credit(): void
{
    $company = Company::factory()->create();
    $user = User::factory()->create();
    session(['active-company-id' => $company->id]);

    $document = Document::factory()->create([
        'company_id' => $company->id,
        'creator_id' => $user->id,
    ]);
    $subject = Subject::factory()->create(['company_id' => $company->id]);

    DocumentService::updateDocumentTransactions($document->id, [
        [
            'transaction_id' => null,
            'subject_id' => $subject->id,
            'desc' => 'بدهکار',
            'debit' => 200000,
            'credit' => 0,
        ],
        [
            'transaction_id' => null,
            'subject_id' => $subject->id,
            'desc' => 'بستانکار',
            'debit' => 0,
            'credit' => 200000,
        ],
    ]);

    $values = $document->fresh()->transactions()->pluck('value');
    $this->assertTrue($values->contains(-200000.0));
    $this->assertTrue($values->contains(200000.0));
}
```

### تأیید سند و کنترل موازنه
```php
public function test_approve_document_requires_zero_balance(): void
{
    $company = Company::factory()->create();
    $user = User::factory()->create();
    session(['active-company-id' => $company->id]);

    $document = Document::factory()->create([
        'company_id' => $company->id,
        'creator_id' => $user->id,
    ]);

    $subject = Subject::factory()->create(['company_id' => $company->id]);
    Transaction::factory()->state([
        'document_id' => $document->id,
        'subject_id' => $subject->id,
        'value' => 50000,
        'desc' => 'ردیف آزمایشی',
    ])->create();

    $this->expectException(\Exception::class);
    DocumentService::approveDocument($document);
}
```

> **کارهای آینده:** تابع کمکی برای سنجش موازنه‌ی سند (مثلاً `DocumentService::isBalanced`) فعلاً در کد وجود ندارد. تا زمان پیاده‌سازی، برای تست موازنه باید مجموع `value` تراکنش‌ها را دستی محاسبه کنید یا از خود متد `approveDocument` استفاده کنید.

## Feature Test‌ها (HTTP Layer)

```php
<?php

namespace Tests\Feature;

use App\Models\Company;
use App\Models\Subject;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class DocumentControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_store_document_via_form(): void
    {
        $company = Company::factory()->create();
        $user = User::factory()->create();
        session(['active-company-id' => $company->id]);

        $cash = Subject::factory()->create(['company_id' => $company->id]);
        $payable = Subject::factory()->create(['company_id' => $company->id]);

        $response = $this->actingAs($user)->post(route('documents.store'), [
            'title' => 'ثبت سند دستی',
            'number' => 1001,
            'date' => now()->format('Y-m-d'),
            'transactions' => [
                [
                    'subject_id' => $cash->id,
                    'desc' => 'دریافت وجه',
                    'debit' => 200000,
                    'credit' => 0,
                ],
                [
                    'subject_id' => $payable->id,
                    'desc' => 'بستانکار مقابل',
                    'debit' => 0,
                    'credit' => 200000,
                ],
            ],
        ]);

        $response->assertRedirect();
        $this->assertDatabaseHas('documents', [
            'title' => 'ثبت سند دستی',
            'number' => 1001,
            'company_id' => $company->id,
        ]);

        $this->assertDatabaseHas('transactions', [
            'desc' => 'دریافت وجه',
            'value' => -200000,
        ]);
        $this->assertDatabaseHas('transactions', [
            'desc' => 'بستانکار مقابل',
            'value' => 200000,
        ]);
    }

    public function test_validation_errors_are_reported_back_to_session(): void
    {
        $company = Company::factory()->create();
        $user = User::factory()->create();
        session(['active-company-id' => $company->id]);

        $response = $this->actingAs($user)->post(route('documents.store'), [
            'title' => 'سند ناقص',
            'transactions' => [
                [
                    'subject_id' => null,
                    'desc' => '',
                    'debit' => null,
                    'credit' => null,
                ],
            ],
        ]);

        $response->assertSessionHasErrors([
            'number',
            'date',
            'transactions.0.subject_id',
            'transactions.0.debit',
            'transactions.0.credit',
            'transactions.0.desc',
        ]);
    }
}
```

در مثال بالا داده‌ی ورودی دقیقاً با ساختار فرم واقعی (ولیدیشن `StoreTransactionRequest`) تنظیم شده است؛ بنابراین نیازی به محاسبه‌ی دستی مقدار `value` نیست و سرویس خودش آن را محاسبه و ذخیره می‌کند.【F:app/Http/Controllers/DocumentController.php†L41-L90】

## استفاده از Factoryها در تست

### `DocumentFactory`
- این فکتوری شماره‌ی سند را با `DB::table('documents')->max('number') + 1` محاسبه می‌کند و از بین شرکت‌ها و کاربران موجود یکی را انتخاب می‌کند.【F:database/factories/DocumentFactory.php†L18-L27】
- قبل از فراخوانی `Document::factory()` مطمئن شوید حداقل یک رکورد در جداول `companies` و `users` وجود داشته باشد؛ در غیر این صورت فراخوانی `random()` خطا می‌دهد.
- برای کنترل دقیق‌تر می‌توانید state اختصاصی بدهید:

```php
$company = Company::factory()->create();
$user = User::factory()->create();

session(['active-company-id' => $company->id]);

$document = Document::factory()->create([
    'company_id' => $company->id,
    'creator_id' => $user->id,
    'title' => 'سند آزمایشی',
]);
```

### `TransactionFactory`
- پیاده‌سازی فعلی مقدار پیش‌فرضی برنمی‌گرداند و یک آرایه‌ی خالی تحویل می‌دهد.【F:database/factories/TransactionFactory.php†L15-L19】 بنابراین هنگام استفاده باید تمام ستون‌های لازم را از طریق `state` یا آرایه‌ی ورودی مشخص کنید.
- یک الگوی متداول برای ایجاد تراکنش متوازن به شکل زیر است:

```php
Transaction::factory()->state([
    'document_id' => $document->id,
    'subject_id' => $cash->id,
    'value' => -50000,
    'desc' => 'بدهکار',
])->create();

Transaction::factory()->state([
    'document_id' => $document->id,
    'subject_id' => $revenue->id,
    'value' => 50000,
    'desc' => 'بستانکار',
])->create();
```

### سایر فکتوری‌ها
`SubjectFactory` و `CompanyFactory` برای ساخت دادهٔ پایه در تست‌ها کاربردی هستند. توجه کنید که `SubjectFactory` مقدار `company_id` را تعیین نمی‌کند؛ اگر می‌خواهید سرفصل به شرکت خاصی تعلق داشته باشد، مقدار را خودتان تعیین کنید یا قبل از ساخت، session شرکت فعال را ست کنید.【F:database/factories/SubjectFactory.php†L17-L22】

## نکات تکمیلی
- هنگام تست متد `deleteDocument` یا `updateDocument` بهتر است از `Document::factory()` استفاده کنید تا شماره سند معتبر تولید شود.
- از آنجا که متدهای سرویس برای ولیدیشن به `Validator` تکیه دارند، تست‌هایی که انتظار خطا دارند باید پیام عمومی استثناء را بررسی کنند (برای `createDocument` از `DocumentServiceException` و برای سایر متدها از `\Exception`).

## برنامه‌های آینده
- **Helper موازنه سند:** تابعی برای محاسبه یا گزارش مانده‌ی سند هنوز نوشته نشده است و در تست‌ها باید مجموع `value` را دستی بسنجید.
- **مقایسهٔ خودکار جمع بدهکار/بستانکار در فکتوری‌ها:** در حال حاضر فکتوری‌ها جمع مانده را کنترل نمی‌کنند؛ در آینده می‌توان stateهای کمکی برای تولید اسناد متوازن اضافه کرد.

## اجرای تست‌ها

```bash
php artisan test                # اجرای تمام تست‌ها
php artisan test --testsuite=Unit
php artisan test --testsuite=Feature
php artisan test --filter=DocumentServiceTest
```

در صورت نیاز به دیتابیس SQLite در حافظه، مقادیر زیر را در فایل env تست قرار دهید:

```ini
APP_ENV=testing
DB_CONNECTION=sqlite
DB_DATABASE=:memory:
CACHE_DRIVER=array
SESSION_DRIVER=array
QUEUE_CONNECTION=sync
```

همیشه پیش از ارسال Pull Request تمام تست‌ها را اجرا کنید و اگر تستی به دلیل کمبود قابلیت فعلی شکست خورد، موضوع را به‌عنوان کار آینده مستند کنید.
