# llm-demo — Asisten Lokal untuk HR, Pajak & Recruiter (Ollama + RAG)

Proyek belajar: menjalankan LLM **lokal** (offline, data tidak keluar dari PC) untuk membantu pekerjaan HRD, pajak, dan recruiter — membaca CV/kontrak/peraturan, ekstraksi data dari dokumen (termasuk gambar/scan), tanya-jawab atas dokumen internal, dan membuat draft (job description, surat, email).

Mesin: **Ollama** sebagai runtime model. Arsitektur: **RAG** (Retrieval-Augmented Generation) supaya jawaban berbasis dokumen sumber, bukan "ingatan" model.

---

## 1. Requirement

### Hardware (PC ini)

| Komponen | Spec | Implikasi |
|---|---|---|
| GPU | NVIDIA RTX 4060 Ti — **8 GB VRAM** | Model 7–8B kuantisasi Q4 muat penuh di GPU → cepat (~35–50 tok/s). Model vision 7B muat tapi ketat. Model 12–14B sebagian spill ke RAM (~8–12 tok/s). |
| CPU | Intel i7-14700F — 20 core / 28 thread | Cukup kuat untuk offload sebagian layer & menjalankan model 13–14B di RAM. |
| RAM | 32 GB | Aman untuk model sampai ~14B; 32B Q4 masih jalan tapi lambat. |
| OS | Windows 11 | Ollama mendukung Windows native + akselerasi CUDA. |
| Disk | Sediakan ≥ 30 GB kosong | Tiap model 4–9 GB; embedding model ~0.5–2 GB. |

### Software

- **Ollama** ≥ 0.5 (runtime + server di `http://localhost:11434`)
- **Python** ≥ 3.10 (untuk pipeline RAG & ekstraksi dokumen)
- Driver NVIDIA terbaru (CUDA) — sudah ada (591.86)
- Opsional: **Docker** (untuk vector DB seperti Qdrant) atau pakai vector store embedded (Chroma)

### Functional requirement

1. Tanya-jawab Bahasa Indonesia atas dokumen internal (peraturan pajak, PKB, kebijakan perusahaan).
2. Ekstraksi data terstruktur (JSON) dari CV — termasuk **CV hasil scan (gambar/PDF gambar)**.
3. Baca dokumen gambar: KTP, NPWP, slip gaji, sertifikat, formulir.
4. Draft teks: job description, surat peringatan, email penawaran.
5. **Privasi**: semua proses lokal — PII (gaji, NPWP, NIK, data kandidat) tidak dikirim ke layanan cloud.
6. Selalu sertakan **sumber** pada jawaban berbasis dokumen + disclaimer untuk hal hukum/pajak.

---

## 2. Model yang dipakai & alasannya

Stack model di PC ini (Ollama load/unload otomatis sesuai kebutuhan — tidak jalan bersamaan):

| Peran | Model | Ukuran | Kenapa |
|---|---|---|---|
| **Vision + teks (utama)** | `qwen2.5vl:7b` | ~6 GB | Qwen2.5-VL di-tune kuat untuk **OCR & dokumen** (formulir, invoice, tabel, scan) — persis kebutuhan HR/recruiter. Multilingual (ID/EN), bisa output JSON, muat di 8 GB VRAM. Lebih cocok daripada `llama3.2-vision` yang lebih ke "describe image" dan lemah Bahasa Indonesia. |
| Teks-only (analisis/JD/Q&A) | `qwen2.5:7b` | ~4.7 GB | Bahasa Indonesia paling natural di kelas 7–8B, context **128K** (muat kontrak panjang), jago instruksi & JSON. Muat penuh di GPU. |
| Embedding (untuk RAG) | `bge-m3` | ~1.2 GB | Embedding multilingual yang bagus untuk Bahasa Indonesia; mendukung dense + (opsional) sparse. Alternatif: `nomic-embed-text` (ringan, English-centris). |
| Ringan / volume besar (opsional) | `gemma3:4b` | ~3.3 GB | Multimodal, cepat, muat lega — untuk screening massal saat butuh throughput. |
| Kualitas lebih tinggi (opsional) | `gemma3:12b` | ~8.1 GB | Tulisan & reasoning lebih baik; sebagian ke RAM → ~8–12 tok/s. |

