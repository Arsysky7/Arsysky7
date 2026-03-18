# Software Requirements Specification (SRS)
# FOTON Unit Monitoring System

**Versi:** 2.0  
**Tanggal:** 2026-03-15  
**Status:** Draft

---

## 1. Pendahuluan

Dokumen ini mendefinisikan persyaratan arsitektur, fungsionalitas, keamanan, dan hak akses (*Roles*) untuk pembangunan ulang aplikasi **FOTON Unit Monitoring System**. Aplikasi ini akan bertransisi dari tumpukan (stack) *Frontend React + Base44 Cloud* menuju arsitektur berdaulat mandiri menggunakan kerangka **PHP Laravel (Monolitik)** dengan *database* **MySQL Lokal**.

---

## 2. Arsitektur Sistem (Technical Stack)

Aplikasi didesain secara Monolitik (*Monolith*) di mana Lapis *Backend* dan *Frontend* digabungkan ke dalam satu wadah *project* untuk memudahkan *deployment* di server kantor.

| Layer | Teknologi |
|-------|-----------|
| **Frontend (UI)** | React.js + Tailwind CSS + shadcn/ui |
| **Adapter** | Inertia.js (menghubungkan React ke Laravel tanpa REST API terpisah) |
| **Backend** | PHP v8.x + Laravel v11.x (Router, Controller, Eloquent ORM) |
| **Auth** | Sesi PHP Laravel + UUID sebagai session identifier (mencegah IDOR) |
| **Database** | MySQL v8.x dengan fitur `JSON Column` untuk spesifikasi dinamis |
| **Real-time** | Polling setiap 5 detik via React Query, siap upgrade ke Laravel Reverb WebSocket |

---

## 3. Sistem Role & Hierarki Akses

### 3.1 Prinsip Dasar

Aplikasi menggunakan **Role-Based Access Control (RBAC)** dengan prinsip:

1. **Field Team** hanya bisa mengerjakan tugasnya sesuai state yang ditugaskan — tidak bisa kelola akun apapun
2. **Sales** mengelola field team di phase/state mereka + dashboard phase 1 & 4
3. **After Sales Head (`as_head`)** adalah "Mini COO" khusus lingkup After Sales — import unit, kelola semua akun AS, akses semua dashboard AS, bisa handle trouble & God Mode dalam scope After Sales
4. **After Sales Workshop (`as_workshop`)** mengelola operasional karoseri & PDI (State 4–7)
5. **After Sales Technical (`as_technical`)** menangani seluruh trouble dari semua state
6. **COO** adalah God Mode — akses penuh ke semua dashboard dan kelola akun Sales & After Sales Head

> **Tidak ada role "admin"** dalam sistem ini. Fungsi administratif didelegasikan sesuai hierarki di atas.

---

### 3.2 Hierarki Lengkap

```
COO  (God Mode — akses menyeluruh)
│
├── SALES
│   ├── Kelola akun: Forwarder, Gudang, Kurir, Ekspedisi, Driver, PIC Sales
│   └── Dashboard: monitor unit CBU 1–3 & 10–16 / CKD 1–3 & 10–14
│       ├── Forwarder        → CBU State 1, 2, 3
│       ├── Gudang           → CBU State 10, 13 / CKD State 10, 13
│       ├── Kurir            → CBU State 11 / CKD State 11
│       ├── LOLO Ker (ker)   → CBU State 12 / CKD State 12
│       └── Ekspedisi/Driver → CBU State 14, 15 / CKD State 14
│
└── AFTER SALES HEAD (as_head) — "Mini COO" khusus After Sales
    ├── Import unit Excel
    ├── Handle trouble (semua state)
    ├── God Mode dalam scope After Sales
    ├── Kelola akun: as_workshop, as_technical, Karoseri, Foreman
    │
    ├── AS WORKSHOP (as_workshop)
    │   └── Dashboard: monitor unit State 4–9
    │       ├── Karoseri → State 4, 5, 6, 7, 8 (0%→100%)
    │       └── Foreman  → State 9 (QC/PDI)
    │
    └── AS TECHNICAL (as_technical)
        └── Handle semua trouble CBU State 1–15 / CKD State 1–14
```

### 3.3 Tabel Hak Akses per Role

