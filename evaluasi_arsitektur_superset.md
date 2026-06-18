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

### Detail Komponen & Container (Fitur Lengkap / Full-Feature):

#### 1. Visualization & Analytics (Apache Superset BI)
- `superset-app` (Aplikasi utama Apache Superset / Web UI): Container ini melayani antarmuka pengguna berbasis web (Web UI) tempat analis membuat grafik (*charts*) dan dashboard. Ia bertindak sebagai klien penerima kueri SQL Lab dan menyajikan visualisasi data secara interaktif. Secara internal, ia berkomunikasi dengan database metadata untuk otentikasi user dan pengambilan definisi dataset.
- `superset-init` (Inisialisasi database metadata): Container sekali-jalan (*run-and-exit*) yang bertugas menyalakan skema database, memigrasikan tabel metadata internal, membuat pengguna administratif (*admin*), serta memetakan hak akses peran bawaan (*default roles*) sebelum `superset-app` mulai menerima koneksi.

#### 2. Asynchronous Task & Caching Engine (Celery & Redis)
- `redis` (Caching & Message Broker): Berperan ganda sebagai media penyimpanan cache memori (untuk menyimpan hasil kueri dashboard guna mempercepat pemuatan berulang) sekaligus sebagai *message broker* (antrean pesan) bagi Celery worker untuk mendelegasikan tugas berat di latar belakang.
- `celery-worker` (Prosesor Eksekusi Latar Belakang): Mengambil tugas kueri analitis jangka panjang yang dikirim oleh Superset melalui Redis, mengeksekusinya ke ClickHouse secara asinkron, dan menulis hasilnya kembali ke cache. Ini mencegah proses Gunicorn pada `superset-app` mengalami *timeout* atau membuat browser pengguna menggantung (*freeze*).
- `celery-beat` (Scheduler Terjadwal): Pemicu berkala (*cron-like scheduler*) yang secara otomatis mengirimkan jadwal tugas ke Redis (message broker) untuk kemudian diambil dan dieksekusi oleh Celery worker guna merender laporan terjadwal (melalui Slack atau Email).

#### 3. Semantic Layer (OLAP Cube)
- `cube-core` (OLAP Cube & Semantic Layer Server): Container ini bertindak sebagai jembatan deklaratif di atas ClickHouse. Ia mendefinisikan skema data logis (*semantic layer*), relasi antar-tabel, dan metrik bisnis terpusat. Ketika Superset (atau BI tool lain seperti Power BI) mengirimkan kueri, Cube.js menerjemahkannya ke SQL ClickHouse yang optimal dan mengelola *pre-aggregations* (tabel agregasi instan di memori/disk) guna meminimalkan latensi kueri.

#### 4. Data Pipeline & CDC Engine (PeerDB & Flow)
- `peerdb-ui` (Antarmuka Web Data Pipeline): Menyediakan dashboard visual (GUI) bagi insinyur data (*data engineer*) untuk mengonfigurasi replikasi data real-time, mendeteksi tabel sumber, serta memantau status transfer data.
- `peerdb-server` (Core CDC Engine): Otak replikasi data yang membaca log perubahan transaksi (*Write-Ahead Log / WAL*) dari database transaksional (PostgreSQL OLTP) menggunakan protokol CDC (Change Data Capture) dan menerjemahkannya menjadi instruksi pemuatan data analitis.
- `flow-worker` (Pengeksekusi Alur Replikasi): Worker asinkron yang melakukan pembacaan log transaksi secara terus-menerus dan menuliskan data perubahan tersebut ke dalam antrean target.
- `flow-snapshot` (Inisialisasi Awal Data): Container khusus untuk memigrasikan seluruh isi data historis lama secara massal (*bulk copy*) dari database transaksional pada proses inisiasi pertama kali.
- `flow-api` (Gateway API Integrasi): Menyediakan titik akhir REST API untuk memicu, menghentikan, atau mengaudit status pekerjaan replikasi data secara programmatic.