> **Catatan kunci untuk pajak/legal:** model 7–14B **tidak bisa dipercaya** untuk angka/aturan pajak (PTKP, tarif progresif, regulasi yang berubah tiap tahun) → **selalu** lewat RAG dengan dokumen sumber, dan beri disclaimer "verifikasi dengan konsultan pajak". Suhu rendah (`temperature 0.1–0.3`) untuk tugas ekstraksi agar tidak mengarang.

---

## 3. Step by step instalasi

### 3.1 Install Ollama

```powershell
winget install Ollama.Ollama
# atau download installer: https://ollama.com/download
```

Setelah install, Ollama berjalan sebagai service (icon di system tray) dan listen di `http://localhost:11434`. Verifikasi:

```powershell
ollama --version
```

### 3.2 Tarik model

```powershell
ollama pull qwen2.5vl:7b      # vision + teks (utama)
ollama pull qwen2.5:7b        # teks-only
ollama pull bge-m3            # embedding untuk RAG
# opsional:
ollama pull gemma3:4b
```

Cek model lokal & status GPU:

```powershell
ollama list
ollama ps          # saat model jalan — kolom PROCESSOR idealnya "100% GPU"
```

### 3.3 Tes cepat

```powershell
# teks
ollama run qwen2.5:7b "Buatkan draft job description untuk Backend Engineer (Java, Spring Boot), 5 poin tanggung jawab."

# vision — tulis path gambar di dalam prompt
ollama run qwen2.5vl:7b
>>> Ekstrak nama, email, no HP, dan skills dari CV ini dalam JSON: C:\data\cv\contoh.png
```

### 3.4 (Opsional) Buat model kustom dengan system prompt

`Modelfile`:

```dockerfile
FROM qwen2.5vl:7b
SYSTEM """Kamu asisten HR & pajak internal. Baca dokumen dengan teliti.
Jawab dalam Bahasa Indonesia formal. Jika sebuah field tidak terbaca atau tidak ada, tulis null — jangan menebak.
Untuk pertanyaan pajak/hukum, jawab hanya berdasarkan dokumen yang diberikan dan akhiri dengan: 'Verifikasi dengan konsultan pajak/HR sebelum dipakai resmi.'"""
PARAMETER temperature 0.2
PARAMETER num_ctx 8192
```

```powershell
ollama create hr-asisten -f Modelfile
ollama run hr-asisten
```

### 3.5 Setup environment Python (untuk RAG)

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install langchain langchain-community langchain-ollama chromadb `
            unstructured "unstructured[docx,xlsx,pptx]" pymupdf openpyxl pandas `
            pillow fastapi uvicorn
```

### 3.6 (Opsional) Antarmuka tanpa ngoding

Kalau mau langsung pakai (drag-drop dokumen lewat browser, RAG built-in):

```powershell
# Open WebUI via Docker
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway `
  -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main
# buka http://localhost:3000  → set Ollama base URL: http://host.docker.internal:11434
```

Alternatif: **AnythingLLM** (desktop app) — juga punya RAG + upload dokumen + koneksi ke Ollama.

---

## 4. Implementasi RAG — teknik & langkah

**RAG = Retrieval-Augmented Generation.** Alih-alih bertanya langsung ke model, kita: (1) simpan dokumen sebagai vektor, (2) saat ada pertanyaan, ambil potongan dokumen paling relevan, (3) kirim potongan itu + pertanyaan ke model, (4) model menjawab berdasarkan potongan tersebut + menyebut sumber.

### 4.1 Pipeline (ingestion → query)

```
INGESTION (sekali / saat dokumen berubah)
  File (pdf, docx, xlsx, gambar)
    → Ekstraksi teks         (unstructured / PyMuPDF / OCR via qwen2.5vl)
    → Cleaning & normalisasi  (hapus header/footer, rapikan tabel jadi Markdown)
    → Chunking                (potong ~500–1000 token, overlap ~100–150)
    → Embedding               (bge-m3 via Ollama)
    → Simpan ke Vector Store  (Chroma / Qdrant) + metadata (nama file, halaman, tanggal)

QUERY (tiap pertanyaan user)
  Pertanyaan user
    → (opsional) Query rewriting / HyDE
    → Embedding pertanyaan    (bge-m3)
    → Retrieval top-k         (similarity search di vector store)
    → (opsional) Re-ranking   (cross-encoder / LLM-rerank → ambil top-n terbaik)
    → Susun Prompt            (context = chunk terpilih + instruksi + pertanyaan)
    → LLM generate            (qwen2.5:7b — atau qwen2.5vl bila ada gambar)
    → Jawaban + sitasi sumber → (opsional) cek "grounded?" sebelum ditampilkan
```

