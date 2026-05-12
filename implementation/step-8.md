# Step 8 — Evaluasi: ukur kualitas sebelum "produksi"

**Referensi README:** bab 4.2 (baris "Evaluasi"), 5.2 (observability).

## Tujuan
Punya angka, bukan firasat. Buat set pertanyaan acuan, ukur apakah jawaban setia pada dokumen (tidak mengarang), relevan, dan retrieval-nya menemukan yang benar.

## Prasyarat
- Step 4–5 selesai (RAG jalan dengan sitasi).
- Kumpulan pertanyaan uji dari step 3 & 5.

## Langkah

1. **Bangun gold set** — 30–50 pasangan `{pertanyaan, jawaban_acuan, dokumen_sumber_yang_benar}` yang mencakup:
   - Pertanyaan faktual dari dokumen (PTKP, tarif, isi pasal, klausul kontrak).
   - Pertanyaan yang jawabannya **tidak ada** di dokumen → acuan = "tidak ditemukan".
   - Pertanyaan ambigu / butuh klarifikasi.
   - (Jika ada tool) pertanyaan yang harus memicu `hitung_pph21`.

2. **Pilih metrik**
   | Metrik | Arti |
   |---|---|
   | **Faithfulness** | Jawaban benar-benar didukung context (tidak mengarang) — paling penting untuk pajak/legal |
   | **Answer relevancy** | Jawaban menjawab pertanyaan |
   | **Context recall** | Retrieval mengambil chunk yang memang berisi jawabannya |
   | **Context precision** | Chunk yang diambil sebagian besar relevan (tidak banyak sampah) |
   | Refusal accuracy | Untuk pertanyaan "di luar dokumen", model benar-benar bilang tidak tahu |

3. **Jalankan evaluasi**
   - Otomatis: **`ragas`** (`pip install ragas`) — bisa pakai LLM lokal sebagai judge (Ollama `qwen2.5:7b`) untuk faithfulness/relevancy.
   - Manual: skor tiap jawaban 0–2 berdasarkan jawaban_acuan; hitung rata-rata. Untuk set kecil, ini sering lebih akurat daripada auto-judge.
   - Catat juga: latensi rata-rata, % jawaban yang menyertakan sumber benar.

4. **Iterasi** — kalau skor jelek, balik ke step 5: perbaiki chunking / hybrid weights / top-k / rerank / prompt. Ukur lagi. Simpan riwayat skor.

5. **Logging untuk evaluasi berkelanjutan** — simpan tiap query produksi: pertanyaan, chunk yang di-retrieve, prompt final, jawaban, tool calls. (Redaksi PII sebelum disimpan — lihat step-9.) Tinjau berkala untuk menemukan kegagalan baru.

## Verifikasi — selesai bila…
- Ada file `eval/gold_set.jsonl` (≥30 item) dan skrip/notebook evaluasi.
- Skor faithfulness, relevancy, context recall terukur dan terdokumentasi.
- Ada ambang yang kamu tetapkan (mis. "faithfulness ≥ 0.9, refusal accuracy ≥ 0.95") sebelum dianggap layak dipakai untuk keputusan nyata.

## Catatan
- Tanpa evaluasi, "kelihatannya bagus" mudah menipu — terutama bahaya untuk pajak/legal.
- Gold set adalah aset: perbarui saat ada jenis pertanyaan baru atau dokumen baru.

➡️ Lanjut: [step-9.md](step-9.md)