| Role | Dashboard | Kelola Akun | Hak Istimewa |
|------|-----------|-------------|---------------|
| `coo` | Semua (Sales + After Sales + Field Team) | `sales`, `as_head` | God Mode penuh, Import/Export |
| `sales` | CBU 1–3 & 10–15 / CKD 1–3 & 10–14 | `forwarder`, `gudang`, `kurir`, `ker`, `ekspedisi`, `driver`, PIC Sales | — |
| `as_head` | Semua dashboard After Sales | `as_workshop`, `as_technical`, `karoseri`, `foreman` | Import unit, handle trouble, God Mode scope AS |
| `as_workshop` | State 4–9 (Workshop Operation) | `karoseri`, `foreman` | Generate link mobile |
| `as_technical` | Trouble semua state | — | Handle trouble |
| `forwarder` | — | — | CBU: State 1, 2, 3 |
| `karoseri` | — | — | CBU & CKD: State 4, 5, 6, 7, 8 (milestone 0%→100%) |
| `foreman` | — | — | CBU & CKD: State 9 (PDI) |
| `gudang` | — | — | CBU: State 10, 13 / CKD: State 10, 13 |
| `kurir` | — | — | CBU: State 11 / CKD: State 11 (Kurir masuk gudang) |
| `ker` | — | — | CBU: State 12 / CKD: State 12 (Verifikasi LOLO Ker) |
| `ekspedisi` / `driver` | — | — | CBU: State 14, 15 / CKD: State 14 |
| `na` | — | — | CKD: State 0, 1 |
| `dealer` | — | — | CKD: State 2, 3 |

---

## 4. Sistem Autentikasi & Login

### 4.1 Mekanisme Login (Opsi A — Single Login URL)

Semua pengguna menggunakan **satu halaman login** yang sama:

```
URL: /login  (atau /MobileUpdate untuk field team via link)
```

**Alur Login:**
```
[Pengguna buka halaman login]
        ↓
[Masukkan Kode Akses]
        ↓
[Sistem validasi kode → cek role di tabel access_codes]
        ↓
┌────────────────────────────────────────┐
│ Role?                                   │
├──────────┬──────────┬──────────────────┤
│ coo      │ sales    │ after_sales      │ → Redirect: /dashboard/[role]
├──────────┤          │                  │
│         field team (forwarder, dll)    │ → Redirect: /mobile (halaman update state)
└────────────────────────────────────────┘
```

**Field Team** mendapatkan link langsung dengan kode tersemat:
```
https://[domain]/MobileUpdate?code=XXXXXXXX
```
Link ini di-generate oleh Sales (untuk field team mereka) atau After Sales (untuk field team mereka).

### 4.2 Perlindungan Akun COO

- Akun COO **tidak bisa diubah** oleh Sales atau After Sales
- Hanya COO sendiri yang bisa mengubah kode aksesnya sendiri
- Akun COO dijaga di level aplikasi (middleware check)

### 4.3 Tabel `access_codes`

| Field | Tipe | Keterangan |
|-------|------|------------|
| `id` | UUID (PK) | Primary key |
| `code` | VARCHAR(50) | Kode akses unik |
| `name` | VARCHAR(200) | Nama pengguna |
| `role` | ENUM | Lihat daftar role di §3.3 |
| `phone` | VARCHAR(20) | Nomor WhatsApp |
| `division` | VARCHAR(100) | Divisi (misal: Technical) |
| `location` | VARCHAR(100) | Lokasi/area kerja |
| `is_active` | BOOLEAN | Status aktif |
| `last_access` | DATETIME | Timestamp login terakhir |
| `created_by` | UUID (FK) | ID pembuat akun |

---

## 5. Use Case Diagram