#### 5. Databases & Staging Storage
- `postgres` (Database Metadata Superset): Database relasional transaksional yang menyimpan seluruh konfigurasi internal Apache Superset seperti kredensial user, definisi dashboard, tata letak grafik, hak akses RBAC, dan metadata dataset.
- `clickhouse-server` (Engine Database OLAP Analitis): Core columnar database yang menyimpan data hasil replikasi dalam bentuk tabel-tabel pipih (*flat tables*). ClickHouse sangat efisien dalam memproses kueri agregasi analitis skala besar di tingkat kolom secara paralel.
- `minio` (Object Storage Lokal): Berperan sebagai penyimpanan perantara (*staging storage*). PeerDB mengekspor data transaksi dari Postgres OLTP ke format CSV terkompresi di MinIO terlebih dahulu sebelum ClickHouse melakukan kueri `COPY` massal, yang jauh lebih cepat dibandingkan menyisipkan baris per baris secara berulang.

### Kelebihan & Justifikasi:
* **Single Source of Truth**: Semantic layer terpusat di Cube.js membantu menjaga agar definisi metrik dan label kolom tetap konsisten meskipun diakses dari berbagai BI tools (Superset, Power BI, Tableau).
* **ETL Mudah Dikelola**: Proses pemindahan data dari OLTP ke OLAP terpantau penuh melalui antarmuka web GUI PeerDB.
* **Performa Tinggi**: Query analitis dapat dieksekusi dengan latensi rendah berkat optimasi columnar database ClickHouse dan pre-aggregations Cube.js.
* **Solusi CDC & Volume Data Masif**: Sangat direkomendasikan jika terdapat kebutuhan **CDC (Change Data Capture) semi-instan** guna menyinkronkan data transaksional bervolume masif dari database OLTP ke database OLAP ClickHouse secara cepat tanpa membebani performa database transaksional utama.
* **Solusi Comprehensive Terintegrasi**: Menjadi pilihan terbaik jika organisasi **belum memiliki infrastruktur BI, ETL pipeline, maupun sistem CDC** yang sudah terpasang (*inplace*) sebelumnya, sehingga memerlukan solusi arsitektur lengkap yang siap pakai dan saling terintegrasi dari pengumpulan hingga visualisasi data.

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

#### A. Container Utama (Wajib / Core Minimalist)
* **Visualization:** `superset-app` (Aplikasi Utama Apache Superset): Container server visualisasi yang dikonfigurasi menggunakan SQLite database (`.db` berbasis file lokal) untuk menyimpan seluruh metadatanya. Semua proses rendering chart dan eksekusi kueri ke ClickHouse ditangani langsung secara sinkron (*synchronous*) oleh web server Gunicorn di dalam container ini.
* **Database OLAP:** `clickhouse-server` (Engine Database Columnar): Menyimpan data terstruktur untuk kueri analitis cepat. Superset terhubung langsung ke port SQL ClickHouse untuk menarik data visualisasi secara real-time tanpa perantara semantic layer.

#### B. Container Pendukung (Opsional - Alur Scale-Up Bertahap)
* **Database Metadata:** `postgres` (Pengganti SQLite): Container database relasional untuk menyimpan konfigurasi metadata Superset secara terpusat guna menghindari error penguncian berkas (*file locking*) saat diakses beberapa pengguna secara bersamaan.
* **Caching & Message Broker:** `redis` (Cache & Broker): Menyimpan hasil kueri visualisasi sebelumnya secara temporal untuk memangkas kueri berulang ke ClickHouse serta mengatur antrean task.
* **Asynchronous Workers:** `celery-worker` & `celery-beat` (Pemroses Latar Belakang & Scheduler): Bertanggung jawab memproses kueri SQL Lab yang membutuhkan waktu lama agar tidak memblokir antarmuka pengguna serta menangani pengiriman laporan terjadwal.

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
| **Rekomendasi Lingkungan Deployment** | **Server Linux Produksi / Kubernetes (K8s)**<br>(Direkomendasikan dengan RAM dedicated ≥16GB untuk stabilitas CDC & BI terpadu). | **Fase PoC / Uji Batas Lokal / VM Tim Kecil**<br>(Cocok untuk uji batas ketahanan pada resource terbatas seperti RAM 8GB). |

---

## 5. Estimasi Kebutuhan Resource per Container

> **Catatan**: Estimasi berikut bersumber dari dokumentasi resmi dan diskusi komunitas (GitHub Issues, forum ClickHouse, Apache Superset). Nilai aktual dapat berbeda tergantung volume data, kompleksitas kueri, dan jumlah pengguna aktif. Angka-angka ini bersifat indikatif sebagai acuan perencanaan kapasitas — untuk mendapatkan angka yang lebih akurat sesuai kondisi spesifik, diperlukan pengujian lebih lanjut (misalnya monitoring resource saat beban nyata).

