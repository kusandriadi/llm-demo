# Step 4 — Bangun RAG minimal sendiri

**Referensi README:** bab 4.1 (pipeline), 4.3 (contoh kode), 7 (pilihan framework).

## Tujuan
Punya pipeline RAG sederhana yang kamu pahami: ingest dokumen → embedding → vector store → retrieve → generate → jawaban + sumber.

## Prasyarat
- Step 1 selesai (`qwen2.5:7b`, `bge-m3` ada).
- Python ≥ 3.10.
- Folder `docs_sumber/` berisi 1–2 PDF peraturan/kebijakan untuk diuji.

## Pilih framework (bab 7.4)
| Tujuan | Pakai |
|---|---|
| Belajar RAG, kode ringkas, fokus dokumen (**disarankan**) | **LlamaIndex** |
| Paham sampai dasar, dependensi tipis | Tanpa framework: Ollama API + `chromadb` + `unstructured` |
| Nanti perlu agent/banyak tool | LangChain (+ LangGraph) |

## Langkah

1. **Setup environment**
   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   pip install --upgrade pip
   ```

2. **Variasi A — LlamaIndex** (disarankan)
   ```powershell
   pip install llama-index llama-index-llms-ollama llama-index-embeddings-ollama `
               llama-index-vector-stores-chroma chromadb "unstructured[pdf,docx,xlsx]" pymupdf
   ```
   ```python
   # implementation/scripts/rag_llamaindex.py
   from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings, StorageContext
   from llama_index.llms.ollama import Ollama
   from llama_index.embeddings.ollama import OllamaEmbedding
   from llama_index.vector_stores.chroma import ChromaVectorStore
   import chromadb

   Settings.llm = Ollama(model="qwen2.5:7b", temperature=0.2, request_timeout=300)
   Settings.embed_model = OllamaEmbedding(model_name="bge-m3")
   Settings.chunk_size, Settings.chunk_overlap = 800, 120

   client = chromadb.PersistentClient(path="./chroma_db")
   vstore = ChromaVectorStore(chroma_collection=client.get_or_create_collection("hr"))
   storage = StorageContext.from_defaults(vector_store=vstore)

   docs = SimpleDirectoryReader("docs_sumber").load_data()
   index = VectorStoreIndex.from_documents(docs, storage_context=storage)

   qe = index.as_query_engine(similarity_top_k=5)
   res = qe.query("Berapa PTKP untuk wajib pajak kawin dengan 1 tanggungan?")
   print(res)
   for n in res.source_nodes:
       print("Sumber:", n.metadata, "score=", round(n.score, 3))
   ```

3. **Variasi B — LangChain** (lihat README bab 4.3, kode sudah ada di sana)
   ```powershell
   pip install langchain langchain-community langchain-ollama chromadb "unstructured[pdf,docx,xlsx]" pymupdf
   ```

4. **Tambahkan CV/gambar ke index** — pakai `ocr.py` dari step 2: OCR → simpan teks hasilnya sebagai `.txt`/`.md` di `docs_sumber/` → masuk pipeline yang sama.

5. **Uji**: jalankan beberapa pertanyaan; periksa jawaban + `source_nodes` (apakah chunk yang diambil relevan?).

## Verifikasi — selesai bila…
- `python implementation\scripts\rag_llamaindex.py` menghasilkan jawaban + daftar sumber.
- Index tersimpan di `./chroma_db` (tidak perlu re-ingest tiap run).
- Kamu bisa menjelaskan tiap tahap pipeline (ingest → chunk → embed → store → retrieve → generate).

## Catatan
- `bge-m3` & `qwen2.5:7b` bergantian dipakai Ollama; ada sedikit delay saat ganti model (swap VRAM) — wajar.
- Untuk peraturan, hasil lebih baik kalau chunking mengikuti struktur (per pasal/ayat) — akan diperbaiki di step 5.
- Jangan masukkan `f"...{isi_dokumen}..."` mentah ke prompt — gunakan delimiter & tandai untrusted. Detail di [step-9.md](step-9.md).

➡️ Lanjut: [step-5.md](step-5.md)