### 4.2 Teknik yang dipakai (dan kapan)

| Teknik | Fungsi | Catatan untuk proyek ini |
|---|---|---|
| **Document parsing/OCR** | Ubah pdf/docx/xlsx/scan jadi teks | `unstructured` (multi-format), `PyMuPDF` (PDF teks), `qwen2.5vl` (PDF scan/gambar). xlsx → ubah ke Markdown table, model lebih paham. |
| **Chunking** | Pecah dokumen jadi potongan yang muat di context & relevan | Mulai: 800 token, overlap 120. Untuk peraturan: chunk per pasal/ayat (`structure-aware splitting`) jauh lebih baik daripada potong buta. |
| **Embedding** | Ubah teks → vektor untuk pencarian semantik | `bge-m3` (multilingual, bagus ID). Konsisten — pakai model embedding yang sama saat ingest & query. |
| **Vector store** | Simpan & cari vektor cepat | `Chroma` (embedded, paling simpel untuk mulai) → naik ke `Qdrant` (Docker) kalau data besar / butuh filter metadata kompleks. |
| **Hybrid search** | Gabung pencarian semantik (dense) + kata kunci (BM25/sparse) | Penting untuk istilah spesifik: "PPh 21", nomor pasal, NPWP, nama orang — keyword sering kalah kalau cuma dense. |
| **Metadata filtering** | Batasi pencarian (mis. hanya dokumen "Pajak 2025", atau departemen tertentu) | Simpan `source`, `page`, `effective_date`, `doc_type` di metadata. |
| **Re-ranking** | Urutkan ulang hasil retrieval dengan model lebih teliti | Ambil top-20 → rerank → kirim top-4–6 ke LLM. Naikkan akurasi signifikan. |
| **Query rewriting / multi-query** | Perbaiki pertanyaan ambigu / pecah jadi beberapa sub-query | Mis. "berapa pajak saya?" → diperjelas dulu (gaji, status PTKP, dll). |
| **HyDE** (Hypothetical Document Embeddings) | Buat jawaban dugaan dulu, lalu cari dokumen mirip jawaban itu | Berguna kalau pertanyaan pendek/kabur. |
| **Citation / grounding check** | Pastikan jawaban benar-benar berasal dari context | Wajib untuk pajak/legal. Tampilkan kutipan + nama dokumen + halaman; tolak jawab kalau context tidak relevan. |
| **Guardrail / refusal** | Jangan menebak | System prompt: "jika tidak ada di dokumen, katakan tidak tahu". `temperature` rendah. |
| **Evaluasi** | Ukur kualitas RAG | Buat ~30–50 pasangan tanya-jawab acuan; cek `faithfulness` (tidak ngarang), `answer relevancy`, `context recall` (pakai `ragas` atau evaluasi manual). |
| **(Lanjutan) Agentic RAG** | LLM memutuskan kapan & apa yang di-retrieve, bisa multi-step | Pakai kalau pertanyaan kompleks lintas dokumen. Bisa dilewati di tahap awal. |

### 4.3 Contoh kode minimal (Chroma + Ollama)

`rag.py`:

```python
from langchain_community.document_loaders import UnstructuredFileLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_ollama import OllamaEmbeddings, ChatOllama
from langchain_community.vectorstores import Chroma
from langchain.chains import RetrievalQA

# --- INGESTION ---
docs = UnstructuredFileLoader("docs/pmk_pph21_2025.pdf").load()
chunks = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=120).split_documents(docs)

emb = OllamaEmbeddings(model="bge-m3")
vs = Chroma.from_documents(chunks, emb, persist_directory="./chroma_db")

# --- QUERY ---
llm = ChatOllama(model="qwen2.5:7b", temperature=0.2)
qa = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vs.as_retriever(search_kwargs={"k": 5}),
    return_source_documents=True,
)
res = qa.invoke({"query": "Berapa PTKP untuk wajib pajak kawin dengan 1 tanggungan?"})
print(res["result"])
for d in res["source_documents"]:
    print("Sumber:", d.metadata)
```

