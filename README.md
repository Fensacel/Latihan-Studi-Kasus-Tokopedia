# Latihan Studi Kasus: Data Cleaning sampai EDA — Tokopedia Listings

Analisis end-to-end dataset listing Tokopedia: dari data mentah, pembersihan, eksplorasi
(EDA), sampai insight dan rekomendasi bisnis. Dikerjakan sebagai latihan studi kasus
mata kuliah, dengan skenario menjadi **Data Analyst di tim strategi seller** yang diminta
menjawab: *kategori dan rentang harga mana yang paling menjanjikan, apa yang membedakan
produk laris dari yang tidak, dan apakah lokasi seller berpengaruh?*

---

## Hasil utama

| Pertanyaan | Temuan |
|---|---|
| Kategori paling menjanjikan | **Skincare** — penjualan per listing 8.865 (tertinggi) dengan hanya 1.454 listing |
| Kategori jenuh | **Elektronik** — 2.285 listing tapi penjualan per listing hanya 1.017 |
| Rentang harga paling laku | **Segmen murah** (median terjual tertinggi di 7 dari 8 kategori) |
| Pembedanya | **Jumlah ulasan** (r = 0,51), bukan rating (r = -0,02) |
| Pengaruh lokasi | **Ya** — median terjual seller Jawa 60 vs luar Jawa 25 |

---

## Dataset

| | |
|---|---|
| Berkas | `produk_tokopedia.csv` |
| Jumlah baris | 29.519 listing |
| Jumlah kolom | 9 |
| Grain | **1 baris = 1 listing produk**, bukan data transaksi |
| Sumber | Indonesia E-Commerce Dataset: Tokopedia Listings (Kaggle) |

Kolom: `Nama Produk`, `Nama Toko`, `Lokasi Toko`, `Terjual`, `Jumlah Ulasan`, `Rating`,
`Harga (IDR)`, `Diskon (%)`, `Produk URL`.

> Karena grain-nya listing (bukan transaksi), tidak ada analisis per pesanan, review
> individual, atau waktu pengiriman.

---

## Struktur repositori

```
.
├── tokopedia.ipynb                       # notebook utama (71 sel, sudah dijalankan)
├── produk_tokopedia.csv                         # dataset mentah
└── README.md
```

---

## Cara menjalankan

### 1. Prasyarat

Python 3.12+ dengan paket:

```bash
pip install pandas numpy matplotlib seaborn nbformat nbclient nbconvert pypdf
```

Dikembangkan dan diuji dengan Python 3.14.7, pandas 3.0.5, numpy 2.5.3, matplotlib 3.11.2,
seaborn 0.13.2. (Batasan 3.12+ berasal dari numpy 2.5.3.)

### 2. Menjalankan notebook

```bash
jupyter notebook tokopedia_simple.ipynb
```

Lalu **Kernel → Restart & Run All**. Notebook membaca `produk_tokopedia.csv` dari
direktori yang sama, jadi jalankan dari dalam folder ini.

### 3. Verifikasi angka (opsional)

Script ini menjalankan ulang seluruh pipeline cleaning langsung dari CSV mentah, tanpa
memakai notebook, lalu mencetak setiap angka yang dipakai di analisis:

```bash
python verifikasi_angka.py
```

### 4. Ekspor ke PDF (opsional)

```bash
python ipynb2pdf.py tokopedia_simple.ipynb
```

Menghasilkan `tokopedia_simple.pdf` (A4, CSS print sudah disesuaikan). Jalankan **setelah**
notebook di-Run All, karena yang dicetak adalah output tersimpan.
Tidak butuh LaTeX — render memakai Chrome/Edge headless.

---

## Alur analisis

### Bagian 1 — Data Understanding
Memuat data, menyusun data dictionary (sebagai dictionary Python), menentukan grain data
dan kandidat ID produk, lalu menurunkan 3 pertanyaan bisnis.

**Temuan grain:** tidak ada kolom yang 100% unik. `Produk URL` paling mendekati
(28.738 unik dari 29.519 baris, 781 duplikat), sementara `Nama Produk` punya 2.634 duplikat.

### Bagian 2 — Data Cleaning

