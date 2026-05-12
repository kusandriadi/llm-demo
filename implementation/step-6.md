# Step 6 — Tambah tool via MCP server

**Referensi README:** bab 6.1 (MCP), 6.4 (kapan pakai tool).

## Tujuan
Beri asisten kemampuan memanggil fungsi/aksi nyata (hitung pajak, cari data karyawan, baca dokumen kebijakan) lewat MCP server lokal — dengan least privilege.

## Prasyarat
- Python ≥ 3.10. `pip install mcp` (atau `fastmcp`).
- Klien MCP: Claude Desktop / Claude Code, atau agent LangChain/LlamaIndex yang bisa baca tool MCP.

## Langkah

1. **Tulis MCP server** — `implementation/scripts/hr_mcp_server.py`:
   ```python
   from mcp.server.fastmcp import FastMCP

   mcp = FastMCP("hr-tools")

   @mcp.tool()
   def hitung_pph21(gaji_bruto_bulanan: float, status_ptkp: str, jumlah_tanggungan: int) -> dict:
       """Estimasi PPh 21 terutang per bulan (TER bulanan, simplifikasi). READ-ONLY."""
       # validasi argumen (range wajar) — jangan percaya input mentah
       assert gaji_bruto_bulanan >= 0 and 0 <= jumlah_tanggungan <= 3
       # ... logika perhitungan / panggil API pajak internal ...
       return {"pph21_bulanan": ..., "dasar": ..., "catatan": "estimasi — verifikasi dgn payroll"}

   @mcp.tool()
   def cari_karyawan(nama_atau_nik: str) -> list[dict]:
       """Cari data karyawan dari HRIS internal. READ-ONLY. Hasil sudah di-filter sesuai hak akses."""
       return [...]

   @mcp.resource("doc://kebijakan/{nama}")
   def baca_kebijakan(nama: str) -> str:
       """Ambil isi dokumen kebijakan internal (nama divalidasi, no path traversal)."""
       import re, pathlib
       assert re.fullmatch(r"[a-z0-9_-]+", nama), "nama tidak valid"
       return (pathlib.Path("docs/kebijakan") / f"{nama}.md").read_text(encoding="utf-8")

   if __name__ == "__main__":
       mcp.run()   # stdio transport
   ```

2. **Daftarkan di klien** — `.mcp.json` (Claude Code) atau `claude_desktop_config.json`:
   ```json
   {
     "mcpServers": {
       "hr-tools": { "command": "python", "args": ["C:\\...\\implementation\\scripts\\hr_mcp_server.py"] }
     }
   }
   ```
   (Catatan: Ollama sendiri belum jadi klien MCP — pakai Claude Code/Desktop, atau jembatani lewat agent LangChain/LlamaIndex yang membaca daftar tool lalu memanggil `qwen2.5:7b` dengan kemampuan tool-calling-nya.)

3. **Terapkan least privilege sejak awal** (lihat [step-9.md](step-9.md) §E):
   - Semua tool **read-only** secara default. Aksi yang mengubah data (update status, kirim email) → wajib konfirmasi manusia.
   - Validasi tipe & range tiap argumen; query DB pakai parameter (no string concat).
   - **Jangan** biarkan isi dokumen yang di-retrieve memicu tool otomatis — pemicu hanya dari user.
   - Tambah timeout, rate limit, dan audit log tiap pemanggilan (siapa, kapan, argumen, hasil).

## Verifikasi — selesai bila…
- Klien mendeteksi tool `hitung_pph21`, `cari_karyawan`, resource `doc://kebijakan/...`.
- Saat ditanya "Berapa PPh 21 untuk gaji 15 juta, K/1?", model memanggil `hitung_pph21` dengan argumen benar dan menjelaskan hasilnya + disclaimer.
- Argumen di luar range ditolak rapi (bukan crash).
- Ada baris di audit log untuk tiap pemanggilan tool.

## Catatan
- Mulai dari 1 tool sederhana yang benar-benar berguna; jangan langsung bikin 10 tool.
- Logika `hitung_pph21` jangan ditaruh di prompt model — hitung di kode (deterministik), model hanya memanggil & menjelaskan.

➡️ Lanjut: [step-7.md](step-7.md)