Untuk **CV/gambar**: ekstrak teks dulu dengan `qwen2.5vl:7b` (kirim gambar base64 ke `POST /api/generate`), simpan hasilnya sebagai dokumen teks, baru masuk pipeline yang sama.

---

## 5. Arsitektur inference

### 5.1 Diagram — RAG inference (runtime)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  PC LOKAL (Windows 11)                        │
│                                                                               │
│  ┌────────────┐     1. pertanyaan / upload dokumen                            │
│  │   User     │ ───────────────────────────────────────────┐                  │
│  │ (browser / │                                            ▼                  │
│  │  CLI / app)│                              ┌───────────────────────────┐    │
│  └────────────┘ ◀─── 8. jawaban + sitasi ────│   APP / API LAYER         │    │
│                                              │  (FastAPI / LangChain /   │    │
│                                              │   Open WebUI)             │    │
│                                              └───────────┬───────────────┘    │
│                                                          │                    │
│                       ┌──────────────────────────────────┼─────────────────┐  │
│                       │                                  │                 │  │
│        (jika gambar)  ▼                  2. embed query  ▼   4. context     │  │
│             ┌──────────────────┐         ┌──────────────────┐    + prompt   │  │
│             │  OCR / VISION    │         │  EMBEDDING MODEL │               │  │
│             │  qwen2.5vl:7b    │         │  bge-m3          │               │  │
│             │  (via Ollama)    │         │  (via Ollama)    │               │  │
│             └────────┬─────────┘         └────────┬─────────┘               │  │
│                      │ teks hasil OCR             │ vektor query            │  │
│                      ▼                            ▼                         │  │
│             ┌──────────────────┐         ┌──────────────────┐               │  │
│             │ DOC PROCESSING   │         │  VECTOR STORE    │  3. top-k     │  │
│             │ parse·clean·     │ ──ingest│  Chroma / Qdrant │ ───retrieve   │  │
│             │ chunk            │  vektor▶│  + metadata      │     ─┐        │  │
│             └──────────────────┘         └──────────────────┘      │        │  │
│                      ▲                            │ (3b. re-rank)  │        │  │
│                      │                            ▼                ▼        │  │
│              ┌────────────────┐          ┌──────────────────────────────┐   │  │
│              │  DOC STORE     │          │   LLM (generation)           │   │  │
│              │  pdf·docx·xlsx │          │   qwen2.5:7b  (atau          │   │  │
│              │  ·gambar       │          │   qwen2.5vl:7b utk gambar)   │   │  │
│              └────────────────┘          │   via Ollama  ── GPU 8GB ──  │   │  │
│                                          └───────────┬──────────────────┘   │  │
│                                                      │ 5. teks jawaban      │  │
│                                                      ▼                      │  │
│                                          ┌──────────────────────────────┐   │  │
│                                          │ POST-PROCESS                 │   │  │
│                                          │ grounding check · sitasi ·   │ ──┘  │
│                                          │ format JSON · disclaimer     │  6→7 │
│                                          └──────────────────────────────┘      │
│                                                                                │
│   ┌──────────────────────────────────────────────────────────────────────┐    │
│   │ OLLAMA SERVER  (localhost:11434)  — load/unload model sesuai request   │    │
│   │   • REST API:  /api/generate, /api/chat, /api/embeddings               │    │
│   │   • OpenAI-compatible:  /v1/chat/completions                           │    │
│   │   • Akselerasi CUDA (RTX 4060 Ti) + fallback CPU/RAM untuk layer sisa  │    │
│   └──────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
│   Semua komponen di mesin ini → data tidak keluar (privasi PII terjaga).        │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Yang dibutuhkan untuk inference

**Wajib:**
- **Ollama server** (runtime model + API) — 1 proses, otomatis pakai GPU
- **Model**: 1 LLM generation (`qwen2.5:7b`), 1 embedding (`bge-m3`), + 1 vision (`qwen2.5vl:7b`) bila ada gambar
- **Vector store**: Chroma (embedded, file lokal) atau Qdrant (Docker)
- **App/API layer**: skrip Python (LangChain/LlamaIndex) atau Open WebUI/AnythingLLM
- **Document loaders**: `unstructured`, `PyMuPDF`, `openpyxl` untuk parsing
- **Storage**: folder dokumen sumber + folder index vektor (`./chroma_db`)
- **Resource**: ~8 GB VRAM (model 7B Q4 muat penuh), ~5–10 GB RAM untuk app + overhead, ~30 GB disk