| Langkah | Tindakan | Hasil |
|---|---|---|
| 2.1 Missing values | Diberi label wajar/bermasalah per kolom | `Jumlah Ulasan` 4,27% dan `Terjual` 0,78% kosong → wajar, dibiarkan `NaN` (bukan diisi 0) |
| 2.2 Duplikat | Baris duplikat penuh dibuang | 416 baris dibuang → 29.103 |
| 2.3 Harga | Parser `parse_harga()`: buang `Rp`, pemisah ribuan, ambil batas bawah pada rentang | 145 baris ≤ Rp1.000 dibuang → 28.958 |
| 2.4 Angka terjual | `parse_jumlah()`: `rb`/`k` ×1.000, `jt` ×1.000.000, `+` dibuang | `NaN` untuk teks tak terbaca (369 baris), bukan dipaksa 0 |
| 2.5 Rating | `Rating = 0` diubah jadi `NaN` | 1.924 baris; rating valid 100% di rentang 0–5 |
| 2.6 Lokasi & kategori | Standarisasi teks + `kab.` dipertahankan sebagai entitas berbeda | Lokasi unik 322 → 251; terbentuk 8 kategori |
| 2.7 Outlier | Anomali dibuang, outlier harga di-flag | 1.055 baris anomali dibuang → **27.693 baris akhir** |

**Catatan penting 2.3:** kolom `Harga (IDR)` di dataset ini sebenarnya sudah bertipe
`int64` (tidak ada simbol `Rp` maupun pemisah ribuan). Parser tetap ditulis lengkap agar
langkah yang diminta soal benar-benar dijalankan dan kode tahan bila kolomnya berupa
string. Hasil parser diverifikasi sama persis dengan nilai asli.

**Aturan outlier harga (IQR):**

```
Q1 = Rp36.300        Q3 = Rp312.099        IQR = Rp275.799
Batas atas = Q3 + 1,5 × IQR = Rp725.797,5
→ 4.162 baris di atas batas: DIBERI FLAG, tidak dibuang
  (produk mahal bisa sah; analisis memakai median agar tidak terpengaruh)
```

**Anomali yang dibuang:** `rating ≥ 4,0` tetapi `terjual = 0` dan ulasan kosong/0
(1.055 baris) — kombinasi ini tidak logis karena rating muncul dari pembelian.
**Stok negatif tidak bisa dicek** karena dataset tidak punya kolom stok.

**Standarisasi lokasi:**
- Awalan `kota`, `kota administrasi`, `administrasi` dibuang → `kota bogor` dan `bogor` menyatu
- Awalan `kab.` **dipertahankan** → `kab. bogor` ≠ `bogor` (dua entitas administratif berbeda)
- Nilai non-kota (`indonesia`, `tokopedia`, `nasional`) → `lainnya`

**Kategori** tidak tersedia di dataset, diturunkan dari kata kunci `Nama Produk`.
7 kategori terbentuk (Skincare, Elektronik, Pakaian, Makanan, Rumah & Dapur, Olahraga,
Bayi & Anak) dan sisanya masuk `Lainnya`.

### Bagian 3 — Exploratory Data Analysis

**Univariate** — statistik deskriptif + histogram & boxplot untuk harga, rating, terjual.
Harga sangat miring (median Rp99.000 vs rata-rata Rp1.029.723, maksimum Rp900 juta),
sehingga histogram memakai `log10` dan boxplot dipotong p99 **hanya untuk keterbacaan**
(data asli tidak diubah).

Distribusi rating menumpuk di 4,5–5,0: **97,4% dari 25.733 listing yang punya rating**.

**Bivariate dan Multivariate**

| No | Analisis | Hasil |
|---|---|---|
| 1 | Top kategori per listing / terjual / revenue | Urutan **tidak sama** untuk ketiga metrik. Elektronik jenuh (2.285 listing, 1.017 per listing); Skincare paling produktif (1.454 listing, 8.865 per listing) |
| 2 | Terjual antar segmen harga per kategori | Segmen **murah** menang di 7 dari 8 kategori. Skincare satu-satunya pengecualian: menengah (250) > murah (100) > premium (80) |
| 3 | Rating tinggi vs penjualan tinggi? | **Tidak.** Median memang 60 vs 30, tapi 97,4% listing sudah di 4,5–5,0 sehingga pembandingnya lemah dan korelasinya -0,02 |
| 4 | Heatmap korelasi | Terkuat: `terjual` ↔ `ulasan` **r = 0,51** (sedang). `harga` ↔ `terjual` -0,01, `rating` ↔ `terjual` -0,02 |

