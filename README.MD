# 📘 RAG Parameter Exploration Assignment

## 🎯 Tujuan Pembelajaran

Memahami fungsi tiap parameter RAG, mengamati trade-off antara kualitas jawaban dan kecepatan, serta menulis insight praktis dari eksperimen.

---

## ⚙️ Parameter yang Diuji

```python
# Bagian KONFIGURASI INTI di app.py
CHUNK_SIZE = 300
TOP_K_INITIAL = 20
FINAL_TOP_M = 5
USE_RERANKER = True
TEMPERATURE = 0.2
GROQ_MODEL = "llama-3.1-8b-instant"
```

---

## 📏 1. CHUNK_SIZE

```python
CHUNK_SIZE = 300  # ubah angka ini
```

**Fungsi:**
CHUNK_SIZE menentukan seberapa besar potongan teks yang dibuat dari dokumen PDF. Saat PDF diupload, isinya tidak diproses sekaligus, melainkan dipotong-potong menjadi bagian kecil. Angka ini mengatur berapa banyak kata dalam satu potongan. Potongan-potongan inilah yang dicari dan dicocokkan dengan pertanyaan.

**Pengaruhnya:**

| Range Nilai | Efek |
|---|---|
| Kecil (100–200) | Potongan jadi sangat spesifik sehingga pencarian lebih presisi — mudah menemukan bagian yang persis relevan. Tapi tiap potongan hanya membawa sedikit konteks, jadi jawaban berisiko terasa terpotong atau kehilangan gambaran besar. |
| Sedang (250–400) ⭐ Recommended | Ukuran ini seimbang. Potongan masih cukup fokus untuk dicari secara akurat, tapi juga membawa konteks yang memadai supaya jawaban tetap utuh dan masuk akal. Cocok untuk kebanyakan dokumen. |
| Besar (500–1000) | Tiap potongan membawa banyak konteks sekaligus, tapi karena isinya mencampur banyak topik, pencarian jadi kurang tajam. Skor kemiripan ikut "terencerkan" oleh kalimat yang tidak relevan, sehingga hasil retrieval bisa kurang tepat sasaran. |

**Pertanyaan Refleksi:**

**Chunk besar atau kecil yang lebih tepat untuk dokumen yang kamu tes? Mengapa?**

Untuk kedua dokumen yang saya tes, chunk ukuran sedang (300) memberi hasil terbaik. Saat saya turunkan ke 150, jawaban untuk pertanyaan konteks dan reasoning terasa terpotong — terutama pada PDF teknis, karena penjelasan teknis biasanya tersebar di beberapa kalimat yang saling berkaitan. Saat saya naikkan ke 700, jawaban memang lengkap tapi sistem beberapa kali mengambil potongan yang kurang relevan, karena satu potongan besar mencampur banyak topik. Nilai 300 memberi jawaban yang utuh tanpa mengorbankan ketepatan pencarian.

**Apa yang terjadi kalau jawaban terasa "terpotong" atau kurang konteks? Parameter mana yang harus diubah?**

Kalau jawaban terasa terpotong, artinya potongan teks yang diambil terlalu kecil sehingga tidak membawa cukup informasi. Parameter utama yang perlu dinaikkan adalah CHUNK_SIZE. Selain itu, FINAL_TOP_M juga bisa ditambah agar lebih banyak potongan dikirim ke LLM, dan CHUNK_OVERLAP membantu supaya informasi tidak terputus di batas antar potongan.

---

## 🔍 2. TOP_K_INITIAL

```python
TOP_K_INITIAL = 20
```

**Fungsi:**
TOP_K_INITIAL menentukan berapa banyak potongan kandidat yang diambil pada tahap pencarian awal. Saat pertanyaan diubah jadi vektor, sistem mencari potongan yang paling mirip. Angka ini ibarat ukuran "jaring" — semakin besar, semakin banyak kandidat yang dikumpulkan sebelum nanti disaring ulang oleh reranker.

**Pengaruhnya:**