### Konsep 1 – Enterprise Full-Stack (14 Container)

| Container | Kategori Beban | Estimasi RAM Idle | Kondisi Saat Aktif |
| :--- | :--- | :--- | :--- |
| `superset-app` | Sedang | ~300–500 MB | Naik hingga ~800 MB–1.5 GB saat banyak user aktif bersamaan |
| `superset-init` | Sangat Ringan | ~50–100 MB | Hanya berjalan sekali saat inisialisasi, lalu berhenti otomatis |
| `redis` | Sangat Ringan | ~30–50 MB | Naik sesuai jumlah hasil cache chart yang tersimpan |
| `celery-worker` | Ringan–Sedang | ~200–400 MB | Naik saat memproses kueri berat atau rendering laporan terjadwal |
| `celery-beat` | Sangat Ringan | ~50–100 MB | Hampir konstan, hanya mengirimkan trigger jadwal |
| `cube-core` | Sedang | ~300–600 MB | Naik saat menghitung *pre-aggregations* baru ke database |
| `peerdb-ui` | Ringan | ~100–200 MB | Relatif konstan sebagai UI monitoring pipeline |
| `peerdb-server` | Sedang | ~200–400 MB | Naik saat CDC aktif mereplikasi perubahan data dari OLTP |
| `flow-worker` | Sedang | ~200–500 MB | Naik signifikan saat proses sinkronisasi data berjalan |
| `flow-snapshot` | Berat (Sesaat) | ~500 MB–1 GB | Aktif saat dump awal data historis, lalu berhenti otomatis |
| `flow-api` | Ringan | ~100–150 MB | Relatif konstan sebagai gateway REST API |
| `postgres` | Sangat Ringan | ~50–100 MB | Hanya menyimpan metadata Superset, tidak bertumbuh besar |
| `clickhouse-server` | Sedang–Tinggi | ~500 MB–1 GB | Naik hingga 2–4 GB saat kueri analitis besar aktif dijalankan |
| `minio` | Ringan | ~100–200 MB | Naik saat ada file CSV *staging* yang sedang ditulis PeerDB |

**Estimasi Total RAM Idle Konsep 1:** ~2.5–4.5 GB (sebelum ada beban query/ETL aktif)

> ⚠️ Pada spesifikasi RAM 8 GB, penggunaan idle Konsep 1 sudah mengonsumsi lebih dari separuh kapasitas RAM — menyisakan sedikit ruang untuk sistem operasi dan aktivitas lainnya. Risiko lonjakan resource (*spike*) menjadi lebih tinggi ketika ETL dan query analitis berjalan bersamaan.

---

### Konsep 2 – Minimalist Setup (2–6 Container)

| Container | Kategori Beban | Estimasi RAM Idle | Kondisi Saat Aktif |
| :--- | :--- | :--- | :--- |
| `superset-app` | Sedang | ~300–500 MB | Naik saat banyak user menjalankan query dashboard bersamaan |
| `clickhouse-server` | Sedang–Tinggi | ~500 MB–1 GB | Naik hingga 2–4 GB saat query analitis besar aktif |
| `postgres` | Sangat Ringan | ~50–100 MB | Relatif konstan, hanya metadata konfigurasi Superset |
| `redis` | Sangat Ringan | ~30–50 MB | Naik sesuai jumlah cache chart yang tersimpan |
| `celery-worker` | Ringan–Sedang | ~200–400 MB | Naik saat ada laporan terjadwal atau kueri latar belakang |
| `celery-beat` | Sangat Ringan | ~50–100 MB | Hampir konstan, hanya mengirimkan trigger jadwal |

**Estimasi Total RAM Idle Konsep 2 (Hanya Wajib):** ~800 MB–1.5 GB

**Estimasi Total RAM Idle Konsep 2 (Dengan Semua Pendukung):** ~1.2–2.1 GB

> ✅ Pada spesifikasi RAM 8 GB, Konsep 2 bahkan dengan semua container pendukung aktif cenderung masih menyisakan ruang RAM yang cukup nyaman untuk sistem operasi dan aktivitas lainnya.

---

