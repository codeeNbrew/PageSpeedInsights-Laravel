# API.md — Google PageSpeed Insights API v5

Detail API yang dipakai **PageSpeed Monitor** untuk menjalankan test dan cara memetakan hasilnya ke database.

> Dokumen terkait: [PROJECT.md](PROJECT.md) · [ERD.md](ERD.md)
> Referensi resmi: https://developers.google.com/speed/docs/insights/v5/get-started

---

## 1. Ringkasan

| Item | Keterangan |
|---|---|
| Penyedia | Google |
| Versi | v5 |
| Mesin | Lighthouse (sama dengan pagespeed.web.dev) |
| Metode | `GET` |
| Format respons | JSON |
| Autentikasi | API key (query param `key`) |
| Biaya | Gratis dalam batas kuota |
| SDK Laravel resmi | Tidak ada; pakai `Http` facade |
| Durasi per request | ±15–40 detik |

API **tidak** menghasilkan PDF/HTML. Laporan PDF/Excel dibuat sendiri oleh aplikasi dari data JSON.

---

## 2. Kuota

| Batas | Nilai default per project |
|---|---|
| Request per hari | 25.000 |
| Request per 100 detik | ±400 |
| Biaya | Gratis, tanpa billing account |

- Tanpa API key, request akan ditolak (HTTP 429) karena kuota bersama Google sudah habis/ditutup.
- 1 website dengan strategi mobile + desktop = **2 request**.
- Batas aktual bisa dicek di Google Cloud Console → *APIs & Services → PageSpeed Insights API → Quotas*.

---

## 3. Membuat API Key

1. Buka https://console.cloud.google.com, lalu buat atau pilih project.
2. **APIs & Services → Library** → cari **PageSpeed Insights API** → **Enable**.
3. **APIs & Services → Credentials → Create credentials → API key**.
4. **Edit key → API restrictions → Restrict key** → pilih hanya **PageSpeed Insights API** → Save.
5. Masukkan ke `.env`:

```env
PSI_API_KEY=AIzaSy...
PSI_LOCALE=id
```

`config/services.php`:

```php
'psi' => [
    'key'      => env('PSI_API_KEY'),
    'locale'   => env('PSI_LOCALE', 'id'),
    'endpoint' => 'https://www.googleapis.com/pagespeedonline/v5/runPagespeed',
],
```

Tes manual:

```bash
curl "https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=https://example.com&key=API_KEY"
```

> ⚠️ Jangan pernah mengirim API key ke browser/frontend. Semua request dilakukan dari queue job di server.

---

## 4. Request

### 4.1 Endpoint

```
GET https://www.googleapis.com/pagespeedonline/v5/runPagespeed
```

### 4.2 Parameter

| Parameter | Wajib | Nilai | Keterangan |
|---|---|---|---|
| `url` | Ya | URL lengkap `https://...` | Halaman yang dites (diambil dari `websites.url`) |
| `key` | Praktis wajib | API key | |
| `strategy` | Tidak | `mobile` / `desktop` | Default `desktop` |
| `category` | Tidak | `performance`, `accessibility`, `best-practices`, `seo` | **Diulang** untuk banyak kategori. Default hanya `performance` |
| `locale` | Tidak | `id`, `en`, dst. | Bahasa `title`/`description` audit |
| `utm_source` | Tidak | string | Penanda analitik |
| `utm_campaign` | Tidak | string | Penanda analitik |

### 4.3 Contoh

```
GET /pagespeedonline/v5/runPagespeed
    ?url=https%3A%2F%2Fexample.com
    &strategy=mobile
    &category=performance
    &category=accessibility
    &category=best-practices
    &category=seo
    &locale=id
    &key=API_KEY
```

> ⚠️ `category` harus dikirim sebagai `category=a&category=b`, **bukan** `category[0]=a`. `http_build_query()` milik PHP menghasilkan format kedua, jadi susun parameter ini secara manual (lihat bagian 8.1).

---

## 5. Struktur Respons