| Range Nilai | Efek |
|---|---|
| Kecil (5–10) | Proses lebih cepat karena kandidat sedikit. Tapi recall rendah — kalau potongan yang benar-benar relevan tidak masuk 5–10 teratas, ia langsung hilang dan tidak akan pernah sampai ke LLM. |
| Sedang (15–25) ⭐ Recommended | Cukup banyak untuk menangkap hampir semua potongan relevan, tapi tidak terlalu besar sehingga proses tetap efisien. Memberi reranker bahan yang cukup untuk bekerja optimal. |
| Besar (30–50) | Recall paling tinggi karena hampir semua potongan relevan pasti tertangkap. Tapi reranker harus mengevaluasi lebih banyak pasangan, sehingga waktu proses bertambah dan banyak kandidat ternyata tidak terpakai. |

> 💡 Catatan: Nilai ini hanya berpengaruh signifikan jika `USE_RERANKER = True`.

**Pertanyaan Refleksi:**

**Apakah top-k kecil (5–10) cukup untuk dataset kamu? Jelaskan alasannya.**

Untuk kedua PDF saya yang tergolong pendek (2–3 halaman), top-k kecil sebenarnya masih cukup karena jumlah total potongan memang sedikit, sehingga peluang potongan relevan tidak tertangkap kecil. Saat saya pakai top-k 5, jawaban untuk pertanyaan definisi masih akurat. Namun pada pertanyaan reasoning yang jawabannya tersebar di beberapa bagian, top-k 5 kadang melewatkan satu potongan penting. Karena itu untuk amannya saya tetap memakai nilai sedang (20), apalagi kalau nanti dokumennya lebih panjang.

**Apa hubungan antara TOP_K_INITIAL dan USE_RERANKER?**

TOP_K_INITIAL adalah tahap "mengumpulkan kandidat", sedangkan reranker adalah tahap "menyaring dan mengurutkan ulang". Reranker hanya bisa memilih dari kandidat yang sudah dikumpulkan TOP_K_INITIAL — ia tidak bisa memunculkan potongan yang tidak ada di daftar. Jadi keduanya berpasangan: TOP_K_INITIAL menyediakan bahan baku, reranker memilih yang terbaik. Kalau reranker dimatikan, TOP_K_INITIAL nyaris tidak berpengaruh karena sistem langsung memakai FINAL_TOP_M teratas dari hasil cosine similarity.

---

## 📋 3. FINAL_TOP_M

```python
FINAL_TOP_M = 5
```

**Fungsi:**
FINAL_TOP_M menentukan berapa banyak potongan teks yang akhirnya benar-benar dikirim ke LLM sebagai bahan untuk menjawab. Dari sekian banyak kandidat hasil pencarian, hanya M potongan terbaik (setelah reranking) yang lolos. Inilah konteks final yang dibaca LLM.

**Pengaruhnya:**

| Range Nilai | Efek |
|---|---|
| Sedikit (2–3) | Jawaban sangat fokus dan to-the-point karena LLM hanya membaca informasi paling relevan. Tapi berisiko: kalau jawaban butuh informasi dari beberapa bagian dokumen, sebagian konteks bisa terlewat dan jawaban jadi tidak lengkap. |
| Sedang (4–6) ⭐ Recommended | Memberi LLM konteks yang cukup lengkap untuk menjawab dengan baik, tanpa membanjiri LLM dengan informasi berlebihan. Latency masih wajar. Pilihan paling seimbang. |
| Banyak (7–10) | Konteks paling lengkap, tapi LLM bisa "kebingungan" karena harus menyaring banyak informasi yang sebagian tidak relevan. Token yang dikirim lebih banyak sehingga jawaban lebih lambat dan lebih mahal. |

> 💡 Catatan: Semakin banyak konteks = semakin banyak token = latency lebih tinggi.

**Pertanyaan Refleksi:**

**Benarkah "semakin banyak konteks semakin bagus"? Jelaskan berdasarkan eksperimenmu.**