```mermaid
flowchart TD
    %% =================== AKTOR ===================
    FT(["👷 Field Team\n─────────────────\nForwarder\nKaroseri\nForeman\nGudang\nKurir\nKer (LOLO Ker)\nEkspedisi / Driver\nNA / Dealer"])
    SALES(["🔵 Sales"])
    AS(["🟣 After Sales"])
    COO(["🔴 COO\n(God Mode)"])

    %% =================== USE CASES ===================
    subgraph UC_AUTH ["🔐 Autentikasi"]
        LOGIN["Login via Kode Akses"]
        REDIRECT["Auto-Redirect ke Dashboard Sesuai Role"]
    end

    subgraph UC_FIELD ["📱 Field Team — Mobile"]
        SCAN["Scan SIN / Input VIN"]
        UPDATE["Update State Unit"]
        PDI["Isi Form PDI\n(Foreman Only)"]
        TROUBLE_RPT["Lapor Trouble"]
    end

    subgraph UC_SALES ["🔵 Sales Dashboard"]
        S_DASH["Pantau Unit\nCBU: State 1–3 & 10–15\nCKD: State 1–3 & 10–14"]
        S_KPI["Lihat KPI & Progress\nPhase 1 dan Phase 4"]
        S_ACC["Kelola Akun\nForwarder, Gudang, Kurir, Ker\nEkspedisi, Driver, PIC Sales"]
        S_LINK["Generate Link Mobile\nUntuk Field Team"]
        S_EXPORT["Export Excel BAST\n(Filter Tanggal, Tipe, Karoseri)"]
    end

    subgraph UC_AS ["🟣 After Sales"]
        subgraph UC_AS_HEAD ["👑 as_head — After Sales Head"]
            AH_IMPORT["Import Unit Excel"]
            AH_ACC["Kelola Akun\nas_workshop, as_technical,\nKaroseri, Foreman"]
            AH_GOD["God Mode Scope AS\n(Edit State Manual)"]
            AH_TROUBLE["Handle Trouble\n(Semua State)"]
        end
        subgraph UC_AS_WO ["A — as_workshop: Workshop Operation"]
            A_DASH["Pantau Unit State 4–9\n(Karoseri & PDI)"]
            A_KPI["KPI Progress Karoseri\n0% / 25% / 50% / 75% / 100% + PDI"]
            A_LINK["Generate Link Mobile\nUntuk Field Team"]
        end
        subgraph UC_AS_TECH ["B — as_technical: Technical (HO)"]
            T_RESP["Beri Instruksi / ho_response\n(Semua Trouble CBU 1–15\n/ CKD 1–14)"]
            T_SOLVE["Tandai Trouble Selesai\n+ Isi Solusi"]
        end
    end

    subgraph UC_COO ["🔴 COO — Dashboard Menyeluruh"]
        C_ALL["Pantau Semua Unit\n(Semua State)"]
        C_SALES_VIEW["Lihat Dashboard Sales"]
        C_AS_VIEW["Lihat Dashboard After Sales"]
        C_ACC["Kelola Akun Sales\n& After Sales"]
        C_GOD["Update State Bebas\n(God Mode — Bypass Semua Aturan)"]
        C_IMPORT["Import & Export\nData Unit (Excel)"]
    end

    subgraph UC_TROUBLE ["⚠️ Trouble — Dilaporkan Field Team"]
        T_NOTIF["Lapor Trouble + Foto\n+ Kronologi + Lokasi"]
        T_LOCK["Unit Terkunci (Trouble Lock)\nTidak Bisa Update State"]
    end

    %% =================== KONEKSI ===================
    FT --> LOGIN
    SALES --> LOGIN
    AS --> LOGIN
    COO --> LOGIN
    LOGIN --> REDIRECT

    FT --> SCAN --> UPDATE
    FT --> PDI
    FT --> T_NOTIF --> T_LOCK

    SALES --> S_DASH
    SALES --> S_KPI
    SALES --> S_ACC
    SALES --> S_LINK
    SALES --> S_EXPORT

    AS --> AH_IMPORT
    AS --> AH_ACC
    AS --> AH_GOD
    AS --> AH_TROUBLE
    AS --> A_DASH
    AS --> A_KPI
    AS --> A_LINK
    AS --> T_RESP
    AS --> T_SOLVE

    COO --> C_ALL
    COO --> C_SALES_VIEW
    COO --> C_AS_VIEW
    COO --> C_ACC
    COO --> C_GOD
    COO --> C_IMPORT
    COO --> T_RESP
    COO --> T_SOLVE

---

## 6. Alur Progres Unit (State Pipeline)

### 6.1 Alur CBU (Completely Built-Up) — 16 States (State 0–15)

Setiap update state dilakukan dengan **Scan SIN** (Stiker Identifikasi Kendaraan yang ditempel saat unit tiba di Priok).

```
State 0 → 1 → 2 → 3 → [4→5→6→7→8→9] → 10  → 11  → 12       → 13 → 14  → 15
  Priok   Forwarder    ←── After Sales ──→  Gudang  Kurir  LOLO Ker  Gudang  Ekspedisi SELESAI
          (Sales)      Karoseri 0-100% PDI  Masuk          Verif.  Keluar
```

| State | Nama | Role | Foto Wajib | Keterangan | Dashboard |
|-------|------|------|------------|------------|-----------|
| 0 | Di Priok | — | — | Belum keluar pabrik | Sales |
| 1 | Keluar Pabrik/Priok | `forwarder` | ✅ | | Sales |
| 2 | Foto Tiba di Karoseri | `forwarder` | ✅ | | Sales |
| 3 | Tiba di Karoseri | `forwarder` | ✅ | | Sales |
| 4 | Karoseri 0% | `karoseri` | ✅ | Mulai pengerjaan | After Sales |
| 5 | Karoseri 25% | `karoseri` | ✅ | Bak naik chassis | After Sales |
| 6 | Karoseri 50% | `karoseri` | ✅ | Aksesoris (side guard, back guard, frame tangga) | After Sales |
| 7 | Karoseri 75% | `karoseri` | ✅ | Painting luar dalam bak | After Sales |
| 8 | Karoseri 100% | `karoseri` | ✅ | Finishing & wiring (lampu, sensor parkir) | After Sales |
| 9 | QC / PDI | `foreman` | ✅ | Lihat PDI Fork | After Sales |
| 10 | Gudang Masuk | `gudang` | ✅ | Konfirmasi unit masuk gudang | Sales |
| 11 | Kurir | `kurir` | ✅ | Kurir mencatat penerimaan unit di gudang | Sales |
| 12 | Verifikasi LOLO Ker | `ker` | ✅ | Lolos / Tidak Lolos — wajib sebelum keluar | Sales |
| 13 | Gudang Keluar | `gudang` | ✅ | Konfirmasi unit keluar gudang | Sales |
| 14 | Keluar ke Customer | `ekspedisi`, `driver` | ✅ | Perjalanan ke customer | Sales |
| 15 | SELESAI | `ekspedisi`, `driver` | ✅ + Alamat + BAST | Foto BAST + Alamat lengkap | Sales |

**Catatan State 0 (Di Priok):**
Sebelum State 1, unit sudah ada di sistem (diimport via Excel) tapi belum dikonfirmasi keluar. Aktivitas di Priok:
- Buka kontainer
- General Inspection (GI)
- Penempelan Stiker SIN (Stiker Identifikasi Kendaraan)

### 6.2 Alur CKD (Completely Knocked-Down) — 15 States (State 0–14)

```
State 0 → 1 → 2 → 3 → [4→5→6→7→8→9] → 10  → 11  → 12       → 13 → 14
  NA      Dealer     ←── After Sales ──→  Gudang  Kurir  LOLO Ker  Gudang  DO
                     Karoseri 0-100% PDI  Masuk          Verif.  Keluar  SELESAI