```
{
  "id": "https://example.com/",
  "analysisUTCTimestamp": "2026-10-30T07:40:12.345Z",
  "loadingExperience":       { ... },   // data pengguna nyata untuk URL
  "originLoadingExperience": { ... },   // data pengguna nyata untuk domain
  "lighthouseResult": {
    "requestedUrl": "...",
    "finalDisplayedUrl": "...",
    "lighthouseVersion": "...",
    "fetchTime": "...",
    "configSettings": { "formFactor": "mobile", "locale": "id", ... },
    "runWarnings": [ ... ],
    "runtimeError": { "code": "...", "message": "..." },   // hanya jika gagal
    "categories": {
      "performance":    { "score": 0.87, "auditRefs": [ ... ] },
      "accessibility":  { ... },
      "best-practices": { ... },
      "seo":            { ... }
    },
    "audits": {
      "largest-contentful-paint": { ... },
      "...": { ... }
    },
    "timing": { "total": 12345.6 },
    "fullPageScreenshot": { ... }
  }
}
```

### 5.1 Objek kategori

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | `performance`, `seo`, dst. |
| `title` | string | Nama kategori |
| `score` | float 0–1 / null | Kalikan 100 untuk tampilan |
| `auditRefs` | array | Daftar `{ id, weight, group }` audit yang termasuk kategori ini |

### 5.2 Objek audit

| Field | Tipe | Keterangan |
|---|---|---|
| `id` | string | ID audit |
| `title` | string | Judul (sesuai `locale`) |
| `description` | string | Penjelasan dalam markdown |
| `score` | float 0–1 / null | |
| `scoreDisplayMode` | string | `numeric`, `binary`, `metricSavings`, `informative`, `notApplicable`, `manual`, `error` |
| `displayValue` | string | Nilai siap tampil, mis. "2,1 s" |
| `numericValue` | number | Nilai mentah |
| `numericUnit` | string | `millisecond`, `byte`, `unitless`, `element` |
| `details` | object | Tabel item, estimasi penghematan, screenshot, dll. |

### 5.3 Objek field data (`loadingExperience.metrics.*`)

| Field | Keterangan |
|---|---|
| `percentile` | Nilai persentil ke-75 |
| `category` | `FAST` / `AVERAGE` / `SLOW` |
| `distributions` | Array `{min, max, proportion}`: porsi pengguna baik / perlu perbaikan / buruk |

`loadingExperience.overall_category` berisi status keseluruhan Core Web Vitals. Seluruh bagian ini **bisa tidak ada** jika website kurang trafik.

---

## 6. Pemetaan ke Database

### 6.1 `test_results`

| Path JSON | Kolom | Transformasi |
|---|---|---|
| `lighthouseResult.finalDisplayedUrl` | `final_url` | |
| `lighthouseResult.categories.performance.score` | `score_performance` | × 100, bulatkan |
| `lighthouseResult.categories.accessibility.score` | `score_accessibility` | × 100 |
| `lighthouseResult.categories.best-practices.score` | `score_best_practices` | × 100 |
| `lighthouseResult.categories.seo.score` | `score_seo` | × 100 |
| `lighthouseResult.audits.first-contentful-paint.numericValue` | `lab_fcp_ms` | bulatkan |
| `lighthouseResult.audits.largest-contentful-paint.numericValue` | `lab_lcp_ms` | bulatkan |
| `lighthouseResult.audits.total-blocking-time.numericValue` | `lab_tbt_ms` | bulatkan |
| `lighthouseResult.audits.cumulative-layout-shift.numericValue` | `lab_cls` | apa adanya |
| `lighthouseResult.audits.speed-index.numericValue` | `lab_si_ms` | bulatkan |
| `lighthouseResult.audits.server-response-time.numericValue` | `lab_ttfb_ms` | bulatkan |
| `lighthouseResult.audits.total-byte-weight.numericValue` | `total_bytes` | bulatkan |
| `loadingExperience.metrics.LARGEST_CONTENTFUL_PAINT_MS.percentile` | `field_lcp_ms` | nullable |
| `loadingExperience.metrics.INTERACTION_TO_NEXT_PAINT.percentile` | `field_inp_ms` | nullable |
| `loadingExperience.metrics.CUMULATIVE_LAYOUT_SHIFT_SCORE.percentile` | `field_cls` | **÷ 100** |
| `loadingExperience.metrics.FIRST_CONTENTFUL_PAINT_MS.percentile` | `field_fcp_ms` | nullable |
| `loadingExperience.metrics.EXPERIMENTAL_TIME_TO_FIRST_BYTE.percentile` | `field_ttfb_ms` | nullable |
| `loadingExperience.overall_category` | `field_overall` | nullable |
| `lighthouseResult.lighthouseVersion` | `lighthouse_version` | |
| `lighthouseResult.runtimeError.message` | `error_message` | jika ada, status `failed` |
| `analysisUTCTimestamp` | `tested_at` | ubah ke timezone aplikasi |
| (seluruh respons) | `raw_json_path` | disimpan `.json.gz` di storage |
| `lighthouseResult.audits.final-screenshot.details.data` | `screenshot_path` | base64 → `.jpg` di storage |