**Opsional / untuk produksi:**
- Re-ranker (cross-encoder, jalan di CPU)
- Antrian/cache (Redis) bila banyak user
- Observability (log query + retrieved chunks) untuk debugging & evaluasi
- Auth & audit log (penting karena data HR/pajak sensitif)
- Scheduler untuk re-index saat dokumen diperbarui

**Pertimbangan kapasitas (8 GB VRAM):**
- Satu model 7B aktif pada satu waktu = aman. Vision 7B = ketat, tutup app berat dulu.
- `bge-m3` kecil — bisa tetap di VRAM bersama model 7B bila masih cukup, jika tidak Ollama swap (sedikit delay tiap ganti).
- Naikkan `num_ctx` hanya seperlunya (context besar makan VRAM/RAM signifikan).
- Untuk throughput tinggi: pakai `gemma3:4b` atau batch embedding.

---

## 6. Skill, MCP, dan Plugin

Tiga cara berbeda untuk **memperluas kemampuan** asisten. Singkatnya:

| | Apa | Untuk apa | Dipakai di mana |
|---|---|---|---|
| **MCP server** | Model Context Protocol — server standar yang mengekspos *tools / resources / prompts* ke klien LLM | Beri model akses ke sistem nyata: filesystem, database HRIS, API pajak, Confluence, Jira | Klien yang dukung MCP (Claude Desktop, Claude Code, beberapa IDE, Open WebUI via bridge) |
| **Tool / function calling** | Fungsi yang dipanggil model saat butuh data/aksi (mekanisme di balik MCP juga) | "Hitung PPh 21", "ambil data karyawan X", "cari dokumen Y" | Langsung via API Ollama (`tools` param) atau lewat LangChain agent |
| **Skill** | Paket instruksi + script + resource yang "diajarkan" ke agent untuk tugas berulang | Workflow domain: "review CV", "buat surat peringatan sesuai template", "rekap pajak bulanan" | Claude Code / agent SDK; di proyek sendiri = kumpulan prompt+script terstruktur |
| **Plugin** | Bundel berisi beberapa skill + MCP + setting, bisa dipasang sekaligus | Distribusi: satu paket "HR Suite" yang isinya semua di atas | Claude Code plugin marketplace; atau modul dalam aplikasi kamu sendiri |

### 6.1 Implementasi MCP server (akses tool nyata)

MCP = protokol terbuka. Kamu tulis server kecil yang mengekspos fungsi; klien LLM otomatis bisa memanggilnya.

Contoh server Python (pakai SDK `mcp`):

```python
# hr_mcp_server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("hr-tools")

@mcp.tool()
def hitung_pph21(gaji_bruto_bulanan: float, status_ptkp: str, jumlah_tanggungan: int) -> dict:
    """Hitung estimasi PPh 21 terutang per bulan (TER bulanan, simplifikasi)."""
    # ... logika perhitungan / panggil API pajak internal ...
    return {"pph21_bulanan": ..., "dasar": ..., "catatan": "estimasi — verifikasi dgn payroll"}

@mcp.tool()
def cari_karyawan(nama_atau_nik: str) -> list[dict]:
    """Cari data karyawan dari HRIS internal (read-only)."""
    return [...]

@mcp.resource("doc://kebijakan/{nama}")
def baca_kebijakan(nama: str) -> str:
    """Ambil isi dokumen kebijakan internal."""
    return open(f"docs/kebijakan/{nama}.md", encoding="utf-8").read()

if __name__ == "__main__":
    mcp.run()   # default: stdio transport
```

Daftarkan di klien (mis. Claude Desktop / Claude Code) — `claude_desktop_config.json` atau `.mcp.json`:

```json
{
  "mcpServers": {
    "hr-tools": { "command": "python", "args": ["C:\\...\\hr_mcp_server.py"] }
  }
}
```