```

> **Catatan Alur:** Setelah State 9 (PDI), unit **wajib** melewati State 10 (Gudang Masuk) → 11 (Kurir) → 12 (LOLO Ker) → 13 (Gudang Keluar) → baru bisa State 14 (DO). Pola **identik dengan CBU**.

| State | Nama | Role | Foto Wajib | Keterangan | Dashboard |
|-------|------|------|------------|------------|-----------|
| 0 | PDI di NA | `na` | ✅ | | After Sales |
| 1 | Keluar ke Dealer | `na`, `dealer` | ✅ | | Sales |
| 2 | Dealer | `dealer` | ✅ | | Sales |
| 3 | Keluar ke Karoseri | `dealer`, `karoseri` | ✅ | | Sales |
| 4 | Karoseri 0% | `karoseri` | ✅ | Mulai pengerjaan | After Sales |
| 5 | Karoseri 25% | `karoseri` | ✅ | Bak naik chassis | After Sales |
| 6 | Karoseri 50% | `karoseri` | ✅ | Aksesoris (side guard, back guard, frame tangga) | After Sales |
| 7 | Karoseri 75% | `karoseri` | ✅ | Painting luar dalam bak | After Sales |
| 8 | Karoseri 100% | `karoseri` | ✅ | Finishing & wiring (lampu, sensor parkir) | After Sales |
| 9 | QC / PDI | `foreman` | ✅ | | After Sales |
| 10 | Gudang Masuk | `gudang` | ✅ | Konfirmasi unit masuk gudang | Sales |
| 11 | Kurir | `kurir` | ✅ | Kurir mencatat penerimaan unit di gudang | Sales |
| 12 | Verifikasi LOLO Ker | `ker` | ✅ | Lolos / Tidak Lolos — wajib sebelum keluar | Sales |
| 13 | Gudang Keluar | `gudang` | ✅ | Konfirmasi unit keluar gudang | Sales |
| 14 | DO / Keluar ke Customer (SELESAI) | `ekspedisi`, `driver` | ✅ + Alamat + BAST | Foto BAST + Alamat lengkap | Sales |

### 6.3 Aturan State (Sistem Tongkat Estafet)

1. **Estafet Sequential** — Role hanya bisa update ke state berikutnya dari posisi saat ini. Tidak bisa loncat state.
2. **Trouble Lock** — Jika unit memiliki trouble berstatus `open` atau `waiting_ho`, tombol update **dikunci**. Unit harus menyelesaikan trouble dahulu.
3. **PDI Fork:**
   - ✅ PDI Good → state lanjut normal
   - ❌ PDI Not Good → wajib buat laporan Trouble, state tidak bisa maju
4. **State 15 CBU / State 14 CKD (SELESAI)** — Wajib mengisi: Foto BAST + Alamat (Provinsi, Kota, Alamat Lengkap)
5. **Gudang Wajib 4 Tahap (CBU & CKD)** — Setelah PDI, unit wajib melewati:
   - State Gudang Masuk (`gudang`) → State Kurir (`kurir`) → State LOLO Ker (`ker`) → State Gudang Keluar (`gudang`)
   - Tidak bisa loncat, sequential ketat untuk semua unit.
6. **LOLO Ker Fork:**
   - ✅ Lolos → state lanjut ke Gudang Keluar
   - ❌ Tidak Lolos → wajib buat laporan Trouble, unit dikunci sampai trouble `solved`
7. **Timestamp Wajib** — Setiap update state **wajib mencatat jam dan tanggal** (format: DD/MM/YYYY HH:mm). Tampil di riwayat unit dan history dashboard.

---

## 7. Dashboard Spesifikasi

### 7.1 Dashboard Sales

**Akses:** Role `sales` dan `coo`

**Konten:**
| Komponen | Deskripsi |
|----------|-----------|
| KPI Cards | Total unit di phase Sales (State 1-3 & 8-11), completed, trouble |
| Flow Overview | Visualisasi jumlah unit per-state di Phase 1 & 4 |
| Unit Table | Tabel unit dengan filter state, search VIN |
| Detail Unit | Lihat info lengkap unit termasuk technical specs |
| History | Riwayat update state per unit |
| Trouble Snapshot | Trouble aktif di unit-unit fase Sales |
| Manajemen Akun | Buat/edit/nonaktifkan akun: Forwarder, Gudang, Ekspedisi, Driver, PIC Sales |
| Generate Link | Buat link mobile untuk field team |

**Filter State yang tampil:**
- CBU: State 1, 2, 3, 10, 11, 12, 13, 14, 15
- CKD: State 1, 2, 3, 10, 11, 12, 13, 14

---

### 7.2 Dashboard After Sales

After Sales terbagi menjadi **3 role** dengan dashboard masing-masing:

---

#### 👑 as_head — After Sales Head ("Mini COO" Scope AS)

**Akses:** Role `as_head` dan `coo`

| Komponen | Deskripsi |
|----------|-----------|
| Sales View AS | Embed dashboard Workshop & Technical |
| KPI Global AS | KPI semua unit After Sales tanpa filter |
| Import/Export Excel | Upload template unit, download report |
| Manajemen Akun AS | Buat/edit akun `as_workshop`, `as_technical`, `karoseri`, `foreman` |
| God Mode Scope AS | Edit state unit manual dalam lingkup After Sales |
| Handle Trouble | Bisa beri instruksi & selesaikan trouble (semua state) |

---

#### A — as_workshop: Workshop Operation

**Akses:** Role `as_workshop`, `as_head`, dan `coo`

Menangani operasional proses fisik unit di karoseri dan PDI.

| Komponen | Deskripsi |
|----------|-----------|
| KPI Cards | Total unit di State 4–9, PDI status (passed/failed/pending) |
| Flow Overview | Visualisasi progress karoseri: 0% → 25% → 50% → 75% → 100% → PDI |
| Karoseri Panel | Unit per karoseri + progress pengerjaan + aging alert |
| Unit Table | Tabel unit dengan filter state, karoseri, progress (0/25/50/75/100%) |
| PDI Summary | Ringkasan hasil PDI: passed/failed/pending per unit |
| Generate Link | Buat link mobile untuk Karoseri & Foreman |

**Filter State yang tampil (Workshop Operation):**
- CBU: State 4, 5, 6, 7, 8, 9
- CKD: State 4, 5, 6, 7, 8, 9 (dan State 0 PDI di NA)

---

#### B — as_technical: Technical (Home Office)

**Akses:** Role `as_technical`, `as_head`, dan `coo`

Menangani seluruh **Trouble Handling** dari **semua state 1–15** (CBU) dan **1–14** (CKD).

| Komponen | Deskripsi |
|----------|-----------|
| Trouble List | Semua trouble aktif dari SEMUA state & SEMUA unit |
| Filter Trouble | Filter by status: `open`, `waiting_ho`, `solved` |
| Detail Trouble | Lihat foto, kronologi, lokasi, rute pelapor |
| Beri Instruksi | Isi `ho_response` → status berubah ke `waiting_ho` |
| Selesaikan Trouble | Isi `solution` → status berubah ke `solved` → unit `active` kembali |
| Trouble History | Riwayat trouble yang sudah solved |

> **as_technical** dan **as_head** (selain COO) yang bisa memberikan instruksi dan menyelesaikan trouble — berapapun state unit saat trouble terjadi.

---

### 7.3 Dashboard COO (Menyeluruh)

**Akses:** Role `coo` saja

**Konten:**
| Komponen | Deskripsi |
|----------|-----------|
| Sales View | Embed/tab Dashboard Sales |
| After Sales View | Embed/tab Dashboard After Sales |
| KPI Global | KPI semua unit tanpa filter |
| Trouble Menyeluruh | Semua trouble dari semua phase |
| Import/Export Excel | Upload template unit, download report |
| Manajemen Akun COO | Buat/edit akun Sales & After Sales |
| FlowEdit (God Mode) | Edit state unit secara manual, bypass semua aturan |
| Training Guide | Panduan penggunaan sistem |

> ⚠️ **Tidak ada "Summary" tab** — digantikan oleh Sales View dan After Sales View.

---

## 8. Modul Manajemen Akun

### 8.1 Siapa Bisa Buat Akun Siapa

| Yang Membuat | Role yang Bisa Dibuat |
|-------------|----------------------|
| `coo` | `sales`, `as_head` |
| `sales` | `forwarder`, `gudang`, `kurir`, `ker`, `ekspedisi`, `driver`, PIC `sales` |
| `as_head` | `as_workshop`, `as_technical`, `karoseri`, `foreman` |
| `as_workshop` | `karoseri`, `foreman` (generate link saja, tidak buat akun baru) |
| `as_technical` | ❌ Tidak bisa buat akun |
| Field Team | ❌ Tidak bisa buat akun |

### 8.2 Fitur Manajemen Akun

- **Buat akun baru** — isi nama, role, kode akses, nomor HP
- **Generate kode akses** — auto-generate kode random 8 karakter
- **Copy link mobile** — salin link lapangan siap kirim via WA
- **Aktif/nonaktif akun** — toggle tanpa hapus data
- **Ubah kode akses** — hanya bisa ubah akun yang berada di bawah hierarkinya
- **COO tidak bisa diedit** oleh Sales atau After Sales

---

## 9. Modul Data Master & Import

### 9.1 File Template Resmi: `template_import_unit.xlsx`

Role `as_head` dan `coo` **wajib** menggunakan template ini untuk upload unit baru ke sistem. Template tersedia via tombol **"⬇ Download Template"** di dashboard masing-masing.

**Kolom Wajib:**

| Kolom | Validasi |
|-------|----------|
| `VIN` | Unik, 17 karakter, uppercase |
| `Engine No` | Wajib isi |
| `Unit Type` | Hanya: `CBU` atau `CKD` |
| `Model` | Wajib isi |
| `Warna` | Wajib isi |

**Kolom Spesifikasi (Spec:) — Opsional:**

Semua kolom ber-prefix `Spec:` bersifat opsional, sistem simpan `null` jika kosong.

Format header: `Spec: KATEGORI: Nama Field`  
Contoh: `Spec: ENGINE: Type` → tersimpan sebagai `{"ENGINE": {"Type": "Diesel"}}`

**Pipeline Import:**
```
Upload .xlsx
  → Baca header kolom
  → Kolom tanpa "Spec:" → kolom biasa di tabel units
  → Kolom "Spec:" → dipecah jadi nested JSON → disimpan ke kolom technical_specs
  → VIN sudah ada → UPDATE | VIN baru → INSERT
  → Tampilkan ringkasan: X berhasil, Y gagal + download log error
