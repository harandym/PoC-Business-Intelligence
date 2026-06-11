# Laporan PoC BI: Evaluasi Apache Superset

Laporan ini merangkum analisis perbandingan dua pendekatan arsitektur untuk visualisasi data menggunakan **Apache Superset** yang dikoneksikan ke database analitis (OLAP), serta memberikan rekomendasi optimasi yang sesuai dengan kenyataan operasional (infrastruktur, stabilitas, dan target deployment server).

---

## 1. Pendahuluan: Apache Superset
**Apache Superset** adalah platform visualisasi data modern, cepat, dan tangguh yang digunakan untuk mengeksplorasi data mentah maupun data olahan dalam bentuk dashboard dan grafik interaktif.

### Fitur Utama Apache Superset:
1. **SQL Lab**: IDE SQL interaktif untuk melakukan eksplorasi query secara langsung ke database.
2. **Dataset Management**: Pendaftaran tabel dari database, serta pengelolaan metadata seperti:
   - Pelabelan kolom (*human-readable names*).
   - Pengaturan format tanggal dan waktu (*datetime format*).
   - Pembuatan kolom kalkulasi (*calculated columns*) menggunakan sintaks SQL database sumber.
3. **Konektivitas Database**: Mendukung koneksi langsung ke berbagai database (PostgreSQL, ClickHouse, MySQL, dll.) melalui konektor SQLAlchemy.
4. **Keamanan & Manajemen Akses (RBAC)**: Pembagian hak akses (dashboard, chart, dataset, koneksi database) berdasarkan User, Role, dan Group.
5. **Kemampuan Ekspor & Berbagi**:
   - Ekspor data hasil query ke CSV/Excel.
   - Ekspor chart/dashboard ke format gambar (PNG) atau PDF.
   - Berbagi dashboard via tautan publik (*share link*) atau menempelkan chart (*embed*).
6. **Laporan & Peringatan Terjadwal**: Pengiriman visualisasi otomatis melalui Email atau Slack secara berkala (memerlukan arsitektur asinkron dengan Celery dan Redis).

---

## 2. Arsitektur Konsep 1: Enterprise Full-Stack (Heavyweight)
Konsep ini mengedepankan integrasi lengkap antara database transaksional (OLTP), database analitis (OLAP), data pipeline (ETL/CDC), semantic layer, dan visualisasi data.

### Diagram Alir Data (Konsep 1)
```
[ OLTP Database (Postgres) ]
            │
            ▼ (CDC / Query Replication via PeerDB)
       [ PeerDB ] ──> [ MinIO (Staging CSV) ]
            │
            ▼ (Bulk Insert / COPY)
   [ ClickHouse OLAP ] (Flat Tables & Materialized Views)
            │
            ▼
       [ Cube.js ] (Semantic Layer & Pre-aggregations)
            │
            ▼
[ Apache Superset / PowerBI / Tableau ] (BI Tools)
```

### Detail Komponen & Container (Semuanya Wajib):
- `superset-app` (aplikasi utama Apache Superset).
- `superset-init` (kontainer inisialisasi database metadata, admin, & role).
- `redis` (caching metadata dashboard & chart untuk mempercepat loading).
- `celery-worker` (prosesor query asinkron di latar belakang).
- `celery-beat` (scheduler trigger laporan terjadwal otomatis).
- `postgres` (database untuk metadata internal Superset).
- `clickhouse-server` (engine database OLAP columnar).
- `cube-core` (OLAP Cube & semantic layer server terpusat).
- `peerdb-ui` (antarmuka web manajemen data pipeline visual).
- `peerdb-server` (core engine transfer data dari OLTP ke ClickHouse).
- `flow-worker` (pengeksekusi job transfer data).
- `flow-snapshot` (penanganan inisialisasi dump awal database).
- `flow-api` (penyedia API untuk integrasi data PeerDB).
- `minio` (object storage lokal sebagai penampung CSV temporal).

### Kelebihan & Justifikasi:
* **Single Source of Truth**: Semantic layer terpusat di Cube.js membantu menjaga agar definisi metrik dan label kolom tetap konsisten meskipun diakses dari berbagai BI tools (Superset, Power BI, Tableau).
* **ETL Mudah Dikelola**: Proses pemindahan data dari OLTP ke OLAP terpantau penuh melalui antarmuka web GUI PeerDB.
* **Performa Tinggi**: Query analitis dapat dieksekusi dengan latensi rendah berkat optimasi columnar database ClickHouse dan pre-aggregations Cube.js.