Tidak selalu benar. Saat saya naikkan FINAL_TOP_M dari 5 ke 9, jawaban tidak otomatis jadi lebih baik — pada pertanyaan definisi, jawaban malah jadi lebih panjang dan bertele-tele karena LLM ikut memasukkan informasi yang sebenarnya tidak ditanyakan. Pada FINAL_TOP_M 2, jawaban memang ringkas tapi pertanyaan reasoning jadi kurang lengkap. Nilai 5 memberi hasil paling rapi: jawaban lengkap tapi tetap fokus. Kesimpulannya, konteks yang relevan dan secukupnya lebih baik daripada konteks banyak tapi bercampur.

**Apa yang terjadi kalau konteks yang dikirim ke LLM saling bertentangan?**

Kalau potongan yang dikirim berisi informasi yang saling bertentangan, LLM bisa bingung menentukan mana yang benar. Hasilnya jawaban bisa jadi tidak konsisten, mencampur kedua versi, atau memilih salah satu secara acak. Inilah kenapa kualitas retrieval (memilih potongan yang tepat) sama pentingnya dengan jumlahnya — reranker dan score threshold membantu mengurangi masuknya konteks yang menyesatkan.

---

## 🎯 4. USE_RERANKER

```python
USE_RERANKER = True  # atau False
```

**Fungsi:**
USE_RERANKER mengaktifkan model tambahan (Cross-Encoder) yang menilai ulang seberapa cocok tiap potongan dengan pertanyaan. Pencarian vektor biasa hanya melihat kemiripan secara kasar (cosine similarity), sedangkan reranker membaca pasangan [pertanyaan + potongan] secara langsung dan menilai relevansinya lebih teliti, lalu mengurutkan ulang.

**Pengaruhnya:**

| Mode | Efek |
|---|---|
| True | Urutan potongan jadi jauh lebih akurat. Potongan yang benar-benar menjawab pertanyaan naik ke atas, sehingga konteks yang dikirim ke LLM lebih tepat dan jawaban lebih berkualitas. Konsekuensinya ada tambahan waktu proses karena model harus mengevaluasi tiap kandidat. |
| False | Sistem hanya mengandalkan cosine similarity. Lebih cepat karena tanpa langkah evaluasi ulang, tapi urutan relevansi kurang tajam — potongan yang sekadar "mirip kata" bisa menang atas potongan yang sebenarnya lebih menjawab pertanyaan. |

> 💡 Catatan: Untuk dokumen teknis atau pertanyaan ambigu, reranker sangat membantu. Untuk kecepatan maksimal, matikan.

**Pertanyaan Refleksi:**

**Pada query jenis apa reranker paling terasa manfaatnya? Berikan contoh.**

Reranker paling terasa manfaatnya pada pertanyaan reasoning dan pertanyaan yang kata-katanya tidak persis sama dengan teks dokumen. Contohnya, saat saya bertanya "Mengapa pendekatan ini lebih efisien?" pada PDF teknis, mode tanpa reranker mengambil potongan yang hanya mengandung kata "efisien" tapi tidak benar-benar menjelaskan alasannya, sehingga jawaban kurang tepat. Dengan reranker dinyalakan, potongan yang benar-benar berisi penjelasan sebab-akibat naik ke atas dan jawaban jadi tepat. Untuk pertanyaan definisi sederhana yang kata kuncinya jelas, perbedaannya tidak terlalu terasa.

**Berapa kira-kira tambahan latency saat reranker dinyalakan di eksperimenmu?**

Dari pengamatan saya, dengan PDF pendek dan TOP_K_INITIAL 20, menyalakan reranker menambah waktu respons sekitar 1,5–2,5 detik dibanding saat dimatikan. Tanpa reranker jawaban muncul kira-kira dalam 1–2 detik, sedangkan dengan reranker sekitar 3–4 detik. Tambahan ini sepadan untuk pertanyaan yang menuntut ketepatan.

---

## 🌡️ 5. TEMPERATURE

```python
TEMPERATURE = 0.2  # range: 0.0 - 1.0
```

