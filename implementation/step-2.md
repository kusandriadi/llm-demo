# Step 2 — Vision/OCR: ekstrak data dari CV/dokumen gambar → JSON

**Referensi README:** bab 2 (model vision), 3.3, 4.3 (catatan CV/gambar).

## Tujuan
Bisa kasih gambar/scan (CV, slip gaji, KTP, NPWP, formulir) ke `qwen2.5vl:7b` dan dapat output JSON terstruktur.

## Prasyarat
- Step 1 selesai (`qwen2.5vl:7b` sudah di-pull).
- 1–2 file contoh: foto/scan CV (`.png`/`.jpg`) atau PDF hasil scan.

## Langkah

1. **Tes cepat via CLI** — tulis path gambar di dalam prompt:
   ```powershell
   ollama run qwen2.5vl:7b
   >>> Ekstrak nama, email, no HP, pendidikan terakhir, dan daftar skill dari CV ini sebagai JSON. C:\data\cv\contoh.png
   ```

2. **Via API** (PowerShell) — kirim gambar base64 ke `POST /api/generate`:
   ```powershell
   $img = [Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\data\cv\contoh.png"))
   $body = @{
     model  = "qwen2.5vl:7b"
     prompt = "Ekstrak data kandidat sebagai JSON dengan field: nama, email, telepon, pendidikan, pengalaman_tahun (number), skills (array). Jika field tidak terbaca, isi null. Jangan menebak."
     images = @($img)
     stream = $false
     format = "json"
   } | ConvertTo-Json
   Invoke-RestMethod -Uri http://localhost:11434/api/generate -Method Post -Body $body | Select-Object -ExpandProperty response
   ```

3. **(Opsional) Skrip `ocr.py`** — bungkus jadi fungsi reusable (dipakai lagi di step 4 & 6):
   ```python
   # implementation/scripts/ocr.py
   import base64, json, sys, requests

   PROMPT = ("Ekstrak data sebagai JSON: nama, email, telepon, pendidikan, "
             "pengalaman_tahun (number), skills (array). Field tak terbaca = null. Jangan menebak.")

   def ocr_to_json(path: str, model: str = "qwen2.5vl:7b") -> dict:
       img = base64.b64encode(open(path, "rb").read()).decode()
       r = requests.post("http://localhost:11434/api/generate", json={
           "model": model, "prompt": PROMPT, "images": [img],
           "stream": False, "format": "json", "options": {"temperature": 0.1},
       }, timeout=300)
       r.raise_for_status()
       return json.loads(r.json()["response"])

   if __name__ == "__main__":
       print(json.dumps(ocr_to_json(sys.argv[1]), ensure_ascii=False, indent=2))
   ```
   ```powershell
   python implementation\scripts\ocr.py C:\data\cv\contoh.png
   ```

4. **PDF multi-halaman scan** → render tiap halaman jadi PNG dulu, proses per halaman:
   ```python
   import fitz  # PyMuPDF
   doc = fitz.open("scan.pdf")
   for i, page in enumerate(doc):
       page.get_pixmap(dpi=200).save(f"page_{i}.png")
   ```

## Verifikasi — selesai bila…
- Output berupa JSON valid (bukan paragraf), field terisi sesuai isi dokumen, yang tak terbaca = `null`.
- Coba 2–3 dokumen berbeda kualitas; nilai apakah akurasinya cukup. Kalau jelek di scan buram → naikkan resolusi gambar / pakai dokumen lebih jelas / pertimbangkan model lain.

## Catatan
- **Resolusi**: makin tinggi = makin banyak token visual = makin makan VRAM. Untuk dokumen padat teks ~1500–2000 px sisi terpanjang biasanya cukup. Kalau OOM, kecilkan.
- **`temperature` rendah (0.1–0.2)** untuk ekstraksi — biar tidak mengarang angka/nama.
- ⚠️ Jangan langsung percaya digit kritis (NIK, NPWP, nominal) dari scan buram — validasi format / cek manual. Ini juga akan ditangani di [step-9.md](step-9.md) (sanitasi & validasi).
- Hasil OCR ini nantinya jadi input untuk RAG (step 4) dan tool (step 6).

➡️ Lanjut: [step-3.md](step-3.md)