### Kekurangan & Tantangan Operasional:
* **Kerentanan Lonjakan Resource (Spikes)**: Pemakaian RAM dasar sebenarnya tidak terlalu besar, namun rawan kehabisan resource RAM & CPU secara ekstrem pada spesifikasi device lokal (seperti Intel i3-1115G4 dan RAM 8GB) ketika terjadi aktivitas bersamaan (proses ETL sinkronisasi berjalan saat user memuat dashboard).
* **Beban I/O Disk Tinggi**: Fitur *pre-aggregations* Cube.js memerlukan penyimpanan hasil agregasi ke database (Postgres/ClickHouse), menghasilkan operasi baca-tulis disk secara intensif.
* **Kompleksitas Pemeliharaan (Moving Parts)**: Banyaknya modul independen yang berjalan bersamaan (caching, queue, semantic layer, ETL) meningkatkan potensi kegagalan sistem dan membutuhkan monitoring infrastruktur yang ketat.

---

## 3. Arsitektur Konsep 2: Minimalist Setup (Lightweight)
Konsep ini memangkas komponen sekunder demi kesederhanaan dan efisiensi resource. Hanya menyisakan Apache Superset (dengan SQLite database bawaan) dan ClickHouse Core.

### Diagram Alir Data (Konsep 2)
```
[ Data Source ] ──> (Custom ETL Scripts / Python Job) ──> [ ClickHouse OLAP ]
                                                                   │
                                                                   ▼
                                                          [ Apache Superset ]
                                                           (SQLite Metadata)
```

### Detail Komponen & Container:

#### A. Container Utama (Wajib)
- `superset-app` (aplikasi Apache Superset, menggunakan SQLite `.db` lokal untuk metadata).
- `clickhouse-server` (engine database OLAP columnar).

#### B. Container Pendukung (Opsional)
- `postgres` (database untuk metadata internal Superset mengganti SQLite, mengatasi isu konkurensi).
- `redis` (caching metadata dashboard & chart).
- `celery-worker` (worker query asinkron untuk mencegah UI freeze).
- `celery-beat` (scheduler trigger laporan terjadwal otomatis).

### Kelebihan & Justifikasi:
* **Sangat Ringan**: Mengonsumsi resource CPU & RAM yang minim. Cenderung lebih stabil berjalan di lingkungan dengan spesifikasi terbatas maupun untuk kebutuhan testing.
* **Setup Cepat & Sederhana**: Arsitektur minimalis dengan konfigurasi container yang relatif sedikit dan lebih mudah dipelihara.
* **Efisiensi Biaya & Kecepatan**: Sangat cocok jika data di ClickHouse sudah matang (agregat/flat table) dan hanya memerlukan visualisasi cepat untuk sedikit pengguna.

### Risiko & Dampak Tanpa Komponen Pendukung (SQLite & Tanpa Caching):
* ⚠️ **Isu Konkurensi (Database SQLite Locked)**: SQLite mengunci database pada tingkat file saat terjadi penulisan. Jika ada lebih dari 3-5 user mengedit dashboard, mengubah dataset, atau mengonfigurasi hak akses bersamaan, Superset akan error dengan pesan `database is locked`.
* ⚠️ **UI Membeku (Freeze) Tanpa Celery**: Karena query berjalan secara synchronous, jika pengguna menjalankan query berat di ClickHouse, tab browser Superset akan menggantung (freeze) dengan animasi loading tanpa bisa dinavigasi, serta rentan terkena *Gunicorn Timeout (Error 504)*.
* ⚠️ **Beban Berulang ke ClickHouse (No Caching/Redis)**: Tanpa Redis Cache, setiap interaksi filter atau refresh halaman akan memaksa ClickHouse mengeksekusi ulang seluruh query chart dari nol. Hal ini memicu lonjakan CPU ClickHouse secara dramatis saat diakses bersamaan.
* ⚠️ **Fitur Laporan Terjadwal Mati**: Pengiriman dashboard/chart otomatis via email/Slack tidak dapat berfungsi karena membutuhkan celery-beat dan worker untuk rendering di latar belakang.

---

## 4. Matriks Perbandingan Arsitektur
Perbandingan komprehensif aspek teknis dan operasional antara Konsep 1 dan Konsep 2:

