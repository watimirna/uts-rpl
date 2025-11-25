# Sistem Stock Opname - Object-Oriented Design

## 📋 Deskripsi Sistem

**Stock Opname** adalah sistem pengelolaan inventaris yang dirancang untuk membantu perusahaan atau organisasi melakukan perhitungan fisik stok barang secara berkala dan mencocokkannya dengan data stok di sistem. Sistem ini mengatasi permasalahan umum seperti:

- **Ketidaksesuaian stok fisik dan sistem** akibat kesalahan pencatatan
- **Proses manual yang memakan waktu** dalam penghitungan dan rekonsiliasi
- **Kurangnya audit trail** dalam perubahan stok
- **Kesulitan approval** dan monitoring stock opname

Sistem ini menggunakan pendekatan **Object-Oriented Design (OOD)** dengan penerapan prinsip SOLID dan design patterns untuk memastikan kode yang maintainable, reusable, dan extendable.

[Google Docs](https://docs.google.com/document/d/1qZQ6MNzSI4l2c_5VrlF8bD-4sVQoslljCEb8uY2pr3Y/edit?tab=t.0#heading=h.xjkf7ykd3zjq)

---

## 1️⃣ Use Case Diagram - Aktor dan Aktivitas Sistem

### Aktor Utama

1. **Admin/Staff**

   - Petugas yang bertanggung jawab melakukan perhitungan stok fisik
   - Melakukan input data stok fisik ke sistem
   - Mereview dan submit data untuk approval

2. **Manager/Supervisor**
   - Atasan yang melakukan verifikasi dan approval
   - Mereview hasil stock opname
   - Menyetujui atau menolak stock opname
   - Generate dan mencetak laporan

### Aktivitas/Use Case

**Untuk Admin/Staff:**

- **Buat Stock Opname**: Membuat dokumen stock opname baru
- **Pilih Barang**: Memilih barang yang akan dihitung
- **Input Stok Fisik**: Memasukkan hasil perhitungan fisik
- **Lihat Selisih Stok**: Melihat perbedaan antara stok sistem dan fisik
- **Tambah Catatan**: Menambahkan keterangan jika diperlukan
- **Submit untuk Approval**: Mengirimkan data untuk disetujui Manager

**Untuk Manager/Supervisor:**

- **Approve Stock Opname**: Menyetujui atau menolak stock opname
- **Update Stok Sistem**: Sistem otomatis update stok setelah approval
- **Generate Laporan**: Membuat laporan hasil stock opname

### Diagram

![Use Case Diagram](./images/UseCase%20Diagram.jpg)

**Relasi Use Case:**

- `<<include>>`: Use case yang wajib dijalankan (contoh: Buat Stock Opname → Pilih Barang)
- `<<extend>>`: Use case opsional (contoh: Submit untuk Approval ← Tambah Catatan)

**Catatan Penting:**

- Selisih dihitung otomatis dengan rumus: `selisih = stok_fisik - stok_sistem`
- Status ditentukan berdasarkan selisih:
  - **Match**: selisih = 0 (stok cocok)
  - **Over**: selisih > 0 (stok lebih)
  - **Short**: selisih < 0 (stok kurang)

---

## 2️⃣ Class Diagram - Struktur dan Relasi Kelas

### Arsitektur: MVC Pattern

Sistem ini menggunakan **Model-View-Controller (MVC)** pattern untuk pemisahan concerns:

#### A. Boundary Layer (View/UI)

```
StockOpnameView
```

Bertanggung jawab untuk antarmuka pengguna:

- Menampilkan form input
- Menampilkan daftar barang
- Menampilkan selisih stok
- Menampilkan laporan

#### B. Control Layer (Controller/Business Logic)

```
StockOpnameController
StockOpnameService
```

Menangani logika bisnis:

- Validasi input
- Perhitungan selisih
- Penentuan status
- Orchestrasi proses bisnis

#### C. Entity Layer (Domain Model)

```
StockOpname (Aggregate Root)
├── ItemOpname (Detail per barang)
└── Barang (Referensi master data)

Enumerations:
- StatusOpname: DRAFT, MENUNGGU_APPROVAL, DISETUJUI, DITOLAK
- StatusSelisih: MATCH, KELEBIHAN, KEKURANGAN
```

### Relasi Antar Kelas

1. **StockOpnameView → StockOpnameController**: View menggunakan controller
2. **StockOpnameController → StockOpnameService**: Controller menggunakan service untuk business logic
3. **StockOpnameController → StockOpname**: Controller mengelola domain model
4. **StockOpname ◆→ ItemOpname**: Composition (1 to many) - Stock Opname memiliki banyak item
5. **ItemOpname → Barang**: Association - Item mereferensi data barang
6. **ItemOpname → StatusSelisih**: Dependency - Item memiliki status selisih
7. **StockOpname → StatusOpname**: Dependency - Stock Opname memiliki status

### Diagram

![Class Diagram](./images/Class%20Diagram.jpg)

### Kelas-Kelas Utama

#### 1. StockOpname (Model)

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class StockOpname extends Model
{
    protected $fillable = [
        'nomor_opname',      // SO/2024/001
        'tanggal_opname',
        'status',            // draft, menunggu_approval, disetujui, ditolak
        'catatan',
        'dibuat_oleh',
        'disetujui_oleh',
    ];

    protected $casts = [
        'tanggal_opname' => 'date',
    ];

    public function items(): HasMany
    {
        return $this->hasMany(ItemOpname::class);
    }

    public function creator(): BelongsTo
    {
        return $this->belongsTo(User::class, 'dibuat_oleh');
    }

    public function approver(): BelongsTo
    {
        return $this->belongsTo(User::class, 'disetujui_oleh');
    }

    public function hitungTotalSelisih(): int
    {
        return $this->items->sum('selisih');
    }

    public function ubahStatus(string $status): void
    {
        $this->update(['status' => $status]);
    }
}
```

#### 2. ItemOpname (Model)

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class ItemOpname extends Model
{
    protected $fillable = [
        'stock_opname_id',
        'barang_id',
        'stok_sistem',
        'stok_fisik',
        'selisih',
        'status',           // match, kelebihan, kekurangan
        'catatan',
    ];

    public function stockOpname(): BelongsTo
    {
        return $this->belongsTo(StockOpname::class);
    }

    public function barang(): BelongsTo
    {
        return $this->belongsTo(Barang::class);
    }

    public function hitungSelisih(): int
    {
        $this->selisih = $this->stok_fisik - $this->stok_sistem;
        return $this->selisih;
    }

    public function tentukanStatus(): string
    {
        if ($this->selisih === 0) {
            $this->status = 'match';
        } elseif ($this->selisih > 0) {
            $this->status = 'kelebihan';
        } else {
            $this->status = 'kekurangan';
        }
        return $this->status;
    }
}
```

#### 3. StockOpnameService (Business Logic)

```php
<?php

namespace App\Services;

use App\Models\StockOpname;
use App\Models\ItemOpname;
use App\Models\Barang;
use Illuminate\Support\Facades\DB;

class StockOpnameService
{
    public function validateInputStok(int $stokFisik): bool
    {
        return $stokFisik >= 0;
    }

    public function hitungSelisih(int $stokSistem, int $stokFisik): int
    {
        return $stokFisik - $stokSistem;
    }

    public function tentukanStatus(int $selisih): string
    {
        if ($selisih === 0) return 'match';
        if ($selisih > 0) return 'kelebihan';
        return 'kekurangan';
    }

    public function updateStokSistem(int $barangId, int $selisih): void
    {
        $barang = Barang::findOrFail($barangId);
        $barang->increment('stok_tersedia', $selisih);
    }

    public function buatTransaksiPenyesuaian(ItemOpname $item): void
    {
        // Catat transaksi untuk audit trail
        DB::table('transaksi_penyesuaian')->insert([
            'barang_id' => $item->barang_id,
            'selisih' => $item->selisih,
            'keterangan' => 'Penyesuaian dari Stock Opname',
            'created_at' => now(),
        ]);
    }
}
```

**Logika Bisnis Utama:**

```
hitungSelisih():
  return stokFisik - stokSistem

tentukanStatus(selisih):
  if selisih == 0: return MATCH
  if selisih > 0: return KELEBIHAN
  if selisih < 0: return KEKURANGAN
```

---

## 3️⃣ Prinsip SOLID dan Design Patterns

### Prinsip SOLID yang Diterapkan

#### 1. **Single Responsibility Principle (SRP)** ⭐ _PALING RELEVAN_

Setiap kelas memiliki satu tanggung jawab:

- `StockOpnameView`: Hanya menangani UI/presentasi
- `StockOpnameController`: Hanya menangani koordinasi alur kerja
- `StockOpnameService`: Hanya menangani business logic
- `StockOpname`: Hanya memodelkan data stock opname
- `ItemOpname`: Hanya memodelkan data item individual

**Contoh Implementasi:**

```php
<?php

// ❌ SALAH - Melanggar SRP
class StockOpnameController extends Controller
{
    public function approveOpname($id)
    {
        // Validasi
        // Hitung selisih
        // Update database
        // Generate PDF
        // Send email
        // Update UI
    }
}

// ✅ BENAR - Menerapkan SRP
class StockOpnameController extends Controller
{
    public function __construct(
        private StockOpnameService $service,
        private ReportGenerator $reportGen,
        private NotificationService $notif
    ) {}

    public function approveOpname($id)
    {
        $this->service->validateAndApprove($id);   // Business logic
        $this->reportGen->generate($id);           // Report generation
        $this->notif->sendApprovalNotif($id);      // Notification
    }
}
```

#### 2. **Open/Closed Principle (OCP)**

Sistem terbuka untuk ekstensi, tertutup untuk modifikasi:

```php
<?php

namespace App\Contracts;

// Interface untuk strategy pattern
interface StockCalculationStrategy
{
    public function calculateDifference(int $systemStock, int $physicalStock): int;
}
```

```php
<?php

namespace App\Services\Calculations;

use App\Contracts\StockCalculationStrategy;

// Implementasi standar
class StandardCalculation implements StockCalculationStrategy
{
    public function calculateDifference(int $systemStock, int $physicalStock): int
    {
        return $physicalStock - $systemStock;
    }
}

// Ekstensi untuk perhitungan dengan toleransi
class ToleranceCalculation implements StockCalculationStrategy
{
    public function __construct(
        private float $tolerancePercent = 0.05
    ) {}

    public function calculateDifference(int $systemStock, int $physicalStock): int
    {
        $diff = $physicalStock - $systemStock;
        $tolerance = $systemStock * $this->tolerancePercent;

        if (abs($diff) <= $tolerance) {
            return 0; // Dalam toleransi
        }

        return $diff;
    }
}
```

#### 3. **Dependency Inversion Principle (DIP)**

Bergantung pada abstraksi, bukan konkret:

```php
<?php

namespace App\Contracts;

use App\Models\StockOpname;
use Illuminate\Support\Collection;

interface StockRepositoryInterface
{
    public function save(StockOpname $opname): StockOpname;
    public function findById(int $id): ?StockOpname;
    public function findByStatus(string $status): Collection;
}
```

```php
<?php

namespace App\Http\Controllers;

use App\Contracts\StockRepositoryInterface;

// Controller bergantung pada interface, bukan implementasi konkret
class StockOpnameController extends Controller
{
    public function __construct(
        private StockRepositoryInterface $repository // Abstraksi
    ) {}

    public function show($id)
    {
        return $this->repository->findById($id);
    }
}
```

```php
<?php

namespace App\Repositories;

use App\Contracts\StockRepositoryInterface;
use App\Models\StockOpname;
use Illuminate\Support\Collection;

// Implementasi bisa diganti tanpa ubah controller
class EloquentStockRepository implements StockRepositoryInterface
{
    public function save(StockOpname $opname): StockOpname
    {
        $opname->save();
        return $opname;
    }

    public function findById(int $id): ?StockOpname
    {
        return StockOpname::find($id);
    }

    public function findByStatus(string $status): Collection
    {
        return StockOpname::where('status', $status)->get();
    }
}

// Binding di AppServiceProvider
// $this->app->bind(StockRepositoryInterface::class, EloquentStockRepository::class);
```

### Design Patterns (Creational)

#### 1. **Factory Method Pattern** 🏭

Membuat objek stock opname dengan logic pembuatan yang terpusat:

```php
<?php

namespace App\Factories;

use App\Models\StockOpname;
use App\Models\User;
use Carbon\Carbon;

class StockOpnameFactory
{
    private static int $counter = 0;

    public static function createNew(User $creator): StockOpname
    {
        self::$counter++;

        return StockOpname::create([
            'nomor_opname' => self::generateNomor(),
            'tanggal_opname' => Carbon::now(),
            'status' => 'draft',
            'dibuat_oleh' => $creator->id,
        ]);
    }

    private static function generateNomor(): string
    {
        $date = Carbon::now()->format('Y/m');
        return sprintf('SO/%s/%03d', $date, self::$counter);
    }
}

// Penggunaan di Controller
$opname = StockOpnameFactory::createNew(auth()->user());
```

**Keuntungan:**

- Centralized logic untuk pembuatan nomor SO
- Konsisten dalam inisialisasi object
- Mudah di-test karena terisolasi

#### 2. **Builder Pattern** 🔨

Membuat objek ItemOpname dengan cara yang lebih readable:

```php
<?php

namespace App\Builders;

use App\Models\ItemOpname;
use App\Models\Barang;

class ItemOpnameBuilder
{
    private array $attributes = [];

    public function withBarang(Barang $barang): self
    {
        $this->attributes['barang_id'] = $barang->id;
        $this->attributes['stok_sistem'] = $barang->stok_tersedia;
        return $this;
    }

    public function withStokFisik(int $stok): self
    {
        $this->attributes['stok_fisik'] = $stok;
        return $this;
    }

    public function withCatatan(?string $catatan): self
    {
        $this->attributes['catatan'] = $catatan;
        return $this;
    }

    public function build(): ItemOpname
    {
        $item = new ItemOpname($this->attributes);
        $item->hitungSelisih();
        $item->tentukanStatus();
        return $item;
    }
}

// Penggunaan dengan Fluent Interface
$item = (new ItemOpnameBuilder())
    ->withBarang($barangA)
    ->withStokFisik(95)
    ->withCatatan('Ada barang rusak 5 unit')
    ->build();
```

**Keuntungan:**

- Kode lebih readable dan self-documenting
- Validasi terpusat di method build()
- Fleksibel untuk optional parameters

#### 3. **Singleton Pattern** 👤

Memastikan hanya ada satu instance dari service yang mengelola stok (dalam Laravel menggunakan Service Container):

```php
<?php

namespace App\Services;

use App\Models\Barang;
use Illuminate\Support\Facades\Cache;

class StockService
{
    private static ?StockService $instance = null;

    private function __construct()
    {
        // Private constructor untuk singleton
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    public function getStokSistem(int $barangId): int
    {
        return Cache::remember("stock_{$barangId}", 3600, function () use ($barangId) {
            $barang = Barang::find($barangId);
            return $barang?->stok_tersedia ?? 0;
        });
    }

    public function updateStok(int $barangId, int $selisih): void
    {
        $barang = Barang::findOrFail($barangId);
        $barang->increment('stok_tersedia', $selisih);

        // Clear cache
        Cache::forget("stock_{$barangId}");
    }
}

// Penggunaan
$service = StockService::getInstance();
$stok = $service->getStokSistem(1);
```

**Alternatif Laravel (Recommended):**

```php
<?php

// Di AppServiceProvider.php
public function register()
{
    $this->app->singleton(StockService::class, function ($app) {
        return new StockService();
    });
}

// Penggunaan via Dependency Injection
class StockOpnameController extends Controller
{
    public function __construct(
        private StockService $stockService
    ) {}

    public function getStok($barangId)
    {
        return $this->stockService->getStokSistem($barangId);
    }
}
```

**Keuntungan:**

- Satu source of truth untuk data stok
- Memory efficient (hanya satu instance)
- Thread-safe dengan synchronized

---

## 4️⃣ Activity & Sequence Diagram - Alur Proses Stock Opname

### Activity Diagram

Activity diagram menunjukkan **alur lengkap proses stock opname** dari awal hingga akhir, mencakup semua decision points dan parallel activities.

![Activity Diagram](./images/Activity%20Diagram.jpg)

### Tahapan Proses (Activity Flow)

#### **Phase 1: Inisialisasi (Staff)**

1. Staff login ke sistem
2. Buka menu Stock Opname
3. Klik "Buat Stock Opname Baru"
4. Sistem generate nomor SO (contoh: SO/2024/001)
5. Status di-set sebagai **DRAFT**

#### **Phase 2: Input Data (Staff)**

6. Staff melihat daftar barang
7. Pilih barang yang akan dihitung
8. Sistem ambil stok sistem dari database (contoh: 100 unit)
9. Staff pergi ke gudang dan hitung stok fisik
10. Input jumlah stok fisik (contoh: 95 unit)

#### **Phase 3: Perhitungan Otomatis (System)**

11. Sistem hitung selisih: `selisih = stok_fisik - stok_sistem`
    - Contoh: 95 - 100 = -5
12. Sistem tentukan status:
    - **Jika selisih = 0**: Status = MATCH (Cocok) ✅
    - **Jika selisih > 0**: Status = KELEBIHAN (Ada stok lebih) 📈
    - **Jika selisih < 0**: Status = KEKURANGAN (Ada stok kurang) 📉
13. Simpan data item

#### **Phase 4: Review & Koreksi (Staff)**

14. Loop untuk setiap barang yang perlu dihitung
15. Staff review semua data yang telah diinput
16. Jika ada kesalahan, edit atau hapus item
17. Jika perlu, tambahkan catatan/keterangan
    - Contoh: "Ditemukan 5 unit barang rusak"

#### **Phase 5: Submit untuk Approval (Staff)**

18. Staff klik "Submit untuk Approval"
19. Sistem ubah status menjadi **MENUNGGU_APPROVAL**
20. Sistem kirim notifikasi ke Manager

#### **Phase 6: Review & Decision (Manager)**

21. Manager terima notifikasi
22. Manager buka detail stock opname
23. Manager review data dan selisih
24. **Decision Point**:
    - **Jika DITOLAK**:
      - Sistem ubah status = DITOLAK
      - Catat alasan penolakan
      - Staff terima notifikasi penolakan
      - Staff perbaiki dan submit ulang
    - **Jika DISETUJUI**: Lanjut ke phase 7

#### **Phase 7: Approval & Update Stok (Manager + System)**

25. Manager klik "Approve"
26. Sistem validasi data
27. **Loop untuk setiap item dengan selisih**:
    - Update stok barang: `stok_baru = stok_lama + selisih`
    - Contoh Barang A: `stok_baru = 100 + (-5) = 95`
    - Catat transaksi penyesuaian untuk audit trail
28. Sistem ubah status menjadi **DISETUJUI**
29. Simpan informasi approver dan waktu approval
30. Commit semua perubahan ke database

#### **Phase 8: Generate Laporan (Optional)**

31. Jika perlu laporan:
    - Sistem generate laporan PDF
    - Tampilkan ringkasan:
      - Total item dihitung
      - Total item dengan selisih
      - Total kelebihan
      - Total kekurangan
32. Manager download/cetak laporan

### Sequence Diagram

Sequence diagram menunjukkan **interaksi antar komponen** secara detail dengan timeline:

![Sequence Diagram](./images/Sequence%20Diagram.jpg)

### Penjelasan Interaksi Utama

#### 1. Membuat Stock Opname Baru

```
User → View → Controller → Service → Controller → View → User
```

- User trigger action
- Controller generate nomor otomatis
- Status awal = DRAFT
- Return opnameId ke user

#### 2. Pilih Barang & Input Stok

```
User → View → Controller → StockBarang → Controller → View → User
```

- Controller fetch stok sistem dari database
- User input stok fisik
- Controller → Service untuk hitung selisih
- Service menentukan status (MATCH/KELEBIHAN/KEKURANGAN)
- Controller simpan ItemOpname

#### 3. Submit untuk Approval

```
User → View → Controller → Controller → View → User
```

- Controller ubah status menjadi MENUNGGU_APPROVAL
- Trigger notifikasi ke Manager

#### 4. Approval Process

```
Manager → View → Controller → Service → Loop(Stock Update) → Controller → View → Manager
```

- Manager review dan approve
- Service validasi data
- **Loop untuk setiap item**: Update stok dan catat transaksi
- Status berubah menjadi DISETUJUI
- Return success message

#### 5. Generate Laporan

```
Manager → View → Controller → Service → Controller → View → Manager
```

- Service generate PDF report
- Return file untuk download/print

---

## 5️⃣ State Diagram & Evaluasi Desain

### State Diagram - Siklus Hidup Stock Opname

State diagram menunjukkan **perubahan status dan transisi** data Stock Opname:

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   [Start] ──► DRAFT ──► MENUNGGU_APPROVAL ──► DISETUJUI ──► [End]   │
│                │              │                                     │
│                │              ▼                                     │
│                └─────────  DITOLAK ──────────────────────► [End]    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Status dan Transisinya

#### 1. **DRAFT** (Status Awal)

```
Karakteristik:
- Stock opname baru dibuat
- Data masih dapat diubah
- User dapat menambah/edit/hapus item
- Belum ada approval
```

**Aksi yang dapat dilakukan:**

- ✅ Tambah item barang
- ✅ Edit stok fisik
- ✅ Hapus item
- ✅ Tambah catatan
- ✅ Submit untuk approval

**Transisi keluar:**

- → **MENUNGGU_APPROVAL**: Saat staff klik "Submit" (syarat: semua item valid)
- → **DRAFT** (loop): Saat staff tambah/edit item

#### 2. **MENUNGGU_APPROVAL** (Pending State)

```
Karakteristik:
- Data sudah lengkap dan final
- Menunggu review dari Manager
- Data tidak dapat diubah (read-only)
- Notifikasi sudah dikirim ke Manager
```

**Aksi yang dapat dilakukan:**

- ⏳ Menunggu keputusan Manager
- 👀 Manager review data
- ✅ Manager dapat approve
- ❌ Manager dapat reject

**Transisi keluar:**

- → **DISETUJUI**: Manager approve (syarat: data valid)
- → **DITOLAK**: Manager reject (syarat: ada masalah/ketidaksesuaian)
- → **DRAFT**: Dikembalikan untuk revisi (jika perlu perbaikan minor)

#### 3. **DISETUJUI** (Final State - Success)

```
Karakteristik:
- Manager telah menyetujui
- Stok sistem telah diupdate otomatis
- Transaksi penyesuaian tercatat
- Laporan dapat digenerate
- Status final (tidak dapat diubah)
```

**Hasil:**

- ✅ Stok barang updated sesuai selisih
- ✅ Audit trail tercatat
- ✅ Laporan tersedia untuk dicetak
- ✅ Data approver dan timestamp tersimpan

**Transisi keluar:**

- → **[*]** (End state): Proses selesai

#### 4. **DITOLAK** (Final State - Rejected)

```
Karakteristik:
- Manager tidak menyetujui data
- Ada catatan alasan penolakan
- Stok sistem tidak berubah
- Stock opname tidak valid
```

**Tindak lanjut:**

- 📝 Lihat alasan penolakan
- 🔄 Buat stock opname baru
- ✏️ Perbaiki data yang salah
- 📤 Submit ulang

**Transisi keluar:**

- → **[*]** (End state): Proses selesai (gagal)

### Siklus Hidup Penuh

```
[Start] → DRAFT → MENUNGGU_APPROVAL → DISETUJUI → [End]
                                    ↓
                                 DITOLAK → [End]
```

### Event-Driven State Changes

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use App\Events\OpnameSubmittedEvent;
use App\Events\OpnameApprovedEvent;
use App\Events\OpnameRejectedEvent;
use App\Exceptions\InvalidStateException;

class StockOpname extends Model
{
    const STATUS_DRAFT = 'draft';
    const STATUS_MENUNGGU_APPROVAL = 'menunggu_approval';
    const STATUS_DISETUJUI = 'disetujui';
    const STATUS_DITOLAK = 'ditolak';

    public function submit(): void
    {
        if ($this->status !== self::STATUS_DRAFT) {
            throw new InvalidStateException('Hanya DRAFT yang dapat disubmit');
        }

        if (!$this->isValid()) {
            throw new \App\Exceptions\ValidationException('Data tidak valid');
        }

        $this->update(['status' => self::STATUS_MENUNGGU_APPROVAL]);

        event(new OpnameSubmittedEvent($this));
    }

    public function approve(User $approver): void
    {
        if ($this->status !== self::STATUS_MENUNGGU_APPROVAL) {
            throw new InvalidStateException('Hanya status MENUNGGU_APPROVAL yang dapat diapprove');
        }

        $this->update([
            'status' => self::STATUS_DISETUJUI,
            'disetujui_oleh' => $approver->id,
            'tanggal_approval' => now(),
        ]);

        // Update stok untuk semua item
        foreach ($this->items as $item) {
            if ($item->selisih !== 0) {
                $item->barang->increment('stok_tersedia', $item->selisih);
            }
        }

        event(new OpnameApprovedEvent($this));
    }

    public function reject(User $rejector, string $alasan): void
    {
        if ($this->status !== self::STATUS_MENUNGGU_APPROVAL) {
            throw new InvalidStateException('Hanya status MENUNGGU_APPROVAL yang dapat direject');
        }

        $this->update([
            'status' => self::STATUS_DITOLAK,
            'ditolak_oleh' => $rejector->id,
            'alasan_penolakan' => $alasan,
        ]);

        event(new OpnameRejectedEvent($this));
    }

    private function isValid(): bool
    {
        return $this->items()->count() > 0;
    }
}
```

---

## 📊 Evaluasi Desain Sistem

### 1. **Maintainability (Kemudahan Perawatan)** ✅ BAIK

**Poin Positif:**

- ✅ **Separation of Concerns**: MVC pattern memisahkan UI, business logic, dan data
- ✅ **Single Responsibility**: Setiap kelas punya tanggung jawab tunggal
- ✅ **Clear Naming**: Nama kelas dan method self-documenting
- ✅ **Encapsulation**: Data dan behavior terkapsul dalam kelas

**Contoh Kemudahan Maintenance:**

```php
<?php

// Jika perlu ubah cara hitung selisih, hanya ubah 1 tempat
class StockOpnameService
{
    public function hitungSelisih(int $stokSistem, int $stokFisik): int
    {
        // Logika terpusat di sini
        return $stokFisik - $stokSistem;
    }
}
```

**Skor: 9/10**

### 2. **Reusability (Kemudahan Digunakan Kembali)** ✅ SANGAT BAIK

**Poin Positif:**

- ✅ **Service Layer**: Business logic dapat digunakan berbagai controller
- ✅ **Factory Pattern**: Logic pembuatan object dapat digunakan di berbagai tempat
- ✅ **Interface-based**: Komponen dapat diganti dengan implementasi lain
- ✅ **Domain Model**: `ItemOpname` dan `Barang` dapat digunakan di modul lain

**Contoh Reusability:**

```php
<?php

// StockOpnameService dapat digunakan di berbagai modul
class PurchaseOrderController extends Controller
{
    public function __construct(
        private StockOpnameService $service
    ) {}

    public function receiveGoods(int $ordered, int $received)
    {
        $selisih = $this->service->hitungSelisih($ordered, $received);
        // ...
    }
}

class SalesController extends Controller
{
    public function __construct(
        private StockOpnameService $service
    ) {}

    public function processReturn(int $sold, int $returned)
    {
        $selisih = $this->service->hitungSelisih($sold, $returned);
        // ...
    }
}
```

**Skor: 9/10**

### 3. **Extendability (Kemudahan Dikembangkan)** ✅ SANGAT BAIK

**Poin Positif:**

- ✅ **Open/Closed Principle**: Terbuka untuk ekstensi, tertutup untuk modifikasi
- ✅ **Strategy Pattern**: Mudah tambah algoritma perhitungan baru
- ✅ **Event-Driven**: Mudah tambah listener untuk notifikasi, logging, etc.
- ✅ **Enum for Status**: Mudah tambah status baru jika diperlukan

**Contoh Extensibility:**

```php
<?php

namespace App\Contracts;

use App\Events\OpnameSubmittedEvent;
use App\Events\OpnameApprovedEvent;
use App\Events\OpnameRejectedEvent;

// Mudah extend dengan fitur baru tanpa ubah kode existing
interface OpnameEventListener
{
    public function onSubmitted(OpnameSubmittedEvent $event): void;
    public function onApproved(OpnameApprovedEvent $event): void;
    public function onRejected(OpnameRejectedEvent $event): void;
}
```

```php
<?php

namespace App\Listeners;

use App\Contracts\OpnameEventListener;
use App\Events\OpnameApprovedEvent;
use App\Services\EmailService;

// Tambah fitur notifikasi email
class EmailNotificationListener implements OpnameEventListener
{
    public function __construct(
        private EmailService $emailService
    ) {}

    public function onApproved(OpnameApprovedEvent $event): void
    {
        $this->emailService->send(
            $event->opname->creator,
            'SO Approved'
        );
    }

    // ... implement other methods
}
```

```php
<?php

namespace App\Listeners;

use App\Contracts\OpnameEventListener;
use App\Events\OpnameApprovedEvent;
use Illuminate\Support\Facades\Log;

// Tambah fitur audit log
class AuditLogListener implements OpnameEventListener
{
    public function onApproved(OpnameApprovedEvent $event): void
    {
        Log::channel('audit')->info(
            "SO {$event->opname->nomor_opname} approved by {$event->opname->approver->name}"
        );
    }

    // ... implement other methods
}
```

**Skor: 9/10**

### 4. **Scalability & Performance**

**Consideration Points:**

- ⚠️ **Database Transaction**: Perlu optimasi untuk approval bulk items
- ✅ **Lazy Loading**: Item dapat di-load on-demand
- ✅ **Caching**: StockService menggunakan cache untuk performa
- ⚠️ **Concurrency**: Perlu handle concurrent stock opname untuk barang yang sama

**Skor: 7/10**

### 5. **Security**

**Implemented:**

- ✅ **Role-based Access**: Staff vs Manager memiliki akses berbeda
- ✅ **State Validation**: Hanya status tertentu yang dapat diubah
- ✅ **Audit Trail**: Track siapa yang buat/approve
- ⚠️ **Data Encryption**: Belum diimplementasikan

**Skor: 7/10**

---

## 🚀 Usulan Pengembangan di Masa Depan

### 1. **Sistem Notifikasi Otomatis** 📧

**Deskripsi:**
Implementasi sistem notifikasi real-time untuk berbagai event dalam proses stock opname.

**Fitur:**

- **Email Notification**:

  - Staff mendapat notifikasi saat SO diapprove/reject
  - Manager mendapat notifikasi saat ada SO baru menunggu approval
  - Reminder otomatis jika SO pending > 2 hari

- **Push Notification**:

  - Mobile app notification untuk approval
  - Desktop notification untuk update status

- **WhatsApp Integration**:
  - Notifikasi via WhatsApp Business API
  - Template: "SO {nomor} perlu approval. Silakan cek sistem."

**Implementasi:**

```php
<?php

namespace App\Contracts;

use App\Models\User;

interface NotificationService
{
    public function sendEmail(User $user, string $subject, string $message): void;
    public function sendPush(User $user, string $title, string $message): void;
    public function sendWhatsApp(User $user, string $message): void;
}
```

```php
<?php

namespace App\Listeners;

use App\Contracts\NotificationService;
use App\Events\OpnameSubmittedEvent;
use App\Events\OpnameApprovedEvent;
use App\Models\User;

class OpnameNotificationListener
{
    public function __construct(
        private NotificationService $notificationService
    ) {}

    public function handleSubmitted(OpnameSubmittedEvent $event): void
    {
        $manager = User::role('manager')->first();

        $this->notificationService->sendEmail(
            $manager,
            'Stock Opname Perlu Approval',
            "SO {$event->opname->nomor_opname} telah disubmit oleh {$event->opname->creator->name}"
        );
    }

    public function handleApproved(OpnameApprovedEvent $event): void
    {
        $this->notificationService->sendWhatsApp(
            $event->opname->creator,
            "✅ SO {$event->opname->nomor_opname} telah disetujui"
        );
    }
}

// Register di EventServiceProvider
// OpnameSubmittedEvent::class => [OpnameNotificationListener::class . '@handleSubmitted'],
// OpnameApprovedEvent::class => [OpnameNotificationListener::class . '@handleApproved'],
```

### 2. **Fitur Rekomendasi & Prediksi** 🤖

**Deskripsi:**
Machine learning untuk deteksi anomali dan rekomendasi jadwal stock opname.

**Fitur:**

- **Anomaly Detection**:

  - Deteksi selisih yang tidak wajar
  - Alert jika selisih > threshold tertentu
  - Pattern recognition untuk barang yang sering bermasalah

- **Schedule Recommendation**:

  - Rekomendasi jadwal stock opname berdasarkan historical data
  - Barang fast-moving → stock opname lebih sering
  - Barang slow-moving → stock opname lebih jarang

- **Predictive Analytics**:
  - Prediksi kemungkinan selisih berdasarkan pola historis
  - Identifikasi barang high-risk

**Implementasi:**

```php
<?php

namespace App\Services;

use App\Models\ItemOpname;
use App\Models\Barang;
use Illuminate\Support\Collection;

class StockOpnameAnalytics
{
    public function detectAnomaly(ItemOpname $item): ?array
    {
        // Analisis historical data (6 bulan terakhir)
        $history = ItemOpname::where('barang_id', $item->barang_id)
            ->where('created_at', '>=', now()->subMonths(6))
            ->pluck('selisih');

        if ($history->isEmpty()) {
            return null;
        }

        $avgSelisih = $history->avg();
        $stdDev = $this->calculateStdDev($history);
        $currentSelisih = $item->selisih;

        // Jika selisih > 2 standard deviasi = anomali
        if (abs($currentSelisih - $avgSelisih) > 2 * $stdDev) {
            return [
                'item' => $item,
                'message' => "Selisih tidak wajar. Rata-rata: {$avgSelisih}",
                'risk_level' => 'HIGH',
            ];
        }

        return null;
    }

    public function recommendSchedule(Barang $barang): array
    {
        // Analisis turnover rate
        $turnoverRate = $this->calculateTurnoverRate($barang);

        if ($turnoverRate > 10) {
            return [
                'barang' => $barang,
                'schedule' => 'Bulanan',
                'reason' => 'High turnover',
            ];
        } elseif ($turnoverRate > 5) {
            return [
                'barang' => $barang,
                'schedule' => 'Triwulan',
                'reason' => 'Medium turnover',
            ];
        }

        return [
            'barang' => $barang,
            'schedule' => 'Semester',
            'reason' => 'Low turnover',
        ];
    }

    private function calculateStdDev(Collection $values): float
    {
        $mean = $values->avg();
        $squaredDiffs = $values->map(fn($value) => pow($value - $mean, 2));
        return sqrt($squaredDiffs->avg());
    }

    private function calculateTurnoverRate(Barang $barang): float
    {
        // Logic untuk hitung turnover rate
        return $barang->transaksi()
            ->where('created_at', '>=', now()->subMonth())
            ->count();
    }
}
```

### 3. **Mobile Application** 📱

**Deskripsi:**
Aplikasi mobile untuk memudahkan staff melakukan stock opname langsung dari gudang.

**Fitur:**

- **Barcode/QR Scanner**: Scan barang untuk input cepat
- **Offline Mode**: Input data tanpa internet, sync otomatis saat online
- **Photo Evidence**: Foto kondisi barang sebagai bukti
- **Voice Input**: Input stok dengan suara
- **GPS Location**: Track lokasi saat penghitungan

### 4. **Dashboard & Reporting** 📊

**Deskripsi:**
Dashboard analytics untuk monitoring dan reporting yang lebih baik.

**Fitur:**

- **Real-time Dashboard**:
  - Jumlah SO pending approval
  - Trend selisih per bulan
  - Top 10 barang dengan selisih terbesar
- **Advanced Reporting**:
  - Report per kategori barang
  - Report per lokasi gudang
  - Report per staff (akurasi penghitungan)
  - Export ke Excel/PDF

---

## 📚 Referensi

1. Gang of Four - Design Patterns: Elements of Reusable Object-Oriented Software
2. Robert C. Martin - Clean Architecture
3. Martin Fowler - Patterns of Enterprise Application Architecture
4. Eric Evans - Domain-Driven Design