### 6.2 `test_audit_items`

Diambil dengan meloop `lighthouseResult.categories.{kategori}.auditRefs`, lalu mencari audit di `lighthouseResult.audits` dengan aturan:

- lewati jika `scoreDisplayMode` = `notApplicable`, `manual`, atau `informative`
- lewati jika `score` null atau `score >= 1` (sudah lulus)

| Path audit | Kolom |
|---|---|
| (kunci kategori) | `category` |
| `id` | `audit_id` |
| `title` | `title` |
| `description` | `description` |
| `score` | `score` |
| `displayValue` | `display_value` |
| `details.overallSavingsMs` | `savings_ms` |
| `details.overallSavingsBytes` | `savings_bytes` |
| `details.items` | `items` |

> ⚠️ ID audit rekomendasi berubah antar versi Lighthouse (versi baru memakai audit jenis *insight* menggantikan sebagian audit lama). **Jangan hardcode ID rekomendasi.** Selalu loop secara dinamis dan simpan `lighthouse_version`.

### 6.3 `websites` (cache skor terakhir)

Setelah test berhasil:

| Kondisi | Kolom diperbarui |
|---|---|
| strategy = mobile | `last_score_mobile` = `score_performance` |
| strategy = desktop | `last_score_desktop` = `score_performance` |
| selalu | `last_tested_at` = `tested_at` |

---

## 7. Ambang Nilai (untuk warna di UI dan PDF)

### Skor kategori

| Skor | Label | Warna |
|---|---|---|
| 90–100 | Baik | hijau |
| 50–89 | Perlu perbaikan | oranye |
| 0–49 | Buruk | merah |

### Core Web Vitals

| Metrik | Baik | Perlu perbaikan | Buruk |
|---|---|---|---|
| LCP | ≤ 2.500 ms | ≤ 4.000 ms | > 4.000 ms |
| INP | ≤ 200 ms | ≤ 500 ms | > 500 ms |
| CLS | ≤ 0,1 | ≤ 0,25 | > 0,25 |
| FCP | ≤ 1.800 ms | ≤ 3.000 ms | > 3.000 ms |
| TTFB | ≤ 800 ms | ≤ 1.800 ms | > 1.800 ms |

---

## 8. Implementasi Laravel

### 8.1 `App\Services\PageSpeedService`

```php
<?php

namespace App\Services;

use Illuminate\Support\Facades\Http;

class PageSpeedService
{
    public const CATEGORIES = ['performance', 'accessibility', 'best-practices', 'seo'];

    public function run(string $url, string $strategy = 'mobile'): array
    {
        $query = http_build_query([
            'url'      => $url,
            'strategy' => $strategy,
            'locale'   => config('services.psi.locale'),
            'key'      => config('services.psi.key'),
        ]);

        foreach (self::CATEGORIES as $category) {
            $query .= '&category=' . urlencode($category);
        }

        return Http::timeout(120)
            ->acceptJson()
            ->get(config('services.psi.endpoint') . '?' . $query)
            ->throw()
            ->json();
    }
}
```

### 8.2 `App\Services\PageSpeedParser`