## 6. Perbandingan: Apache Superset vs Power BI Desktop
Berikut adalah analisis komprehensif antara **Apache Superset** dan **Microsoft Power BI Desktop** untuk kebutuhan visualisasi data analitis dari sumber data (source):

| Kriteria Evaluasi | Apache Superset | Power BI Desktop |
| :--- | :--- | :--- |
| **Akses & Platform** | Berbasis Web (Cloud-native). Dapat diakses dari sistem operasi apa pun melalui web browser. | Aplikasi Desktop khusus Windows. Pengguna macOS/Linux memerlukan VM atau Remote Desktop. |
| **Konektivitas Sumber Data** | Koneksi langsung via SQLAlchemy. Relatif mudah dikonfigurasi dan ramah terhadap query besar. | Membutuhkan driver ODBC khusus atau konektor bawaan di Windows. |
| **Metode Pengambilan Data** | Mengirimkan query SQL langsung ke sumber data, dengan sebagian pemrosesan hasil (seperti sorting dan filtering) dilakukan di sisi server Superset (Python/Gunicorn) sebelum ditampilkan ke pengguna. | Mendukung *Import Mode* (data diimpor ke RAM lokal) atau *DirectQuery* (query langsung ke sumber data). |
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

## 7. Rekomendasi Final

Berdasarkan analisis arsitektur, estimasi resource, dan kondisi infrastruktur yang dievaluasi, bagian ini merangkum keputusan teknis serta pilihan arsitektur yang disarankan.

> 💡 **Catatan Penting Konten Pengujian:** Spesifikasi hardware laptop yang minim (RAM 8 GB, Intel i3) **hanya digunakan untuk menguji batas ketahanan (stress test/boundary testing)** sistem pada kondisi resource yang sangat terbatas. Konteks ini **bukanlah target implementasi produksi final**. Untuk deployment produksi di masa mendatang, sistem pasti akan dijalankan pada **server Linux menggunakan Docker CLI atau diorkestrasikan melalui Kubernetes (K8s)** dengan alokasi resource penuh.

---

### A. Mengapa Konsep 1 Tidak Diprioritaskan untuk Tahap Pengujian Batas (PoC) Ini

Konsep 1 (Enterprise Full-Stack) dirancang untuk lingkungan produksi skala besar. Selama fase pengujian PoC di lingkungan lokal yang dibatasi (RAM 8 GB, i3):

* **Tujuan Pengujian Batas:** Pengujian sengaja menggunakan hardware berspesifikasi minim untuk memetakan batasan kritis. Konsep 1 dengan 14 container (estimasi idle RAM ~2.5–4.5 GB) terbukti memicu crash akibat lonjakan resource saat proses ETL CDC (PeerDB) dan query analitis berjalan bersamaan.
* **Relevansi Produksi:** Meskipun berat untuk laptop testing, **Konsep 1 merupakan arsitektur target utama (recommended)** ketika dideploy ke infrastruktur server Linux produksi sesungguhnya menggunakan Docker CLI atau Kubernetes, di mana resource hardware dedicated tersedia secara melimpah dan isolasi container terjamin.
* **Kebutuhan CDC Semi-Instan & Data Masif:** Jika terdapat kebutuhan menyinkronkan data transaksional bervolume masif dari database OLTP ke database OLAP ClickHouse secara cepat tanpa membebani performa database transaksional utama, maka Konsep 1 dengan infrastruktur CDC (PeerDB) menjadi kebutuhan wajib.
* **Skenario Belum Memiliki Sistem BI/ETL Inplace:** Konsep 1 bertindak sebagai **solusi komprehensif (*comprehensive solution*)** yang siap pakai. Jika internal organisasi belum memiliki sistem pipeline ETL, CDC, maupun semantic layer yang terpasang (*inplace*), arsitektur lengkap ini sangat direkomendasikan karena mengintegrasikan seluruh silo data dari hulu ke hilir.
* **Kapasitas Operasional:** Arsitektur 14 container ini membutuhkan tim ops untuk monitoring pipeline CDC, pre-aggregations Cube.js, dan queue Celery secara berkelanjutan di cluster Kubernetes.

> 💡 **Konsep 1 tetap relevan sebagai target arsitektur jangka panjang** ketika tahap implementasi final dilakukan ke Linux server dengan Docker CLI / Kubernetes.