Segmen harga dibuat dengan `pd.qcut(harga, 3)` → murah (≤ Rp52.250),
menengah (≤ Rp198.766,67), premium (di atasnya).

**Geografis**

| No | Analisis | Hasil |
|---|---|---|
| 1 | Top 10 kota & konsentrasi Jabodetabek | **62,8% listing** dan **69,1% volume terjual** dari Jabodetabek. Luar Jabodetabek hanya Bandung dan Surabaya yang masuk 10 besar |
| 2 | Median harga/rating/terjual per lokasi | Seller Jawa median terjual **60**, luar Jawa **25** — median harga dan rating hampir sama, jadi pembedanya bukan harga/kualitas |
| 3 | Kategori vs kota | Skincare & Elektronik terkonsentrasi di Jakarta (~0,21 per baris kategori), Pakaian di Bandung (0,15). Hipotesis: pusat distribusi vs sentra produksi — **belum diuji**, data tidak memuat info distribusi |

### Bagian 4 — Insight dan Rekomendasi

**5 temuan utama:**
1. Kategori terbesar tidak otomatis paling laku — Elektronik banyak listing tapi jenuh
2. Segmen murah paling laku di hampir semua kategori (7 dari 8)
3. Rating tidak bisa dipakai membedakan produk (r = -0,02, menumpuk di 4,5–5,0)
4. Jumlah ulasan adalah sinyal terkuat (r = 0,51) — hubungan, bukan sebab-akibat
5. Seller menumpuk di Jabodetabek (62,8% listing, 69,1% penjualan)

**3 rekomendasi:**
1. **Fokus Skincare di segmen menengah** — penjualan per listing 8.865, satu-satunya
   kategori yang paling laris di segmen menengah (median 250), listing masih sedikit (1.454)
2. **Untuk kategori lain, main di harga murah** — median terjual segmen murah tertinggi
   di 7 dari 8 kategori (rentang 50–250, umumnya 100) vs menengah 30–80 dan premium 9–30
3. **Mulai dari seller/gudang di Jabodetabek** — median terjual `jakarta utara` 100,
   `tangerang` 90, `jakarta barat` 90, jauh di atas luar Jawa (25)

---

## Batasan

- Satu baris = satu listing, bukan transaksi. Tidak ada kolom tanggal → tidak ada analisis tren.
- `Terjual` berupa rentang (`1rb+`, `250+`) → nilainya perkiraan batas bawah, bukan angka pasti.
- Kategori diturunkan dari kata kunci; setelah cleaning **65,3% masuk `Lainnya`**, jadi temuan
  per kategori hanya berlaku untuk bagian yang terklasifikasi.
- Tidak ada kolom stok → cek "stok negatif" tidak bisa dilakukan.
- Harga ekstrem (sampai Rp900 juta) masih ada di data bersih → rata-rata dan varians
  terpengaruh, median lebih aman dibaca.
- Daftar kota Jawa dan definisi Jabodetabek disusun manual dari nama lokasi → pengelompokan
  geografis bersifat perkiraan.

---

## Catatan teknis

- **Tipe data:** `Terjual` dan `Jumlah Ulasan` dikonversi ke `Int64` (integer nullable) agar
  `NaN` tidak hilang dan tidak dipaksa menjadi 0.
- **Reproducibility:** seluruh angka pada Bagian 4 dapat dihitung ulang dengan
  `verifikasi_angka.py`, yang menjalankan pipeline cleaning secara independen dari CSV mentah.
  Semua angka sudah dicocokkan dengan output notebook.
- **Rujukan silang:** kesimpulan di Bagian 4 merujuk ke nomor soal (mis. "lihat Bivariate dan
  Multivariate poin 1") supaya mudah ditelusuri.

---

## Lisensi & atribusi

Dataset: *Indonesia E-Commerce Dataset: Tokopedia Listings* (Kaggle). Notebook dan analisis
dibuat untuk keperluan akademik.