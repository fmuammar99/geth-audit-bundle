PAKET FILE UNTUK OPUS_TASK_GETH_01 (dipakai kalau git clone gagal di Android)

Isi paket ini:
- OPUS_TASK_GETH_01.md : file task (baca ini dulu)
- geth/core/vm/ : seluruh kode EVM geth (gas, instruksi, memory, stack)
- geth/params/ : konstanta protocol (angka gas ada di sini)
- geth/core/types/authorization.go + geth/core/state_processor.go : EIP-7702
- eips/eip-8037.md + eip-8038.md : spesifikasi state gas
- eels/vm/gas.py : implementasi referensi EELS (untuk perbandingan)

Cara pakai di aplikasi Claude Code (Android):
1. Download dan extract zip ini di HP.
2. Lampirkan folder hasil extract (atau file OPUS_TASK_GETH_01.md) ke chat Opus.
3. Paste prompt: "Kerjakan OPUS_TASK_GETH_01.md yang saya lampirkan. Repo sudah
   saya lampirkan sebagai file di folder ini (git clone tidak bisa di sini),
   jadi baca kode dari file-file tersebut. Jawab dalam tabel seperti yang
   diminta di file, dalam Bahasa Indonesia."
