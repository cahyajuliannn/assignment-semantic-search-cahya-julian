# Semantic Search & LLM Application: FAQ E-Commerce

Prototype aplikasi *customer service* berbasis **semantic search** yang terintegrasi dengan **LLM**. Sistem mencari entri FAQ yang paling relevan dengan pertanyaan pengguna, lalu meminta LLM menyusun jawaban yang hanya berdasarkan FAQ tersebut.

## 1. Problem Statement dan Knowledge Base

**Problem statement**

Pelanggan e-commerce sering mengajukan pertanyaan yang sama dengan kata-kata yang berbeda-beda, misalnya "kenapa paket saya belum sampai?" atau "bisa bayar dicicil per bulan?". Pencarian berbasis kata kunci gagal bila kata yang dipakai pengguna berbeda dari kata di FAQ. Selain itu, LLM yang dipakai langsung tanpa acuan berisiko mengarang jawaban (halusinasi) tentang kebijakan toko.

Proyek ini menjawab dua masalah tersebut dengan:
1. **Semantic search** untuk menangkap *makna* pertanyaan, bukan sekadar kecocokan kata.
2. **Prompt terstruktur** yang membatasi LLM agar hanya menjawab berdasarkan FAQ hasil retrieval, dan mengarahkan pengguna ke customer service bila informasi tidak tersedia.

**Knowledge base**

| Item | Keterangan |
|------|------------|
| File | `data/ecommerce_faq_knowledge_base.csv` |
| Kolom | `faq_id`, `question`, `answer`, `category` |
| Unit retrievable | Satu entri FAQ = satu teks `Q: {question} A: {answer}` |
| Topik | FAQ e-commerce (status pesanan, pembayaran, cicilan, paket rusak, dll.) |

## 2. Alur Sistem

```
CSV FAQ → Ingest → Embed → Store (FAISS) → Retrieve → Generate (LLM) → Jawaban
```

| Tahap | Penjelasan | Implementasi |
|-------|------------|--------------|
| **Ingest** | Memuat CSV, lalu menggabungkan `question` dan `answer` menjadi satu kolom `content` per entri. | `pandas` |
| **Embed** | Setiap `content` diubah menjadi vektor lewat Gemini Embeddings API. | `gemini-embedding-2` |
| **Store** | Seluruh vektor disimpan di indeks FAISS dengan jarak L2. | `faiss.IndexFlatL2` |
| **Retrieve** | Query pengguna di-embed dengan model yang sama, lalu dicari `top_k` (default 3) vektor terdekat. Hasil berisi `rank`, `question`, `answer`, `category`, dan `distance`. | `semantic_search()` |
| **Generate** | Hasil retrieval disisipkan ke prompt (Persona, Task, Context, Format), lalu dikirim ke LLM. | `gemini-3.6-flash`, `temperature=0.3` |

**Desain prompt**

- **Persona**: asisten customer service e-commerce yang ramah dan membantu.
- **Task**: jawab pertanyaan HANYA berdasarkan FAQ yang diberikan. Jika informasi tidak ada, katakan belum memiliki informasinya dan sarankan menghubungi customer service.
- **Context**: top-k FAQ hasil semantic search (pertanyaan dan jawaban).
- **Format**: jawaban singkat dalam bahasa Indonesia, diawali pertanyaan pengguna dan diakhiri `Jawaban:`.

## 3. Hasil Pengujian Query dan Analisis Relevansi

Pengujian dilakukan dengan 5 query: 2 query yang ada di FAQ, 2 query parafrase/sinonim, dan 1 query di luar knowledge base.

| No | Jenis Query | Query | Sumber Teratas | Retrieval | Respons LLM |
|----|-------------|-------|----------------|-----------|-------------|
| 1 | Ada di FAQ (redaksi sedikit berbeda) | Bagaimana cara melacak status pesanan saya? | Bagaimana cara mengetahui status pesanan saya saat ini? | Tepat | Sesuai FAQ |
| 2 | Ada di FAQ (identik) | Metode pembayaran apa saja yang tersedia? | Metode pembayaran apa saja yang tersedia? | Tepat | Sesuai FAQ |
| 3 | Parafrase | bisa bayar dicicil per bulan? | Apakah saya bisa membayar dengan cicilan? | Tepat | Sesuai FAQ |
| 4 | Parafrase | barang yang saya terima rusak, uangnya bisa dikembalikan nggak? | Apa yang harus dilakukan jika paket diterima dalam kondisi rusak? | Relevan sebagian | Menjawab bagian yang ada, menolak bagian refund |
| 5 | Tidak ada di FAQ | Siapa CEO perusahaan ini? | Bagaimana cara mengetahui status pesanan saya saat ini? | Tidak relevan (wajar) | Menolak menjawab |

### Query yang berhasil di-retrieve dengan baik

- Query 1 dan 2 mengambil FAQ yang tepat di peringkat teratas. Query 2 identik dengan pertanyaan di FAQ, sedangkan query 1 hanya berbeda redaksi ("melacak" vs "mengetahui").
- Query parafrase 3 ("bisa bayar dicicil per bulan?") tetap menemukan FAQ "Apakah saya bisa membayar dengan cicilan?" meskipun kalimatnya berbeda dan lebih santai. Ini menunjukkan embedding menangkap makna ("dicicil" ≈ "cicilan"), bukan sekadar kecocokan kata kunci.
- Pada query 1-3, LLM menjawab sesuai isi FAQ tanpa menambah informasi di luar konteks.

