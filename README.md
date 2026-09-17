<h1 align="center">INTERN-RPJMN</h1>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.12-blue" alt="Python">
  <img src="https://img.shields.io/badge/indikator-292-green" alt="Indikator">
  <img src="https://img.shields.io/badge/model-2-orange" alt="Model">
  <img src="https://img.shields.io/badge/status-inferensi%20selesai-success" alt="Status">
</p>

> Pipeline verifikasi status realisasi indikator program prioritas RPJMN 2025-2029 menggunakan LLM dengan pencarian web, dievaluasi terhadap gold standard hasil verifikasi manual.

Setiap indikator diverifikasi secara otomatis: model menyusun kueri pencarian, menilai otoritas sumber yang ditemukan, lalu memutuskan apakah program indikator tersebut berjalan pada Tahun 2025 beserta bukti publikasi resminya. Repositori ini berisi kode inferensi, data mentah, dan hasil yang siap dianalisis.

---

## Table of Contents

- [Untuk Tim Analisis](#untuk-tim-analisis)
- [Arsitektur](#arsitektur)
- [Struktur Repositori](#struktur-repositori)
- [Model & Konfigurasi](#model--konfigurasi)
- [File Output](#file-output)
- [Kamus Kolom](#kamus-kolom)
- [Keterbatasan](#keterbatasan)
- [Saran Analisis](#saran-analisis)

---

## Untuk Tim Analisis

Kalau kamu baru bergabung dan hanya butuh datanya, tiga file ini yang penting:

| File | Isi | Kapan dipakai |
| --- | --- | --- |
| `data/output/detail_<model>_serper_<timestamp>.csv` | Satu baris per indikator, kolom siap pakai | Analisis utama |
| `data/output/raw_<model>_serper.jsonl` | Data mentah: seluruh hasil pencarian per tool call + respons model sebelum parsing | Analisis kesalahan |
| `data/raw/gold_standard_sample.csv` | Gold standard hasil verifikasi manual | Acuan pembanding |

**Baca [Keterbatasan](#keterbatasan) sebelum menghitung apa pun.** Distribusi kelasnya sangat timpang (286 TRUE : 6 FALSE), sehingga akurasi agregat hampir tidak bermakna tanpa konteks.

**Jangan buang JSONL-nya.** CSV hanya memuat keputusan akhir. Untuk membedakan "pencarian tidak menemukan dokumennya" dari "dokumennya ada tapi model salah menilai" — dua masalah dengan solusi yang sama sekali berbeda — kamu perlu melihat cuplikan hasil pencarian yang tersimpan di JSONL.

---

## Arsitektur

Model-model di ModelHub **tidak memiliki kemampuan web search bawaan**. Hal ini sudah diverifikasi terhadap seluruh model chat yang tersedia: parameter `tools: [{"type": "web_search"}]` ditolak dengan HTTP 400, dan `web_search_options` diterima tetapi diabaikan tanpa efek. Backend-nya vLLM, yang murni melayani inference dan tidak berurusan dengan jaringan.

Yang berfungsi adalah **function calling**. Karena itu pencarian dijalankan oleh kode di notebook, bukan oleh model:

```
model  →  "carikan: laporan kinerja 2025 Kemenkes STBM"
             ↓
       cari_web()  ← fungsi Python di notebook
             ↓
       Serper API (indeks Google)
             ↓
       4 hasil (judul + URL + cuplikan 250 karakter)
             ↓
model  ←  hasil dikirim balik, model membaca dan menilai
             ↓
       JSON: {realisasi, implementation_summary, source_url, confidence, ...}
```

Pembagian kerjanya: search engine sebagai mata, model sebagai penilai. Model tetap yang menentukan kueri, menilai otoritas sumber, menyaring tahun, dan memutuskan label.

---

## Struktur Repositori

```
1. llm/
├── .env                               # MODELHUB_LLM_API_KEY, SERPER_API_KEY, TAVILY_API_KEY
├── data/
│   ├── raw/
│   │   └── gold_standard_sample.csv   # 292 indikator + label gold
│   ├── cache/                         # cache hasil pencarian, dibagi antar model
│   │   ├── search_cache_serper.json
│   │   ├── search_cache_tavily.json
│   │   └── search_cache_ddg.json
│   └── output/
│       ├── detail_<model>_serper_<ts>.csv    # HASIL UTAMA
│       ├── raw_<model>_serper.jsonl          # data mentah + checkpoint
│       ├── smoke_detail_<ts>.csv             # hasil smoke test 10 baris
│       ├── smoke_summary_<ts>.csv
│       ├── model_test_results_10rows.csv     # eksplorasi awal
│       └── model_test_summary_10rows.csv
└── notebook/
    ├── 1_test_modelhub_llm.ipynb             # cek konektivitas & daftar model
    ├── 1b_test_modelhub_llm_generate.ipynb   # smoke test kehidupan model
    ├── 2_rpjmn_smoke_test.ipynb              # smoke test pertama (DuckDuckGo)
    ├── 3_rpjmn_smoke_test_tavily_serper.ipynb# smoke test Tavily vs Serper
    └── 4_rpjmn_production_run.ipynb          # RUN PRODUKSI 292 baris
```

Notebook bernomor menunjukkan urutan pengerjaan. Notebook 1-3 adalah proses eksplorasi dan penetapan konfigurasi; **notebook 4 yang menghasilkan data final.**

Cache pencarian dibagi antar model. Kueri identik dari model berbeda hanya dibayar sekali, sehingga run model kedua jauh lebih hemat kredit sekaligus memastikan kedua model menerima hasil pencarian yang sama untuk kueri yang sama.

---

## Model & Konfigurasi

Dua model dibandingkan dengan desain berpasangan: keduanya mengerjakan 292 item yang identik.

| Model | Peran | Karakter dari smoke test |
| --- | --- | --- |
| `watsonx-qwen3-30b-a3b-instruct-2507` | Model utama | MoE 30B. Cenderung skeptis: lebih sering menjawab FALSE, `recall_FALSE` tinggi tetapi `recall_TRUE` rendah. ~7.300 token/baris |
| `gemma-4-26B-A4B-it` | Pembanding | Lebih cepat dan hemat. Cenderung percaya: `recall_TRUE` sempurna tetapi lebih sering false positive. ~4.400 token/baris |

Pasangan ini dipilih justru karena **profil kesalahannya berlawanan**. Model yang sama-sama bias ke arah yang sama tidak menghasilkan analisis yang informatif.

### Parameter run

| Parameter | Nilai | Alasan |
| --- | --- | --- |
| Search backend | Serper (indeks Google) | Unggul konsisten atas Tavily di smoke test pada kedua model |
| Hasil per pencarian | 4 | Hemat token konteks |
| Panjang cuplikan | 250 karakter | Hemat token konteks |
| Tool result dipotong | 3.000 karakter | Hemat token konteks |
| `temperature` | **0.0** | Deterministik. Dengan cache pencarian, run ulang menghasilkan output identik |
| `max_tokens` | 800 per panggilan | Hemat token |
| Maks pencarian per baris | 3 (batas di prompt) | Hemat kredit search |
| Maks ronde tool-calling | 6 | Pengaman agar loop tidak menggantung |
---

## File Output

| File | Format | Isi |
| --- | --- | --- |
| `detail_<model>_serper_<ts>.csv` | CSV (`utf-8-sig`) | 292 baris, 35 kolom. Siap dibuka di Excel maupun pandas |
| `raw_<model>_serper.jsonl` | JSON Lines | Satu objek per indikator. Memuat semua kolom CSV **plus** `tool_calls_log` (seluruh hasil pencarian per kueri) dan `raw_response` (teks model sebelum parsing) |

JSONL juga berfungsi sebagai checkpoint: menjalankan ulang notebook 4 akan melewati baris yang sudah selesai, sehingga run yang putus di tengah dapat dilanjutkan.

---

## Kamus Kolom

### Identitas & acuan

| Kolom | Tipe | Keterangan |
| --- | --- | --- |
| `run_id`, `timestamp` | str | Penanda waktu run |
| `model`, `backend`, `temperature` | str/float | Konfigurasi yang dipakai |
| `row_no`, `id` | int/str | Nomor dan ID indikator, kunci join antar model |
| `sektor`, `kl` | str | Sektor dan K/L pengampu |
| `sub_indikator`, `target_2025`, `satuan` | str | Isi indikator |
| `gold` | bool | Label acuan dari verifikasi manual |
| `summary_gold` | str | Ringkasan bukti versi manual |
| `gold_suspect` | bool | **Label gold diragukan, perlu adjudikasi.** Lihat Keterbatasan poin 3 |

### Hasil model

| Kolom | Tipe | Keterangan |
| --- | --- | --- |
| `pred` | bool | Label dari model |
| `benar` | bool | `pred == gold` |
| `confidence` | int 0-100 | Keyakinan yang dilaporkan model. **Untuk analisis ambang / ROC** |
| `source_type` | str | `lkj` \| `siaran_pers` \| `bappenas` \| `bps_setkab` \| `media` \| `tidak_ada` |
| `evidence_year` | str | `2025` \| `2026` \| `lain` \| `tidak_ada` |
| `summary_pred` | str | Ringkasan bukti versi model |
| `source_url_pred` | str | URL bukti utama |

### Integritas & diagnostik

| Kolom | Tipe | Keterangan |
| --- | --- | --- |
| `url_dari_search` | bool | Apakah `source_url_pred` benar berasal dari hasil pencarian yang diterima model. **`False` = URL karangan** |
| `domain`, `domain_goid` | str/bool | Domain sumber dan apakah berakhiran `.go.id` |
| `pakai_template` | bool | Apakah ringkasan mengikuti format "Program berjalan/belum berjalan, ditandai dengan ..." |
| `n_search`, `n_search_gagal` | int | Jumlah pencarian dan yang error/kosong. **`n_search_gagal` tinggi = kegagalan retrieval, bukan kesalahan penalaran** |
| `n_query`, `queries_str` | int/str | Kueri yang disusun model |
| `status`, `error` | str | `OK` \| `PARSE_FAIL` \| `MAX_ROUNDS` \| `FAIL` |
| `in_tok`, `out_tok`, `latency_s` | int/float | Biaya dan waktu |

### Hanya di JSONL

| Field | Keterangan |
| --- | --- |
| `tool_calls_log` | List berisi `{query, situs, hasil}`. `hasil` memuat judul, URL, dan cuplikan setiap hasil pencarian yang diterima model |
| `raw_response` | Teks mentah model sebelum parsing JSON. Berguna bila parsing perlu diperbaiki di kemudian hari tanpa akses API |

---

## Keterbatasan

Lima hal berikut wajib disebutkan saat menafsirkan atau melaporkan angka.

**1. Distribusi kelas sangat timpang: 286 TRUE : 6 FALSE (97,9% TRUE).**
Menebak TRUE untuk seluruh baris sudah menghasilkan akurasi 97,9%. Akurasi agregat karena itu hampir tidak bermakna — laporkan metrik per kelas. `recall_FALSE` dihitung dari hanya 6 baris, sehingga confidence interval-nya sangat lebar dan tidak dapat dijadikan dasar kesimpulan kuat.

Implikasi strategis: nilai sesungguhnya dari sistem ini kemungkinan bukan pada akurasi klasifikasi, melainkan pada **kualitas penemuan bukti** — apakah `source_url_pred` mengarah ke dokumen resmi yang relevan, dan apakah `summary_pred` dapat dipertanggungjawabkan.

**2. Tidak ada ulangan (repeat).**
Setiap baris dijalankan satu kali karena keterbatasan akses API. Variansi antar-run tidak dapat diestimasi dari data ini. Mitigasi: `temperature=0` dan hasil pencarian di-cache, sehingga run ulang dengan cache yang sama bersifat deterministik.

**3. Gold standard belum diadjudikasi.**
Baris dengan `gold_suspect=True` ditemukan bertentangan dengan bukti resmi pada saat smoke test. Kasus yang tercatat: **no=11 (Produksi vanili)** dilabeli FALSE dengan alasan tidak ada bukti publikasi resmi, namun empat run terpisah (dua model x dua backend) menemukan `LAKIN-DITJENBUN-2025.pdf` yang melaporkan produksi vanili 2025 sebesar 1.286 ton atau 81,29% dari target 1.582 ton.

Disarankan melakukan **adjudikasi buta** — acak urutan baris yang gold dan pred berbeda, periksa ulang tanpa melihat label lama — sebelum menyimpulkan akurasi. Gold yang keliru menyebabkan jawaban benar dihitung salah, dan dengan hanya 6 baris FALSE, satu kekeliruan sudah cukup menggeser kesimpulan.

**4. Bukti terbatas pada cuplikan hasil pencarian.**
Sistem tidak membuka isi PDF. Angka capaian yang hanya tercantum di dalam dokumen tidak terlihat oleh model. Ini menjelaskan sebagian kasus di mana model menyatakan tidak menemukan capaian padahal dokumennya ditemukan.

**5. Kualitas retrieval ikut menentukan hasil.**
Baris dengan `n_search_gagal > 0` atau `source_type = tidak_ada` kemungkinan gagal karena pencarian, bukan karena penalaran model. Pola yang teramati: model kadang menebak domain K/L dan salah (misalnya `kemenkes.go.id` padahal yang benar `kemkes.go.id`), sehingga pencarian ber-`site:` mengembalikan nol hasil. Bedakan kedua jenis kegagalan ini saat menyusun taksonomi kesalahan.

**Hasil pencarian adalah snapshot pada tanggal run.** Menjalankan ulang di kemudian hari dapat memberi hasil berbeda karena isi web berubah.

---

## Saran Analisis

| Analisis | Metode | Kolom yang dipakai |
| --- | --- | --- |
| Perbandingan dua model | **Uji McNemar** (desain berpasangan, item identik) | `benar` per model, di-join pada `row_no` |
| Interval selisih akurasi | Bootstrap CI dengan resampling atas item | `benar` |
| Analisis ambang | Kurva ROC dan precision-recall; cari titik potong operasional | `confidence`, `gold` |
| Kalibrasi | Reliability diagram, Brier score | `confidence`, `benar` |
| Kesepakatan antar model | Cohen's kappa; bandingkan akurasi saat sepakat vs berbeda | `pred` kedua model |
| Ensemble | Aturan "TRUE hanya bila kedua model setuju" | `pred` kedua model |
| Akurasi per jenis sumber | Tabulasi silang | `source_type`, `evidence_year`, `sektor`, `benar` |
| Taksonomi kesalahan | Klasifikasi manual atas baris salah | `tool_calls_log` di JSONL, `n_search_gagal` |
| Integritas sumber | Proporsi URL karangan dan domain non-`.go.id` | `url_dari_search`, `domain_goid` |

Himpunan baris tempat kedua model **berbeda pendapat** adalah bahan paling padat untuk analisis kesalahan, dan kemungkinan besar beririsan dengan baris yang gold-nya perlu diadjudikasi.
