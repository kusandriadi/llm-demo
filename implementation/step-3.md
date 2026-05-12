# Step 3 — Validasi cepat tanpa kode: AnythingLLM + upload dokumen

**Referensi README:** bab 7.2 (aplikasi siap pakai), 3.6.

## Tujuan
Sebelum menulis kode RAG sendiri, buktikan dulu: apakah `qwen2.5:7b` + retrieval cukup baik untuk dokumen HR/pajak kamu? Ini menghemat waktu — kalau hasilnya jelek, lebih baik tahu sekarang.

## Prasyarat
- Step 1 selesai (Ollama jalan, `qwen2.5:7b` ada).
- Beberapa dokumen contoh: 1–2 PDF peraturan pajak / kebijakan perusahaan / kontrak, plus beberapa CV.

## Langkah

### Opsi A — AnythingLLM (desktop app, paling simpel)
1. Download & install AnythingLLM Desktop: https://anythingllm.com (Windows).
2. Saat setup, pilih **LLM Provider: Ollama** → base URL `http://localhost:11434` → model `qwen2.5:7b`.
   Embedding: bisa pakai bawaan AnythingLLM, atau Ollama `bge-m3`.
3. Buat **Workspace** baru (mis. "Pajak", "Recruitment") — pisahkan per topik.
4. Upload dokumen ke workspace (drag-drop pdf/docx/xlsx).
5. Tanya: *"Berapa PTKP untuk wajib pajak kawin dengan 1 tanggungan menurut dokumen?"* — perhatikan apakah jawaban benar & menyebut sumber.

### Opsi B — Open WebUI (via Docker, UI di browser)
```powershell
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway `
  -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
```
Buka `http://localhost:3000` → Settings → Connections → Ollama base URL `http://host.docker.internal:11434`. Lalu pakai fitur **Documents** untuk RAG.

## Yang dinilai (catat hasilnya)
- Akurasi jawaban atas dokumen pajak/kebijakan — benar? lengkap?
- Apakah menyertakan sumber/kutipan?
- Kualitas Bahasa Indonesia.
- Kecepatan respons (≈ tok/s) — nyaman dipakai?
- Coba pertanyaan "jebakan": yang jawabannya **tidak ada** di dokumen — apakah dia jujur bilang tidak tahu, atau mengarang?

## Verifikasi — selesai bila…
- Kamu sudah punya kesimpulan: "model + RAG dasar ini cukup" atau "perlu model lebih besar / chunking lebih baik / hybrid search".
- Kalau cukup → lanjut bangun versi sendiri (step 4) dengan percaya diri.
- Kalau kurang → catat apa yang kurang; akan diperbaiki di step 4–5 (chunking, hybrid search, re-rank) atau pertimbangkan `qwen2.5:14b` / `gemma3:12b`.

## Catatan
- Ini langkah **opsional tapi sangat disarankan**. Boleh dilewati kalau kamu langsung mau ngoding — tapi banyak pelajaran datang dari melihat baseline ini dulu.
- AnythingLLM/Open WebUI bisa terus dipakai sebagai antarmuka sehari-hari walaupun nanti kamu bangun pipeline sendiri untuk kasus khusus.

➡️ Lanjut: [step-4.md](step-4.md)