### Query dengan hasil kurang relevan atau sebagian relevan

- Query 4 menggabungkan dua maksud: barang rusak dan pengembalian dana (refund). Retrieval mengambil FAQ paket rusak di peringkat 1, yang sesuai dengan maksud pertama, tetapi FAQ tersebut tidak menjelaskan refund. Akibatnya jawaban hanya menjawab sebagian pertanyaan. Kemungkinan penyebabnya adalah pertanyaan multi-topik atau belum adanya FAQ khusus refund. Hal ini perlu diverifikasi dengan melihat peringkat 2-3 hasil retrieval.

### Respons LLM saat informasi tidak tersedia

- Pada query 5, retrieval tetap mengembalikan FAQ yang tidak berhubungan ("status pesanan"). Ini wajar karena FAISS selalu mengembalikan top-k hasil terdekat, seberapa pun jauhnya.
- Meski konteksnya tidak relevan, LLM tidak mengarang jawaban. LLM menyatakan belum memiliki informasi dan menyarankan menghubungi Customer Service, sesuai instruksi prompt.
- Pada query 4, LLM juga jujur menyebut bagian refund tidak ada di FAQ, sambil tetap memberi informasi yang tersedia.
- Kesimpulannya, desain prompt (persona, batasan "hanya berdasarkan FAQ", dan instruksi fallback) efektif mencegah halusinasi pada pengujian ini.

## 4. Keterbatasan Sistem

1. **Tidak ada deteksi query di luar knowledge base.** FAISS selalu mengembalikan top-k, sehingga query seperti "Siapa CEO perusahaan ini?" tetap mendapat konteks yang tidak relevan. Pencegahan halusinasi sepenuhnya bergantung pada prompt, dan setiap query tetap memanggil LLM sehingga membuang biaya API.
2. **Query multi-topik hanya terjawab sebagian.** Contohnya query 4: sistem menjawab bagian "barang rusak" tetapi tidak bagian "refund".
3. **Cakupan terbatas pada isi FAQ.** Sistem tidak bisa menjawab hal yang tidak tertulis di CSV, dan tidak bisa mengakses data real-time seperti status pesanan aktual pengguna.
4. **Metrik jarak L2 kurang mudah ditafsirkan.** Nilai `distance` tidak punya batas atas yang jelas, sehingga menentukan ambang "relevan" atau "tidak relevan" lebih sulit dibanding cosine similarity.
5. **Pengujian masih kecil dan kualitatif.** Hanya 5 query dengan penilaian manual, tanpa metrik kuantitatif seperti precision@k atau recall@k.
6. **Embedding dihitung ulang setiap dijalankan.** Vektor tidak disimpan ke disk dan embedding dilakukan satu per satu, sehingga lambat bila knowledge base membesar.
7. **Tanpa riwayat percakapan.** Setiap pertanyaan diproses terpisah sehingga pertanyaan lanjutan ("kalau yang tadi?") tidak dipahami.
8. **Ketergantungan pada layanan eksternal.** Sistem membutuhkan koneksi internet, API key Gemini, dan tunduk pada batas kuota API.

**Saran pengembangan:** tambahkan threshold `distance` agar query di luar cakupan langsung dijawab dengan pesan fallback tanpa memanggil LLM; ganti ke cosine similarity (normalisasi vektor + `IndexFlatIP`); pecah query multi-topik menjadi sub-pertanyaan; simpan indeks FAISS ke disk (`faiss.write_index`); serta tambahkan evaluasi kuantitatif dengan kumpulan query berlabel.

## 5. Cara Menjalankan

### Prasyarat

- Python 3.12
- API key Gemini (dapat dibuat di Google AI Studio)

### Struktur folder

```
.
├── data/
│   └── ecommerce_faq_knowledge_base.csv
├── notebooks/
│   └── <nama_notebook>.ipynb
└── README.md
└── requirements.txt
```

Notebook membaca data dengan path relatif `../data/ecommerce_faq_knowledge_base.csv`, jadi jalankan notebook dari dalam folder `notebooks/`.

### Langkah-langkah

1. **Instal dependensi**

   ```bash
   pip install pandas numpy faiss-cpu google-genai jupyter
   ```
   atau

   ```bash
   pip install -r semantic-search-assignment/requirements.txt
   ```

2. **Set API key sebagai environment variable**

   Linux/macOS:
   ```bash
   export GEMINI_API_KEY="isi_api_key_kamu"
   ```
   Windows (PowerShell):
   ```powershell
   $env:GEMINI_API_KEY="isi_api_key_kamu"
   ```

3. **Jalankan Jupyter dari folder `notebooks/`**

4. **Jalankan sel notebook secara berurutan (Section 1 sampai 8).**

5. **Coba query sendiri pada Section 8**

### Catatan

- Nama model (`gemini-embedding-2` dan `gemini-3.6-flash`) dapat disesuaikan dengan model yang tersedia di akun Gemini API kamu.
- Jangan menyimpan API key langsung di dalam notebook atau repository.