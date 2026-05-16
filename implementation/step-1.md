# Step 1 — Install Ollama, tarik model, tes cepat

**Referensi README:** bab 3.1–3.4.

## Tujuan
Ollama terpasang & jalan, model inti ke-download, terbukti bisa inference di GPU.

## Prasyarat
- Windows 11, koneksi internet, ~30 GB disk kosong.
- Driver NVIDIA terbaru (sudah ada — RTX 4060 Ti).

## Langkah

1. **Install Ollama** — pilih salah satu cara (Windows):
   ```powershell
   # Cara 1: winget
   winget install Ollama.Ollama

   # Cara 2: installer .exe — download lalu klik dua kali
   #   https://ollama.com/download/OllamaSetup.exe

   # Cara 3: one-liner PowerShell
   irm https://ollama.com/install.ps1 | iex
   ```
   Untuk OS lain:
   - **Linux:** `curl -fsSL https://ollama.com/install.sh | sh`
   - **macOS:** download `Ollama.dmg` dari <https://ollama.com/download/Ollama.dmg>
   - Daftar lengkap: <https://ollama.com/download>
   - Dokumentasi resmi (CLI, REST API, konfigurasi, troubleshooting): <https://docs.ollama.com/>

   Setelah ini Ollama jalan sebagai service (di Windows: icon di system tray), listen di `http://localhost:11434`.

2. **Verifikasi**
   ```powershell
   ollama --version
   ```

3. **Tarik model**
   ```powershell
   ollama pull qwen2.5:7b        # teks: Q&A / draft / analisis
   ollama pull qwen2.5vl:7b      # vision + teks (OCR dokumen/scan)
   ollama pull bge-m3            # embedding untuk RAG (dipakai step 4)
   # opsional:
   ollama pull gemma3:4b
   ```

4. **Tes teks**
   ```powershell
   ollama run qwen2.5:7b "Buatkan draft job description Backend Engineer (Java, Spring Boot), 5 poin tanggung jawab."
   ```
   Ketik `/bye` untuk keluar.

5. **Cek pemakaian GPU** (saat model masih jalan, terminal lain)
   ```powershell
   ollama ps
   ```
   Kolom `PROCESSOR` idealnya `100% GPU`. Kalau ada `% CPU` → sebagian spill ke RAM (wajar untuk model >8 GB; untuk 7B harusnya full GPU).

6. **(Opsional) Model kustom dengan system prompt** — `Modelfile`:
   ```dockerfile
   FROM qwen2.5:7b
   SYSTEM """Kamu asisten HR & pajak internal. Jawab dalam Bahasa Indonesia formal.
   Jika info tidak ada di dokumen yang diberikan, katakan tidak tahu — jangan menebak.
   Untuk pertanyaan pajak/hukum, akhiri dengan: 'Verifikasi dengan konsultan pajak/HR sebelum dipakai resmi.'"""
   PARAMETER temperature 0.2
   PARAMETER num_ctx 8192
   ```
   ```powershell
   ollama create hr-asisten -f Modelfile
   ollama run hr-asisten
   ```

## Verifikasi — selesai bila…
- `ollama --version` keluar nomor versi.
- `ollama list` menampilkan `qwen2.5:7b`, `qwen2.5vl:7b`, `bge-m3`.
- `ollama run qwen2.5:7b "..."` membalas, dan `ollama ps` menunjukkan `100% GPU`.

## Catatan
- Tutup app berat (browser banyak tab, game) sebelum memakai `qwen2.5vl:7b` — VRAM 8 GB cukup ketat untuk vision 7B.
- Model disimpan di `C:\Users\<user>\.ollama\models` — pastikan disk cukup.

➡️ Lanjut: [step-2.md](step-2.md)