**Fungsi:**
TEMPERATURE mengatur seberapa "berani" LLM saat memilih kata berikutnya. Nilai rendah membuat LLM cenderung memilih kata yang paling pasti, sehingga jawabannya konsisten dan dapat diprediksi. Nilai tinggi membuat LLM lebih berani memilih kata yang kurang umum, sehingga jawaban lebih bervariasi dan kreatif — tapi juga lebih berisiko menyimpang.

**Pengaruhnya:**

| Range Nilai | Efek |
|---|---|
| 0.0–0.2 ⭐ Recommended untuk RAG | Jawaban paling konsisten dan faktual. LLM patuh pada konteks dokumen dan tidak mengarang. Kalau ditanya pertanyaan sama berkali-kali, jawabannya hampir selalu serupa. |
| 0.3–0.5 | Masih cukup stabil, tapi ada sedikit variasi dalam pemilihan kata dan susunan kalimat. Jawaban terasa sedikit lebih luwes tanpa banyak menyimpang dari fakta. |
| 0.6–0.8 | Jawaban mulai kreatif dan bervariasi. Cocok untuk penulisan bebas, tapi untuk RAG mulai berisiko karena LLM bisa menambahkan hal yang tidak ada di dokumen. |
| 0.9–1.0 | Jawaban sangat acak dan tidak terduga. Risiko halusinasi tinggi — LLM bisa mengarang informasi yang tidak ada di dokumen. Tidak disarankan untuk RAG. |

> ⚠️ Catatan: Untuk RAG, gunakan temperature rendah (0.1–0.3) agar jawaban tetap grounded pada konteks.

**Pertanyaan Refleksi:**

**Untuk QA berbasis dokumen perusahaan, temperatur ideal berapa? Jelaskan alasannya.**

Temperatur ideal sekitar 0.1–0.2. Untuk dokumen perusahaan, yang dibutuhkan adalah jawaban yang akurat, konsisten, dan benar-benar berdasarkan isi dokumen — bukan jawaban kreatif. Temperatur rendah memastikan LLM tidak mengarang dan memberi jawaban yang sama setiap kali ditanya hal yang sama, sehingga bisa dipercaya untuk keperluan kerja.

**Apa yang terjadi pada jawaban saat kamu naikkan temperature ke 0.9?**

Saat saya naikkan ke 0.9, jawaban menjadi jauh lebih bervariasi dan kurang fokus. Ketika saya tanyakan pertanyaan yang sama tiga kali berturut-turut, ketiga jawabannya berbeda susunan dan pilihan katanya. Lebih dari itu, pada PDF teknis muncul satu-dua kalimat yang isinya tidak bisa saya temukan di dokumen aslinya — indikasi halusinasi. Ini menegaskan bahwa temperatur tinggi tidak cocok untuk RAG, karena tujuan RAG justru menjaga jawaban tetap berpegang pada dokumen.

---

## 🤖 6. GROQ_MODEL

```python
GROQ_MODEL = "llama-3.1-8b-instant"  # model default
```

**Fungsi:**
GROQ_MODEL menentukan model LLM yang dipakai untuk menyusun jawaban. Model dengan jumlah parameter lebih besar umumnya lebih pintar memahami konteks dan menyusun jawaban, tapi biasanya lebih lambat.

| Model | Ukuran | Pengaruh terhadap Kecepatan & Kualitas |
|---|---|---|
| llama-3.1-8b-instant (Default) | 8B parameter | Paling cepat — jawaban muncul hampir seketika. Kualitas sudah cukup baik untuk pertanyaan definisi dan fakta langsung. Pada pertanyaan reasoning, jawabannya kadang kurang dalam. Cocok untuk tahap pengembangan dan saat kecepatan diutamakan. |
| openai/gpt-oss-120b | 120B parameter | Paling besar, jawaban paling kaya dan paling baik dalam menangani pertanyaan kompleks dan reasoning. Konsekuensinya respons paling lambat (terasa beberapa detik lebih lama dari model 8B). Cocok dipakai saat akurasi sangat penting dan kecepatan bukan prioritas. |
| llama-3.3-70b-versatile | 70B parameter | Penengah yang seimbang. Lebih pintar dari model 8B dalam memahami konteks dan menyusun jawaban yang rapi serta terstruktur, tapi tidak selambat model 120B. Cocok saat butuh kualitas tinggi dengan kecepatan yang masih wajar. |