| Aspek Evaluasi | Konsep 1: Enterprise Full-Stack | Konsep 2: Minimalist Setup |
| :--- | :--- | :--- |
| **Beban Resource (RAM/CPU)** | **Rawan Melonjak**<br>Penggunaan dasar sedang, tetapi rentan kehabisan RAM/CPU saat ETL & query aktif bersamaan (terutama pada spesifikasi RAM 8GB / Intel i3). | **Ringan**<br>2 Container Docker, RAM idle ~1GB (naik ke 2-4GB saat kueri aktif, jauh lebih hemat). |
| **Metrik Bisnis & Labeling** | **Tersentralisasi (Cube.js)**<br>Satu deklarasi semantic layer untuk semua tools analitik. | **Lokal per Dataset**<br>Konfigurasi label & kalkulasi manual di dalam Superset. |
| **Stabilitas Konkurensi** | **Tinggi**<br>Metadata menggunakan PostgreSQL dan dilindungi cache Redis. | **Rendah**<br>SQLite rentan "Database Locked" jika diakses bersamaan. |
| **Kemudahan Pipeline ETL** | **Mudah (No-Code UI)**<br>Menggunakan panel GUI PeerDB untuk CDC otomatis. | **Manual**<br>Membutuhkan script ETL Python/SQL untuk memindahkan data. |
| **Penyimpanan Disk** | **Rawan Bengkak**<br>Butuh penanganan berkala terhadap file staging MinIO & Cube. | **Sangat Hemat**<br>Hanya database ClickHouse terkompresi & metadata mini. |
| **Rekomendasi Lingkungan Deployment** | Server Produksi / VM Dedicated (RAM minimal 16GB). | Device lokal / tim kecil (RAM 8GB relatif mencukupi). |

---

## 5. Perbandingan: Apache Superset vs Power BI Desktop
Berikut adalah analisis komprehensif antara **Apache Superset** dan **Microsoft Power BI Desktop** untuk kebutuhan visualisasi data analitis dari sumber data (source):

| Kriteria Evaluasi | Apache Superset | Power BI Desktop |
| :--- | :--- | :--- |
| **Akses & Platform** | Berbasis Web (Cloud-native). Dapat diakses dari sistem operasi apa pun melalui web browser. | Aplikasi Desktop khusus Windows. Pengguna macOS/Linux memerlukan VM atau Remote Desktop. |
| **Konektivitas Sumber Data** | Koneksi langsung via SQLAlchemy. Relatif mudah dikonfigurasi dan ramah terhadap query besar. | Membutuhkan driver ODBC khusus atau konektor bawaan di Windows. |
| **Metode Pengambilan Data** | Selalu melakukan query langsung (Direct Query) ke sumber data (source) secara native. | Mendukung *Import Mode* (data diimpor ke RAM lokal) atau *DirectQuery* (query langsung ke sumber data). |
| **Semantic Layer** | Bersifat tipis (ringan). Untuk logika metrik bisnis yang rumit biasanya dipasangkan dengan Cube.js atau dbt. | Sangat kaya dan matang. Menggunakan bahasa DAX dan pemodelan Tabular untuk relasi data yang kompleks secara bawaan. |
| **Beban Device Lokal** | Sangat ringan di sisi klien (device lokal hanya menjalankan peramban/browser). | Cenderung berat jika menggunakan *Import Mode* karena memproses data di RAM device lokal. |
| **Pembaruan Data & File Sharing** | Memungkinkan akses mandiri karena dataset utama sudah terpusat dan terupdate otomatis di web tanpa perlu menunggu kiriman berkas terbaru, dengan tetap mendukung unggah berkas Excel tambahan jika diperlukan. | Jika bergantung pada berkas Excel lokal, berkas perlu diperbarui berkala (baik meminta berkas baru atau menunggu email otomatis) sehingga alur kerjanya kurang praktis. |
| **Kolaborasi & Lisensi** | Open-source (Gratis). Berbagi dashboard ke pengguna lain tidak memerlukan biaya lisensi tambahan. | Pembuatan laporan lokal gratis, tetapi berbagi/kolaborasi via web service membutuhkan lisensi Pro/Premium per user. |

### Panduan Pemilihan (Kapan Memilih):
* **Cenderung Memilih Apache Superset jika**:
  - Membutuhkan platform visualisasi berbasis web terpusat agar seluruh pengguna dapat mengakses data terupdate secara mandiri tanpa perlu menunggu kiriman file berkala.
  - Ingin menghindari biaya lisensi per pengguna untuk kebutuhan kolaborasi dan berbagi dashboard.
  - Lebih menyukai analisis data berbasis kueri SQL langsung ke sumber data.
  - Memiliki infrastruktur server/Docker terpisah agar pemrosesan data tidak membebani device lokal.