> Ollama sendiri belum jadi "klien MCP". Pola umum: pakai **LangChain/LlamaIndex agent** atau **Open WebUI** sebagai jembatan — agent membaca daftar tool dari MCP server, lalu memanggil `qwen2.5:7b` dengan kemampuan tool-calling-nya. Atau pakai Claude Code/Claude Desktop sebagai klien dan Ollama hanya untuk task lokal tertentu.

### 6.2 Implementasi Skill

Skill = folder berisi `SKILL.md` (instruksi + kapan dipakai) plus script/template pendukung. Contoh struktur:

```
.claude/skills/review-cv/
  SKILL.md            # "Gunakan skill ini saat user minta review/skrining CV..."
  rubrik.md           # kriteria penilaian
  extract.py          # panggil qwen2.5vl untuk OCR + ekstrak ke JSON
  template-feedback.md
```

`SKILL.md` (ringkas):

```markdown
---
name: review-cv
description: Skrining & review CV kandidat — ekstrak data, nilai sesuai rubrik, hasilkan ringkasan + rekomendasi.
---
Langkah:
1. Jalankan `extract.py <file>` untuk OCR (qwen2.5vl) → JSON kandidat.
2. Bandingkan dengan `rubrik.md` dan job requirement yang diberikan user.
3. Keluarkan: ringkasan 5 baris, skor per kriteria, red flags, rekomendasi (lanjut/tidak), dalam Bahasa Indonesia.
4. Jangan menyimpulkan gaji/diskriminatif dari data pribadi.
```

Di Claude Code, skill dipanggil otomatis saat relevan, atau via `/review-cv`. Dalam **aplikasi kamu sendiri**, "skill" = sekadar modul: `prompt template + fungsi orchestrasi` yang dipilih router berdasarkan intent user.

### 6.3 Implementasi Plugin

Plugin = bundel yang dipasang sekaligus, isinya skill + MCP server + perintah + setting. Struktur khas:

```
hr-suite-plugin/
  .claude-plugin/plugin.json     # nama, versi, deskripsi
  skills/
    review-cv/SKILL.md
    surat-peringatan/SKILL.md
    rekap-pajak/SKILL.md
  commands/
    rekap-pajak.md               # slash command
  .mcp.json                      # daftarkan hr_mcp_server.py
  hooks/hooks.json               # opsional: validasi/audit otomatis
```

`plugin.json`:

```json
{
  "name": "hr-suite",
  "version": "0.1.0",
  "description": "Asisten HR, pajak & recruiter — skill + MCP tools internal"
}
```

Install di Claude Code: tambahkan marketplace (`/plugin marketplace add <repo>`) lalu `/plugin install hr-suite`. Untuk aplikasi sendiri: "plugin" = paket Python (entry-point) yang mendaftarkan tool & prompt-nya ke registry app saat startup.

### 6.4 Kapan pakai yang mana

- Butuh model **mengakses data/sistem nyata** (HRIS, API pajak, file) → **MCP server / tools**.
- Butuh **prosedur domain berulang** yang konsisten → **Skill**.
- Mau **mendistribusikan paket lengkap** ke tim → **Plugin**.
- Cuma butuh jawaban berbasis dokumen → cukup **RAG** (bab 4), belum perlu tools.

---

## 7. Roadmap belajar (urutan disarankan)

1. Install Ollama → `ollama run qwen2.5:7b` (rasakan chat lokal).
2. Coba `qwen2.5vl:7b` dengan 1 CV scan → minta output JSON.
3. Bangun RAG minimal (bab 4.3) dengan 1–2 PDF peraturan.
4. Tambah hybrid search + sitasi sumber + disclaimer.
5. Pasang antarmuka (Open WebUI) untuk dipakai sehari-hari.
6. Tambah 1 MCP tool sederhana (mis. `hitung_pph21`).
7. Rapikan jadi skill, lalu bundel jadi plugin bila perlu dibagikan.
8. Buat set evaluasi (~30 Q&A) → ukur faithfulness sebelum dianggap "produksi".

---

## Lisensi & catatan

- Model: `qwen2.5` / `qwen2.5vl` (Apache-2.0), `gemma3` (Gemma Terms of Use — ada batasan pemakaian), `bge-m3` (MIT). Cek lisensi sebelum pemakaian komersial.
- Asisten ini alat bantu draft & analisis — **bukan pengganti** konsultan pajak/hukum/HR profesional. Selalu verifikasi keluaran sebelum dipakai resmi.
