# Step 5 — Tingkatkan RAG: hybrid search, re-rank, sitasi, disclaimer

**Referensi README:** bab 4.2 (tabel teknik).

## Tujuan
Naikkan akurasi & kepercayaan: pencarian lebih tepat untuk istilah spesifik, jawaban menyebut sumber, dan ada guardrail untuk pajak/hukum.

## Prasyarat
- Step 4 selesai (RAG dasar jalan, ada `./chroma_db`).

## Langkah (kerjakan bertahap, ukur efek tiap perubahan)

1. **Structure-aware chunking** — untuk peraturan, potong per pasal/ayat alih-alih buta. Simpan `pasal`, `bab`, `source`, `effective_date` di metadata.

2. **Hybrid search** (dense + keyword/BM25) — penting untuk "PPh 21", nomor pasal, NPWP, nama orang:
   - LlamaIndex: `QueryFusionRetriever` (gabung vector retriever + `BM25Retriever`).
   - LangChain: `EnsembleRetriever([bm25_retriever, vector_retriever], weights=[0.4, 0.6])`.

3. **Metadata filtering** — saat query, batasi ke dokumen relevan (mis. `doc_type == "pajak"` dan `effective_date` terbaru). Cegah jawaban dari peraturan kadaluarsa.

4. **Re-ranking** — ambil top-20 dari retrieval → urutkan ulang → kirim top-4–6 ke LLM:
   - Pakai cross-encoder (mis. `BAAI/bge-reranker-v2-m3` via `sentence-transformers`, jalan di CPU), atau
   - LLM-rerank (LlamaIndex `LLMRerank`).

5. **Sitasi sumber di jawaban** — instruksikan model menyertakan `[sumber: nama_file, pasal/hal]` untuk tiap klaim; tampilkan kutipan asli di bawah jawaban. (LlamaIndex: `CitationQueryEngine`.)

6. **Guardrail / disclaimer** — system prompt:
   > "Jawab HANYA berdasarkan konteks yang diberikan. Jika tidak ada di konteks, katakan 'informasi tidak ditemukan di dokumen'. Untuk pertanyaan pajak/hukum, akhiri dengan: 'Estimasi/ringkasan — verifikasi dengan konsultan pajak/HR sebelum dipakai resmi.'"
   - Set `temperature` 0.1–0.3.

7. **(Opsional) Query rewriting / HyDE** — untuk pertanyaan pendek/ambigu, perjelas atau buat jawaban-dugaan dulu lalu cari dokumen mirip.

## Verifikasi — selesai bila…
- Pertanyaan dengan istilah spesifik (nomor pasal, "PPh 21", nama) sekarang mengambil chunk yang benar (bandingkan sebelum/sesudah hybrid + rerank).
- Jawaban menyertakan sumber yang bisa ditelusuri.
- Pertanyaan di luar dokumen → model bilang "tidak ditemukan", bukan mengarang.
- Pertanyaan pajak → ada disclaimer.

## Catatan
- Tambah satu teknik, uji, baru lanjut — biar tahu mana yang benar-benar membantu.
- Re-ranker jalan di CPU (kecil) — tidak mengganggu VRAM.
- Simpan ~10 pertanyaan uji sekarang; akan dipakai sebagai cikal-bakal set evaluasi di [step-8.md](step-8.md).

➡️ Lanjut: [step-6.md](step-6.md)
