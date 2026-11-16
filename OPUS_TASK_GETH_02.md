# OPUS_TASK_GETH_02 — Geth: txpool & state/trie

Konteks: ini task kedua audit Geth. Kamu mulai dari KOSONG (tidak ada repo lokal).
Target: https://github.com/ethereum/go-ethereum — commit `fac2e99`
("accounts/keystore: fix the race between key scan and delete (#35846)").
Repo bundle yang dihubungkan berisi kode yang dibutuhkan (lihat di bawah).

## Langkah 0 — sumber yang dipakai

Dari repo bundle (folder `geth-task02/`):
- `geth/core/txpool/` — validasi tx, legacypool, blobpool, subpool
- `geth/core/state/` — StateDB, snapshot/revert, journal
- `geth/trie/` — Merkle Patricia Trie, proof, iterator
- `geth/core/types/` — tipe transaksi (legacy, 1559, blob, setcode)
- `geth/core/state_processor.go` — pemrosesan state per blok
- `geth/core/state_transition.go` — eksekusi transaksi (sudah ada di bundle Task 01)

BACA DULU file-file itu. Bila butuh file lain yang tidak ada di bundle,
ambil dari GitHub pada commit `fac2e99` dan catat file apa yang kamu ambil.

## Yang dicek (8 pemeriksaan)

1. **Validasi transaksi.** `core/txpool/validation.go`: intrinsic gas,
   nonce, balance, gas price / fee untuk semua tipe tx (legacy, 1559,
   blob 4844, setcode 7702). Cari cek yang hilang, salah, atau tidak
   konsisten antar tipe tx (mis. tx yang lolos validasi tapi gagal
   dieksekusi, atau sebaliknya).
2. **Txpool: admission & eviction.** `core/txpool/legacypool/`,
   `blobpool/`, `subpool.go`: batas jumlah tx, price bumping saat
   mengganti tx, eviction saat pool penuh. Cari jalur di mana penyerang
   bisa membanjiri pool, mengusir tx orang lain dengan murah, atau
   membuat pool macet (DoS).
3. **Nonce.** Penanganan nonce gap (tx masa depan), penggantian tx
   dengan nonce sama (gas lebih tinggi), dan urutan eksekusi. Cari
   kasus di mana tx terjebak selamanya, atau tx lama bisa dieksekusi
   dua kali.
4. **StateDB: snapshot & revert.** `core/state/statedb.go`, `journal.go`,
   `state_object.go`: snapshot, revert ke snapshot, dirty tracking,
   penghapusan akun kosong (EIP-161), SELFDESTRUCT. Cari jalur di mana
   revert tidak mengembalikan state sepenuhnya, atau perubahan bocor
   ke luar frame.
5. **Trie.** `trie/`: encoding node, hashing, proof (`proof.go`),
   iterator, path compression. Cari bug yang membuat state root salah,
   proof tidak valid, atau node hilang/rusak setelah update/delete.
6. **Pemrosesan blok.** `core/state_processor.go`: state root akhir,
   receipts, gas used, dan penerapan BAL (EIP-7928, lihat
   `core/state/statedb_eip_7928.go` dan `reader_eip_7928.go`). Cari
   ketidakcocokan antara state yang di-commit dan yang dihitung.
7. **Blob tx (EIP-4844).** `core/types/tx_blob.go`, `core/txpool/blobpool/`:
   blob gas price, batas jumlah blob, validasi sidecar. Cari jalur
   di mana blob bisa masuk tanpa bayar yang seharusnya, atau blob
   valid ditolak.
8. **Divergensi vs EELS.** Untuk tiap area di atas, bandingkan dengan
   perilaku yang diharapkan dari spesifikasi (EIP terkait / EELS).
   Catat perbedaan yang kamu temukan, sekecil apa pun.

## Aturan

- JANGAN menjalankan build/test berat di sesimu (kamu di Android).
  Kalau perlu bukti, tulis perintah `go test` persisnya + file test
  yang relevan — Muse yang menjalankan di VM (geth sudah bisa di-build
  di sana, `go test` jalan).
- Tandai tiap temuan "checked" (kamu baca kodenya langsung) atau
  "inferred" (penalaran dari kode tanpa menjalankan).
- JANGAN klaim "belum pernah dilaporkan" — Muse yang cek duplikat di GitHub.

## Format jawaban (Bahasa Indonesia)

- (A) Kandidat bug — perilaku salah untuk input valid (dana bisa
  dicuri / terkunci, state root salah, DoS, konsensus split).
  Tiap kandidat: file+baris, skenario pemicu minimal, dampak.
- (B) Kandidat divergensi — geth vs spesifikasi/EELS.
- (C) Temuan lemah / perlu cek lanjutan.
- 3 kandidat terkuat.
- 3 hal yang sudah dicek dan ternyata aman (dengan file+baris).