```php
<?php

namespace App\Services;

use Illuminate\Support\Arr;
use Illuminate\Support\Carbon;

class PageSpeedParser
{
    private const SKIP_MODES = ['notApplicable', 'manual', 'informative'];

    /** Ringkasan untuk kolom test_results */
    public function summary(array $res): array
    {
        $lh     = $res['lighthouseResult'] ?? [];
        $audits = $lh['audits'] ?? [];
        $field  = $res['loadingExperience']['metrics'] ?? [];

        $score = fn (string $k) => isset($lh['categories'][$k]['score'])
            ? (int) round($lh['categories'][$k]['score'] * 100) : null;
        $num   = fn (string $id) => $audits[$id]['numericValue'] ?? null;
        $p75   = fn (string $k) => $field[$k]['percentile'] ?? null;
        $cls   = $p75('CUMULATIVE_LAYOUT_SHIFT_SCORE');

        return [
            'final_url'            => $lh['finalDisplayedUrl'] ?? null,
            'score_performance'    => $score('performance'),
            'score_accessibility'  => $score('accessibility'),
            'score_best_practices' => $score('best-practices'),
            'score_seo'            => $score('seo'),

            'lab_fcp_ms'  => $this->int($num('first-contentful-paint')),
            'lab_lcp_ms'  => $this->int($num('largest-contentful-paint')),
            'lab_tbt_ms'  => $this->int($num('total-blocking-time')),
            'lab_cls'     => $num('cumulative-layout-shift'),
            'lab_si_ms'   => $this->int($num('speed-index')),
            'lab_ttfb_ms' => $this->int($num('server-response-time')),
            'total_bytes' => $this->int($num('total-byte-weight')),

            'field_lcp_ms'  => $p75('LARGEST_CONTENTFUL_PAINT_MS'),
            'field_inp_ms'  => $p75('INTERACTION_TO_NEXT_PAINT'),
            'field_cls'     => $cls !== null ? $cls / 100 : null,
            'field_fcp_ms'  => $p75('FIRST_CONTENTFUL_PAINT_MS'),
            'field_ttfb_ms' => $p75('EXPERIMENTAL_TIME_TO_FIRST_BYTE'),
            'field_overall' => $res['loadingExperience']['overall_category'] ?? null,

            'lighthouse_version' => $lh['lighthouseVersion'] ?? null,
            'error_message'      => $lh['runtimeError']['message'] ?? null,
            'tested_at'          => isset($res['analysisUTCTimestamp'])
                ? Carbon::parse($res['analysisUTCTimestamp'])->timezone(config('app.timezone'))
                : now(),
        ];
    }

    /** Baris untuk test_audit_items */
    public function auditItems(array $res): array
    {
        $lh   = $res['lighthouseResult'] ?? [];
        $rows = [];

        foreach ($lh['categories'] ?? [] as $categoryKey => $category) {
            foreach ($category['auditRefs'] ?? [] as $ref) {
                $a = $lh['audits'][$ref['id']] ?? null;

                if (! $a) continue;
                if (in_array($a['scoreDisplayMode'] ?? '', self::SKIP_MODES, true)) continue;
                if (! isset($a['score']) || $a['score'] >= 1) continue;

                $rows[] = [
                    'category'      => $categoryKey,
                    'audit_id'      => $a['id'],
                    'title'         => $a['title'],
                    'description'   => $a['description'] ?? null,
                    'score'         => $a['score'],
                    'display_value' => $a['displayValue'] ?? null,
                    'savings_ms'    => $this->int(Arr::get($a, 'details.overallSavingsMs')),
                    'savings_bytes' => $this->int(Arr::get($a, 'details.overallSavingsBytes')),
                    'items'         => Arr::get($a, 'details.items'),
                ];
            }
        }

        return $rows;
    }

    /** Screenshot akhir sebagai binary JPEG */
    public function screenshot(array $res): ?string
    {
        $data = Arr::get($res, 'lighthouseResult.audits.final-screenshot.details.data');

        if (! $data || ! str_contains($data, ',')) return null;

        return base64_decode(explode(',', $data, 2)[1]) ?: null;
    }

    private function int(mixed $v): ?int
    {
        return $v === null ? null : (int) round($v);
    }
}
```

### 8.3 `App\Jobs\RunPageSpeedTest`