```

### 9.2 Daftar Kolom Spesifikasi Lengkap

| No | Header Kolom Excel | Contoh Nilai |
|----|-------------------|--------------| 
| 1 | `Spec: MODEL: Drive System` | BJ1088VEJEA-FR |
| 2 | `Spec: ENGINE: Make & Model` | Cummins ISF3.8s4R154 |
| 3 | `Spec: ENGINE: Type` | Diesel, Turbocharged, Intercooled, Four-Stroke |
| 4 | `Spec: ENGINE: Number of Cylinders` | 4 Cylinders In-Line |
| 5 | `Spec: ENGINE: Displacement (L)` | 3.76 |
| 6 | `Spec: ENGINE: Rated Max. Output` | 154 PS (115kW) @ 2,600 rpm |
| 7 | `Spec: ENGINE: Rated Max. Torque` | 500 N-m (50.9kg-m) @ 1,200-1,900 rpm |
| 8 | `Spec: ENGINE: Emission Level` | Euro 4 |
| 9 | `Spec: ENGINE: Fuel Injection System` | ECU Controlled, Bosch |
| 10 | `Spec: TRANSMISSION: Model` | ZF6S506TO |
| 11 | `Spec: TRANSMISSION: Type` | 6-Speed Manual |
| 12 | `Spec: TRANSMISSION: Gear Ratio 1st` | 6.198 |
| 13 | `Spec: TRANSMISSION: Gear Ratio 2nd` | 3.287 |
| 14 | `Spec: TRANSMISSION: Gear Ratio 3rd` | 2.025 |
| 15 | `Spec: TRANSMISSION: Gear Ratio 4th` | 1.371 |
| 16 | `Spec: TRANSMISSION: Gear Ratio 5th` | 1.000 |
| 17 | `Spec: TRANSMISSION: Gear Ratio 6th` | 0.780 *(kosong jika 5-speed)* |
| 18 | `Spec: TRANSMISSION: Gear Ratio Reverse` | 5.681 |
| 19 | `Spec: TRANSMISSION: Gear Ratio Final` | 4.333 |
| 20 | `Spec: WEIGHT: Gross Vehicle Weight (kg)` | 8,250 |
| 21 | `Spec: WEIGHT: Curb Weight Rear All Model (kg)` | 2,995 |
| 22 | `Spec: WEIGHT: Curb Weight Front (kg)` | 1,860 |
| 23 | `Spec: WEIGHT: Curb Weight Rear (kg)` | 1,135 |
| 24 | `Spec: WEIGHT: Fuel Tank Capacity (l)` | 200 |
| 25 | `Spec: AXLE: Front Type` | Reversed Elliot, I-Beam |
| 26 | `Spec: AXLE: Front Axle Design Capacity (kg)` | 4,000 |
| 27 | `Spec: AXLE: Rear Type` | Banjo Type, Full Floating |
| 28 | `Spec: AXLE: Rear Axle Design Capacity (kg)` | 6,500 |
| 29 | `Spec: SUSPENSION: Front` | 8-Leaf Spring with Hydraulic Shock Absorber |
| 30 | `Spec: SUSPENSION: Rear` | 8+6-Leaf Spring with Hydraulic Shock Absorber |
| 31 | `Spec: TYRES & WHEELS: Tyres` | 7.50R16 |
| 32 | `Spec: TYRES & WHEELS: Wheels` | 16 x 6.00 |
| 33 | `Spec: BRAKES: Service` | Air Braking Dual Circuit with ABS |
| 34 | `Spec: BRAKES: Parking Brake Type` | Pneumatic Controlled Spring Brake |
| 35 | `Spec: BRAKES: Auxiliary` | Exhaust Brake |
| 36 | `Spec: STEERING: System` | Hydraulic Power Assisted |
| 37 | `Spec: STEERING: Min. Turning Radius (m)` | 6.7 |
| 38 | `Spec: ELECTRICAL: Battery` | 24V, 100 AH, 2pcs |
| 39 | `Spec: DIMENSION: Wheelbase (mm)` | 3,360 |
| 40 | `Spec: DIMENSION: Overall Length (mm)` | 5,960 |
| 41 | `Spec: DIMENSION: Overall Width (mm)` | 2,030 |
| 42 | `Spec: DIMENSION: Overall Height (mm)` | 2,260 |
| 43 | `Spec: DIMENSION: Front Overhang (mm)` | 1,110 |
| 44 | `Spec: DIMENSION: Rear Overhang (mm)` | 1,420 |
| 45 | `Spec: DIMENSION: Front Tread (mm)` | 1,590 |
| 46 | `Spec: DIMENSION: Rear Tread (mm)` | 1,534 |

---

## 10. Modul Trouble Handling

> **Penting:** Seluruh trouble dari **semua state (1–15 CBU / 1–14 CKD)** ditangani oleh **After Sales divisi Technical**. Sales hanya melihat trouble di dashboardnya sebagai informasi, tetapi **tidak berwenang** memberi instruksi atau menyelesaikan trouble.

### 10.1 Alur Status Trouble

```
[Field Team]          [After Sales Technical / COO]
     │                          │
     ▼                          ▼
  open ──── notifikasi ───► waiting_ho ──── instruksi ───► solved
 (lapor)    ke sistem HO    (beri arahan)   + solusi      (unit active)