---

### B. Rekomendasi Langkah Awal: Konsep 2 yang Dioptimalkan (Solusi Hybrid untuk Fase PoC / Uji Batas)

Untuk kelancaran fase pengujian PoC di device dengan resource terbatas ini, disarankan menggunakan **Konsep 2 dengan penambahan komponen kritis**. Pendekatan ini menjaga sistem tetap ringan namun menghindari kelemahan fatal SQLite.

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

#### B.1 Migrasi Metadata Superset ke PostgreSQL (Menggantikan SQLite)
* **Solusi**: Mengganti SQLite database bawaan Superset dengan PostgreSQL (cukup menambah 1 container database kecil atau mengarah ke database Postgres yang ada).
* **Manfaat**:
  - **Mengurangi Risiko Database Locked**: PostgreSQL menangani konkurensi tinggi dengan lebih baik. Banyak user dapat mengedit dashboard secara bersamaan dengan risiko hambatan konkurensi yang minim.
  - **Keamanan Data**: Menggunakan mekanisme WAL (Write-Ahead Logging) yang membantu meminimalkan risiko kerusakan data metadata (korup) apabila terjadi crash sistem mendadak.
  - **Kemudahan Migrasi ke Produksi**: Basis database metadata PostgreSQL ini akan mempermudah tim saat memindahkan seluruh konfigurasi dasbor ke server produksi Linux (baik via Docker Compose maupun Kubernetes manifests).

#### B.2 Optimasi Dataset Superset (Konsep 2 menggunakan SQLite Local DB)
Tanpa Cube.js, standarisasi data dipindahkan langsung ke dalam pengaturan Dataset di Superset:
* **Labeling Kolom (Human-Readable)**: Ubah nama kolom teknis database yang kaku (misal: `dtl_tr_id` atau `hdr_cust_name`) menjadi nama yang ramah pengguna (misal: `"ID Detail Transaksi"` atau `"Nama Pelanggan"`). Pengguna tinggal melakukan drag-and-drop kolom yang sudah rapi ini.
* **Penguncian Format Tanggal (Datetime Format)**: Kunci format tanggal di tingkat dataset (misal: `YYYY-MM-DD HH:mm:ss`) agar pengguna tidak perlu memformat tanggal secara manual setiap kali membuat chart baru.
* **Calculated Columns (Kolom Kalkulasi)**: Tulis rumus kalkulasi bisnis sederhana menggunakan sintaks SQL ClickHouse (misal: `harga * jumlah * (1 - diskon)`) sekali saja di tingkat dataset. Kolom ini otomatis muncul dan siap pakai oleh siapapun.
* **Keterbatasan Konversi Label**: Konsep 2 menggunakan `sqlite3` Local DB untuk menyimpan data tabel SQL metadata Superset. Tidak bisa mengonversi label yang sudah ada di Superset secara langsung ke SQLite. Label harus dikonfigurasi ulang dengan mengubah kolom database analitis menjadi string ID unik.

#### B.3 Manajemen Siklus Data di ClickHouse (DML & Partisi)
* **Mitigasi Performanya**:
  - **ReplacingMergeTree Engine**: Gunakan engine tabel ini untuk mengatasi data transaksional yang sering berubah status (misal: `UPDATE` status order_items menjadi `REPLACE`).
  - **Partisi Tabel**: Selalu partisi data tabel ClickHouse (misal per bulan/tahun).
  - **Catatan**: Partisi yang tepat membantu mengurangi kemungkinan terjadinya kueri memindai seluruh tabel data (full-table scan) saat filter dashboard diubah, sehingga membantu menghemat penggunaan CPU ClickHouse.

* **Catatan Teknis Penting**:
  - **Konsep 2 Menggunakan SQLite Local DB**: Konversi label ke SQLite **tidak** dilakukan di konsep ini. Label harus dikonfigurasi ulang dengan mengubah kolom database analitis menjadi string ID unik (contoh: `dtl_tr_id` -> `'1000'`).
  - **Keterbatasan**: Hanya valid untuk testing cepat dan minimalisasi resource.
  - **Saran Production**: Untuk deployment production yang memerlukan standardisasi schema, disarankan menggunakan konsep 1 atau menerapkan strategi "Database Migration" (lihat B.1) sebelum mengonversi ke SQLite.
