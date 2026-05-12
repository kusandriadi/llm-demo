# Implementasi — Ringkasan & Peta Langkah

Pecahan dari [`../README.md`](../README.md) menjadi langkah-langkah yang bisa dikerjakan satu per satu.
Tujuan akhir: asisten lokal (Ollama + RAG) untuk HR, pajak & recruiter — privat, jalan di PC ini.

## Hardware & target

| | |
|---|---|
| GPU | RTX 4060 Ti — 8 GB VRAM (model 7–8B Q4 muat penuh → cepat) |
| CPU / RAM | i7-14700F (20c/28t) / 32 GB |
| OS | Windows 11 |
| Runtime | Ollama (`http://localhost:11434`) |
| Arsitektur | RAG (jawaban berbasis dokumen sumber, bukan ingatan model) |

## Model stack

| Peran | Model | Ukuran |
|---|---|---|
| Vision + teks (OCR dokumen/scan) | `qwen2.5vl:7b` | ~6 GB |
| Teks: Q&A / draft / analisis | `qwen2.5:7b` | ~4.7 GB |
| Embedding (RAG) | `bge-m3` | ~1.2 GB |
| Guard (opsional, keamanan) | `llama-guard3` | ~3 GB |
| Ringan / volume besar (opsional) | `gemma3:4b` | ~3.3 GB |

## Urutan langkah

| # | File | Isi | Referensi README |
|---|---|---|---|
| 1 | [step-1.md](step-1.md) | Install Ollama + tarik model + tes cepat | bab 3 |
| 2 | [step-2.md](step-2.md) | Vision/OCR — ekstrak data CV scan → JSON dengan `qwen2.5vl:7b` | bab 2, 3.3 |
| 3 | [step-3.md](step-3.md) | Validasi cepat tanpa kode — AnythingLLM + upload dokumen | bab 7.2, 3.6 |
| 4 | [step-4.md](step-4.md) | Bangun RAG minimal sendiri (LlamaIndex / LangChain) | bab 4.1, 4.3, 7 |
| 5 | [step-5.md](step-5.md) | Tingkatkan RAG: hybrid search + re-rank + sitasi + disclaimer | bab 4.2 |
| 6 | [step-6.md](step-6.md) | Tambah tool via MCP server (`hitung_pph21`, `cari_karyawan`) | bab 6.1 |
| 7 | [step-7.md](step-7.md) | Rapikan jadi Skill, lalu bundel jadi Plugin | bab 6.2–6.3 |
| 8 | [step-8.md](step-8.md) | Evaluasi: ~30 Q&A, ukur faithfulness sebelum "produksi" | bab 4.2 (Evaluasi) |
| 9 | [step-9.md](step-9.md) | Hardening keamanan: cegah prompt injection + security layer | bab 8 |

## Cara pakai

- Kerjakan **berurutan**. Tiap file punya: *Tujuan*, *Prasyarat*, *Langkah*, *Verifikasi (selesai bila…)*, *Catatan*.
- Langkah 1–2 wajib dulu. Langkah 3 opsional tapi sangat disarankan (validasi cepat sebelum ngoding).
- Langkah 9 (keamanan) **bukan opsional** kalau asisten akan memproses dokumen dari pihak luar (CV pelamar, upload) — terapkan sambil mengerjakan langkah 4–7, jangan ditunda.
- Diagram arsitektur: [`../docs/arsitektur-rag.png`](../docs/arsitektur-rag.png) (pipeline RAG) & [`../docs/arsitektur-security.png`](../docs/arsitektur-security.png) (security layer). Sumber `.puml` di folder yang sama.

## Definition of Done (keseluruhan)

- [ ] Bisa tanya-jawab Bahasa Indonesia atas dokumen internal, jawaban menyertakan sumber + disclaimer untuk hal pajak/hukum.
- [ ] Bisa ekstrak data terstruktur (JSON tervalidasi skema) dari CV/slip, termasuk yang berupa gambar/scan.
- [ ] Ada minimal 1 tool (MCP) yang dipanggil model dengan least-privilege + audit log.
- [ ] Security layer aktif: input guard, pemisahan data/instruksi, output guard, tool broker, egress dibatasi.
- [ ] Set evaluasi ~30 Q&A jalan; skor faithfulness terukur.
- [ ] Semua proses lokal — tidak ada data yang keluar PC.