> 💡 Tips: Mulai dengan llama-3.1-8b-instant untuk development. Saat butuh akurasi tinggi, switch ke model yang lebih besar.

**Pertanyaan Refleksi:**

**Apakah peningkatan kualitas jawaban dari model besar sebanding dengan penambahan waktu? Jelaskan.**

Tergantung jenis pertanyaan. Untuk pertanyaan definisi dan fakta langsung, model 8B sudah memberi jawaban yang cukup baik, sehingga menunggu lebih lama untuk model besar kurang sebanding — bedanya tipis tapi waktunya jelas lebih lama. Tapi untuk pertanyaan reasoning, model 70B dan 120B memberi jawaban yang jelas lebih tepat, terstruktur, dan menjelaskan sebab-akibat dengan baik, sehingga tambahan waktunya terasa sepadan. Intinya, model besar layak dipakai saat pertanyaannya memang sulit.

**Model mana yang paling cocok untuk use-case dokumenmu? Mengapa?**

Untuk dokumen yang saya tes — terutama PDF teknis — llama-3.3-70b-versatile terasa paling cocok. Model ini cukup pintar memahami istilah teknis dan menyusun jawaban yang rapi untuk pertanyaan reasoning, tapi kecepatannya masih nyaman dipakai dan tidak selambat model 120B. Untuk sekadar uji coba cepat atau pertanyaan sederhana, model 8B sudah memadai.

---

## ⚙️ Parameter Tambahan (Non-Tunable)

```python
CHUNK_OVERLAP = 60     # overlap antar chunk (kata)
SCORE_THRESHOLD = 0.30 # skor minimum cosine similarity
```

| Parameter | Nilai Default | Fungsi |
|---|---|---|
| CHUNK_OVERLAP | 60 | Mengatur berapa banyak kata yang "dipinjam" dari potongan sebelumnya saat membuat potongan baru. Tujuannya supaya kalimat atau ide yang kebetulan terbelah di batas potongan tidak hilang konteksnya. Kalau terlalu kecil, informasi bisa terputus di sambungan antar potongan. Kalau terlalu besar, banyak teks terduplikasi sehingga jumlah potongan membengkak dan proses jadi kurang efisien. |
| SCORE_THRESHOLD | 0.30 | Batas skor kemiripan minimum agar sebuah potongan dianggap relevan. Potongan dengan skor di bawah ini dibuang. Threshold dinaikkan kalau jawaban sering memasukkan konteks yang tidak nyambung, supaya hanya yang benar-benar relevan lolos. Threshold diturunkan kalau sistem terlalu sering tidak menemukan jawaban padahal informasinya ada di dokumen. |

---

## 📝 Kesimpulan Eksperimen

Dari eksperimen ini, saya menyimpulkan bahwa konfigurasi default (CHUNK_SIZE 300, TOP_K_INITIAL 20, FINAL_TOP_M 5, reranker aktif, temperature 0.2) memang sudah seimbang untuk dokumen pendek. Kualitas jawaban RAG tidak hanya soal "memperbanyak" — chunk atau konteks yang berlebihan justru menurunkan fokus jawaban. Reranker memberi peningkatan kualitas yang nyata pada pertanyaan reasoning dengan tambahan waktu yang masih wajar. Temperature harus dijaga rendah agar jawaban tetap berpegang pada dokumen. Pemilihan model adalah trade-off antara kecepatan dan kedalaman jawaban, dan sebaiknya disesuaikan dengan tingkat kesulitan pertanyaan.

---

## 🔗 Referensi

- Dokumentasi parameter: `docs.html` atau menu 📖 Dokumentasi di RAG Playground
- Source code: `app.py` — bagian KONFIGURASI INTI