```

| Status | Siapa Aksi | Yang Dilakukan |
|--------|------------|----------------|
| `open` | **Field Team** | Lapor trouble: tipe, kronologi, foto, lokasi |
| `waiting_ho` | **After Sales Technical** atau **COO** | Beri instruksi via field `ho_response` |
| `solved` | **After Sales Technical** atau **COO** | Isi `solution` → selesaikan → unit kembali `active` |

### 10.2 Form Laporan Trouble (Field Team — Mobile)

Wajib diisi:
- **Tipe Trouble** — dropdown: `driver`, `karoseri`, `gudang`, `transit`
- **Rute** — dari mana ke mana kejadiannya
- **Kronologi / Deskripsi** — narasi selengkap mungkin
- **Foto Dashboard** — foto odometer/dashboard unit wajib
- **Foto Kerusakan** — opsional
- **Lokasi Trouble** — kota/alamat kejadian

> Trouble bisa dilaporkan dari **state mana saja** (CBU: 1–15 / CKD: 1–14). Khusus **LOLO Ker (State 12)**: jika Tidak Lolos, otomatis membuat trouble dan mengunci unit.

### 10.3 Trouble Lock

Jika unit punya trouble berstatus `open` atau `waiting_ho`:
- Tombol update state di mobile **dikunci**
- Banner merah muncul: *"🔒 Update Dikunci — Trouble Aktif"*
- Unit tidak bisa maju ke state berikutnya sampai trouble `solved`


---

## 11. Export Excel BAST

### 11.1 Deskripsi

Fitur export untuk menghasilkan file **Excel (.xlsx)** berisi data unit yang sudah SELESAI, lengkap dengan foto BAST dan foto PDI yang tertanam (embed) di dalam file.

### 11.2 Filter Export

Sebelum export, user dapat memfilter data berdasarkan:

| Filter | Opsi |
|--------|------|
| **Tanggal** | Pilih hari, bulan, tahun (range tanggal SELESAI) |
| **Tipe Unit** | CBU / CKD / Semua |
| **Karoseri** | Pilih vendor karoseri atau Semua |
| **Status** | SELESAI saja (default) |

> Tombol **"Export Excel"** hanya aktif setelah filter dipilih.

### 11.3 Kolom di File Excel

| Kolom | Keterangan |
|-------|------------|
| VIN | Nomor identifikasi kendaraan |
| Engine No | Nomor mesin |
| Tipe | CBU / CKD |
| Model | Model unit |
| Karoseri | Vendor karoseri |
| Tgl Masuk | Tanggal unit di-import ke sistem |
| Tgl PDI | Tanggal & jam QC/PDI selesai |
| Tgl Selesai | Tanggal & jam unit SELESAI |
| Provinsi | Lokasi serah terima |
| Kota | Lokasi serah terima |
| Alamat | Alamat lengkap serah terima |
| Foto PDI | Gambar/foto PDI (embed di Excel) |
| Foto BAST | Gambar/foto surat terima unit (embed di Excel) |

### 11.4 Format File

- Format: **Excel (.xlsx)**
- Gambar ter-embed langsung di dalam sel / baris yang sesuai
- Nama file otomatis: `BAST_Export_[TglAwal]_[TglAkhir].xlsx`
- Foto yang gagal diambil akan diganti placeholder teks: `[Foto tidak tersedia]`

> **Kenapa Excel bukan CSV?** CSV tidak mendukung gambar. Excel (.xlsx) mendukung embed gambar sehingga laporan bisa langsung diprint atau difilter tanpa kehilangan visual.

---

## 12. Modul Part Code (Spare Parts)

Belum diimplementasikan di frontend. Akan dibangun di backend Laravel.

| Field | Tipe | Keterangan |
|-------|------|------------|
| `id` | UUID | Primary Key |
| `part_code` | VARCHAR(50) | Kode unik (cth: `ISF38-FAN-001`) |
| `part_name` | VARCHAR(200) | Nama komponen |
| `category` | VARCHAR(100) | Kategori (Engine, Rem, dll) |
| `is_active` | BOOLEAN | Aktif/nonaktif |

- Dikelola COO dari dashboard
- Di form trouble, field team bisa input Part Code → **auto-complete** menampilkan nama
- Satu trouble bisa punya **lebih dari 1 part rusak** (tabel `trouble_items`)

---

## 12. Database Schema Utama

### 12.1 Tabel `units`

| Field | Tipe | Keterangan |
|-------|------|------------|
| `id` | UUID (PK) | |
| `vin` | VARCHAR(17) UNIQUE | Vehicle Identification Number |
| `engine_number` | VARCHAR(50) | Nomor mesin |
| `unit_type` | ENUM(CBU,CKD) | |
| `model` | VARCHAR(100) | |
| `color` | VARCHAR(50) | |
| `current_state` | TINYINT | 0–15 CBU / 0–14 CKD |
| `current_location` | VARCHAR(100) | |
| `progress_percent` | TINYINT | 0, 25, 50, 75, 100 (milestone karoseri) |
| `status` | ENUM(active, completed, trouble) | |
| `forwarder` | VARCHAR(100) | |
| `delivery_letter` | VARCHAR(100) | Nomor surat jalan |
| `driver_name` | VARCHAR(100) | |
| `driver_phone` | VARCHAR(20) | |
| `dealer` | VARCHAR(100) | |
| `karoseri` | VARCHAR(100) | |
| `karoseri_entry_date` | DATE | |
| `pic_name` | VARCHAR(100) | |
| `customer_name` | VARCHAR(100) | |
| `entry_date` | DATE | |
| `estimated_completion` | DATE | |
| `completion_date` | DATE | |
| `photos` | JSON | Array URL foto |
| `pdi_status` | ENUM(pending, passed, failed) | |
| `pdi_date` | DATE | |
| `pdi_foreman` | VARCHAR(100) | |
| `pdi_notes` | TEXT | |
| `pdi_checklist` | JSON | {engine_ok, brake_ok, electrical_ok, body_ok, ac_ok, tyre_ok, fuel_ok, document_ok} |
| `pdi_photos` | JSON | Array URL foto PDI |
| `technical_specs` | JSON | Nested JSON spesifikasi teknis |

### 12.2 Tabel `unit_histories`

| Field | Keterangan |
|-------|------------|
| `id` | UUID (PK) |
| `unit_id` | FK ke units |
| `vin` | VIN unit |
| `state` | Nomor state yang dicapai |
| `state_name` | Nama state |
| `location` | Lokasi saat state ini |
| `progress_percent` | Progress % |
| `pic_name` | Nama yang update |
| `pic_role` | Role yang update |
| `notes` | Catatan + alamat (untuk state SELESAI) |
| `photos` | JSON — URL foto |
| `created_at` | Timestamp otomatis |

---

## 13. Real-Time & Auto Refresh

- Dashboard polling **setiap 5 detik** via React Query `refetchInterval: 5000`
- Tombol **Manual Refresh** dengan timestamp terakhir refresh
- Saat field team update state → dashboard Sales/After Sales/COO otomatis memperbarui
- Siap upgrade ke **Laravel Reverb (WebSocket)** tanpa perubahan arsitektur besar

---

## 14. Catatan Migrasi (Base44 → Laravel)

| Komponen | Sekarang (Base44) | Target (Laravel) |
|----------|-------------------|------------------|
| Auth | `base44.entities.AccessCode.list()` | Session + Middleware Laravel |
| CRUD | `base44.entities.X.list/create/update/delete` | Eloquent + Controller |
| File Upload | `base44.integrations.Core.UploadFile()` | Laravel Storage (local/S3) |
| Real-time | `base44.entities.X.subscribe()` | Polling / Laravel Reverb |
| Routing | React SPA + BrowserRouter | Inertia.js + Laravel Router |
| UUID | Auto dari Base44 | `Str::uuid()` Laravel + DB UUID PK |