```php
<?php

namespace App\Jobs;

use App\Models\TestResult;
use App\Services\{PageSpeedParser, PageSpeedService};
use Illuminate\Bus\{Batchable, Queueable};
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Http\Client\RequestException;
use Illuminate\Queue\{InteractsWithQueue, SerializesModels};
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Support\Facades\Storage;

class RunPageSpeedTest implements ShouldQueue
{
    use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 180;
    public array $backoff = [30, 90, 180];

    public function __construct(public TestResult $result) {}

    public function middleware(): array
    {
        return [new RateLimited('pagespeed')];
    }

    public function handle(PageSpeedService $psi, PageSpeedParser $parser): void
    {
        if ($this->batch()?->cancelled()) return;

        $this->result->update(['status' => 'running']);

        try {
            $res = $psi->run($this->result->url, $this->result->strategy);
        } catch (RequestException $e) {
            // 400/403 = URL bermasalah atau key salah → tidak perlu retry
            if (in_array($e->response->status(), [400, 403], true)) {
                $this->markFailed($this->errorMessage($e));
                return;
            }
            throw $e; // 429 / 5xx → retry dengan backoff
        }

        $dir = "pagespeed/{$this->result->test_batch_id}";
        Storage::put("$dir/{$this->result->id}.json.gz", gzencode(json_encode($res), 9));

        $shot = null;
        if ($img = $parser->screenshot($res)) {
            Storage::put($shot = "$dir/{$this->result->id}.jpg", $img);
        }

        $summary = $parser->summary($res);
        $failed  = $summary['error_message'] !== null;

        $this->result->update($summary + [
            'status'          => $failed ? 'failed' : 'done',
            'raw_json_path'   => "$dir/{$this->result->id}.json.gz",
            'screenshot_path' => $shot,
        ]);

        $this->result->auditItems()->delete();
        $this->result->auditItems()->createMany($parser->auditItems($res));

        if (! $failed) {
            $column = $this->result->strategy === 'mobile' ? 'last_score_mobile' : 'last_score_desktop';
            $this->result->website()->update([
                $column          => $summary['score_performance'],
                'last_tested_at' => $summary['tested_at'],
            ]);
        }

        $this->result->batch->increment($failed ? 'failed' : 'completed');
    }

    public function failed(\Throwable $e): void
    {
        $this->markFailed($e->getMessage());
    }

    private function markFailed(string $message): void
    {
        $this->result->update(['status' => 'failed', 'error_message' => $message]);
        $this->result->batch->increment('failed');
    }

    private function errorMessage(RequestException $e): string
    {
        return $e->response->json('error.message') ?? $e->getMessage();
    }
}
```

### 8.4 Rate limiter

`app/Providers/AppServiceProvider.php`:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    RateLimiter::for('pagespeed', fn () => Limit::perMinute(200));
}
```

### 8.5 `App\Services\TestRunner`: menjalankan multi test dari website tersimpan

```php
<?php

namespace App\Services;

use App\Jobs\RunPageSpeedTest;
use App\Models\{TestBatch, User, Website};
use Illuminate\Support\Facades\{Bus, DB};

class TestRunner
{
    public function __construct(private ActivityLogger $logger) {}

    /** @param int[] $websiteIds  @param string[] $strategies */
    public function run(User $user, array $websiteIds, array $strategies, ?string $name = null): TestBatch
    {
        $websites = Website::query()
            ->whereIn('id', $websiteIds)
            ->where('is_active', true)
            ->when(! $user->isAdmin(), fn ($q) => $q->where('user_id', $user->id))
            ->get();

        $batch = DB::transaction(function () use ($user, $websites, $strategies, $name) {
            $batch = TestBatch::create([
                'user_id'    => $user->id,
                'name'       => $name,
                'strategies' => $strategies,
                'status'     => 'queued',
                'total'      => $websites->count() * count($strategies),
            ]);

            foreach ($websites as $website) {
                foreach ($strategies as $strategy) {
                    $batch->results()->create([
                        'website_id' => $website->id,
                        'url'        => $website->url,   // snapshot URL tersimpan
                        'strategy'   => $strategy,
                    ]);
                }
            }

            return $batch;
        });

        $jobs = $batch->results->map(fn ($r) => new RunPageSpeedTest($r))->all();

        $jobBatch = Bus::batch($jobs)
            ->name("pagespeed-batch-{$batch->id}")
            ->allowFailures()
            ->onQueue('pagespeed')
            ->before(fn () => $batch->update(['status' => 'running', 'started_at' => now()]))
            ->finally(function () use ($batch) {
                $batch->refresh();
                $batch->update([
                    'status'      => $batch->completed > 0 ? 'done' : 'failed',
                    'finished_at' => now(),
                ]);
            })
            ->dispatch();

        $batch->update(['job_batch_id' => $jobBatch->id]);

        $this->logger->log('test.started', $batch, [
            'websites'   => $websites->pluck('id'),
            'strategies' => $strategies,
        ]);

        return $batch;
    }
}
```

Pemanggilan dari komponen Livewire:

```php
// Livewire\Websites\Index
public array $selected = [];
public array $strategies = ['mobile'];