* **Cenderung Memilih Power BI Desktop jika**:
  - Tim sudah terintegrasi dalam ekosistem Microsoft dan pengguna lebih terbiasa menggunakan Excel atau formula DAX.
  - Membutuhkan pemodelan data lokal yang kompleks dan relasi antar tabel yang rumit sebelum laporan dibagikan.
  - Memiliki spesifikasi device lokal yang memadai jika ingin menggunakan *Import Mode* untuk data berukuran besar guna meminimalkan risiko lambat.

---

## 6. Rekomendasi & Solusi Jalan Tengah (Rekomendasi Kenyataan)
Untuk mendapatkan arsitektur yang **ringan** tetapi tetap **stabil, aman, dan fungsional**, disarankan untuk menerapkan perbaikan kritis pada arsitektur Konsep 2 sebagai berikut:

```
                  ┌──────────────────────────────────────────────┐
                  │          [ PostgreSQL Metadata DB ]          │
                  │  (Mengatasi Isu SQLite "Database Locked")     │
                  └──────────────────────┬───────────────────────┘
                                         │
                                         ▼
┌───────────────────────┐      ┌──────────────────┐      ┌────────────────────────┐
│  [ ClickHouse Core ]  ├─────>│[ Dataset Layer ] ├─────>│   [ Apache Superset ]  │
│  - ReplacingMergeTree │      │- Kolom Kalkulasi │      │  - Visualisasi Cepat   │
│  - Partisi Tabel      │      │- Labeling Rapi   │      │  - Siap Scale-Up Cache │
└───────────────────────┘      └──────────────────┘      └────────────────────────┘
```

### A. Migrasi Metadata Superset ke PostgreSQL (Menggantikan SQLite)
* **Solusi**: Mengganti SQLite database bawaan Superset dengan PostgreSQL (cukup menambah 1 container database kecil atau mengarah ke database Postgres yang ada).
* **Manfaat**:
  - **Mengurangi Risiko Database Locked**: PostgreSQL menangani konkurensi tinggi dengan lebih baik. Banyak user dapat mengedit dashboard secara bersamaan dengan risiko hambatan konkurensi yang minim.
  - **Keamanan Data**: Menggunakan mekanisme WAL (Write-Ahead Logging) yang membantu meminimalkan risiko kerusakan data metadata (korup) apabila terjadi crash sistem mendadak.
  - **Siap Scale-Up**: Postgres dapat menjadi basis fondasi jika di kemudian hari ingin mengaktifkan caching (Redis) dan query asinkron (Celery).

### B. Optimalisasi Labeling & Metadata di Dataset Superset (Pengganti Cube.js)
Karena Cube.js dibuang untuk menghemat resource, standarisasi data dipindahkan langsung ke dalam pengaturan Dataset di Superset:
* **Labeling Kolom (Human-Readable)**: Ubah nama kolom teknis database yang kaku (misal: `dtl_tr_id` atau `hdr_cust_name`) menjadi nama yang ramah pengguna (misal: `"ID Detail Transaksi"` atau `"Nama Pelanggan"`). Pengguna tinggal melakukan drag-and-drop kolom yang sudah rapi ini.
* **Penguncian Format Tanggal (Datetime Format)**: Kunci format tanggal di tingkat dataset (misal: `YYYY-MM-DD HH:mm:ss`) agar pengguna tidak perlu memformat tanggal secara manual setiap kali membuat chart baru.
* **Calculated Columns (Kolom Kalkulasi)**: Tulis rumus kalkulasi bisnis sederhana menggunakan sintaks SQL ClickHouse (misal: `harga * jumlah * (1 - diskon)`) sekali saja di tingkat dataset. Kolom ini otomatis muncul dan siap pakai oleh siapapun.

### C. Manajemen Data ClickHouse (Memitigasi Karakteristik Columnar)
* **Operasi DML (Update/Delete) via ReplacingMergeTree**: ClickHouse adalah columnar database yang didesain untuk operasi *insert* cepat dan bersifat *append-only*. Jika data OLTP (Postgres) sering mengalami perubahan status (update/delete), gunakan engine tabel **`ReplacingMergeTree`** di ClickHouse. Engine ini akan menyaring data lama dan menyisakan data dengan versi terbaru secara berkala di latar belakang.
* **Partisi Tabel**: Lakukan partisi tabel ClickHouse secara berkala (misal berdasarkan bulan atau tahun). Partisi yang tepat membantu meminimalkan risiko terjadinya *full scan* data saat rendering dashboard, sehingga query berpotensi berjalan lebih cepat dan menekan risiko timeout.