public function runTest(TestRunner $runner)
{
    $this->validate([
        'selected'     => 'required|array|min:1',
        'strategies'   => 'required|array|min:1',
        'strategies.*' => 'in:mobile,desktop',
    ]);

    $batch = $runner->run(auth()->user(), $this->selected, $this->strategies);

    return $this->redirectRoute('batches.show', $batch, navigate: true);
}

// Tombol satu klik di setiap baris
public function testOne(int $websiteId, TestRunner $runner)
{
    $batch = $runner->run(auth()->user(), [$websiteId], ['mobile', 'desktop']);

    return $this->redirectRoute('batches.show', $batch, navigate: true);
}
```

---

## 9. Penanganan Error

| HTTP | Penyebab umum | Tindakan di aplikasi |
|---|---|---|
| 200 + `runtimeError` | Lighthouse gagal memuat halaman (timeout, halaman error, DNS) | Status `failed`, simpan pesan |
| 400 | URL tidak valid / halaman tidak bisa diakses (`FAILED_DOCUMENT_REQUEST`, `ERRORED_DOCUMENT_REQUEST`) | Status `failed`, tanpa retry |
| 403 | API key salah, API belum di-enable, key dibatasi | Status `failed`, cek konfigurasi |
| 429 | Batas per menit atau kuota harian terlampaui | Retry dengan backoff; jika kuota harian, hentikan batch |
| 500 / 503 | Gangguan di sisi Google | Retry dengan backoff |
| Timeout koneksi | Test terlalu lama | Retry dengan backoff |

Contoh body error:

```json
{
  "error": {
    "code": 400,
    "message": "Lighthouse returned error: FAILED_DOCUMENT_REQUEST. ...",
    "errors": [{ "domain": "lighthouse", "reason": "lighthouseUserError" }]
  }
}
```

---

## 10. Penyimpanan File

| File | Lokasi | Ukuran kira-kira |
|---|---|---|
| Respons mentah | `storage/app/private/pagespeed/{batch_id}/{result_id}.json.gz` | 50–300 KB (terkompres) |
| Screenshot akhir | `storage/app/private/pagespeed/{batch_id}/{result_id}.jpg` | 20–60 KB |

Membaca kembali JSON mentah:

```php
// TestResult.php
public function rawJson(): ?array
{
    return $this->raw_json_path && Storage::exists($this->raw_json_path)
        ? json_decode(gzdecode(Storage::get($this->raw_json_path)), true)
        : null;
}
```

File berumur lebih dari `PSI_RAW_RETENTION_DAYS` dihapus oleh scheduler; ringkasan di database tetap tersimpan.

---

## 11. Catatan

- Hasil test bervariasi ±5–10 poin antar run karena kondisi jaringan dan server Google.
- Field data (CrUX) hanya ada untuk website dengan trafik Chrome yang cukup.
- PSI mengakses website dari server Google, sehingga website yang perlu login, berada di localhost, atau diblokir firewall tidak bisa dites.
- `locale=id` membuat judul dan deskripsi rekomendasi berbahasa Indonesia, cocok untuk laporan PDF.
- Simpan `lighthouse_version` di setiap hasil, karena perubahan versi Lighthouse bisa menggeser skor.
