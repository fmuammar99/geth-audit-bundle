# OPUS_TASK_GETH_TRIE — Geth: audit penuh Merkle Patricia Trie

Konteks: ini audit penuh `trie/` go-ethereum (lanjutan Geth Task 02, di mana
trie hanya di-spot-check). Kamu mulai dari KOSONG (tidak ada repo lokal).
Target: https://github.com/ethereum/go-ethereum — commit `fac2e99`
("accounts/keystore: fix the race between key scan and delete (#35846)").
Kode yang dibutuhkan sudah ada di repo bundle (folder `geth-task02/geth/`).

## Langkah 0 — sumber yang dipakai

Dari repo bundle:
- `geth-task02/geth/trie/` — seluruh paket trie (node, encoding, hasher,
  committer, proof, iterator, secure_trie, stacktrie, sync, pbt)
- `geth-task02/geth/core/state/` — pemakaian trie oleh StateDB
  (`database_mpt.go`, `statedb.go`)
- `geth-task02/geth/core/state_processor.go` — state root per blok

Bila butuh file lain yang tidak ada di bundle, ambil dari GitHub pada
commit `fac2e99` dan catat file apa yang kamu ambil.

## Yang dicek (8 pemeriksaan)

1. **Encoding node.** `trie/node.go`, `node_enc.go`, `encoding.go`:
   RLP encoding short node / full node / value node, batas 32 byte
   (node kecil di-embed, node besar di-hash), dan decoding-nya. Cari
   kasus di mana encode lalu decode tidak kembali ke node yang sama,
   atau node yang salah lolos decode.
2. **Hashing.** `trie/hasher.go`, `trie.go` (Hash/Commit): dirty
   tracking, cache hash, collapse node. Cari jalur di mana hash tidak
   diperbarui setelah update/delete, atau hash lama dipakai ulang
   setelah perubahan (state root salah).
3. **Committer.** `trie/committer.go`: commit node dirty ke database,
   penanganan node yang dihapus, dan snapshot. Cari jalur di mana node
   yang sudah dihapus tetap tersimpan, atau node baru hilang setelah
   commit (state tidak bisa dibaca kembali).
4. **Proof.** `trie/proof.go`: Prove / VerifyProof / proofToPath,
   proof untuk key yang tidak ada, range proof. Cari proof yang salah
   tapi diterima verifikasi, atau proof yang benar tapi ditolak
   (termasuk proof dengan node yang hilang / prefix bersama).
5. **Iterator.** `trie/iterator.go`, `secure_trie.go`: traversal,
   seek, iterasi prefix, dan perilaku saat node berubah/dihapus di
   tengah iterasi. Cari jalur di mana iterator melewatkan key, membaca
   key dua kali, atau crash pada trie yang baru diubah.
6. **StackTrie.** `trie/stacktrie.go`, `stacktrie_partial.go`: trie
   in-memory untuk membangun blok, dan partial trie. Bandingkan hash
   yang dihasilkan dengan trie penuh untuk input yang sama — cari
   divergensi (mis. pada update/delete berurutan atau key dengan
   prefix panjang).
7. **Path-based trie (pbt/).** `trie/pbt/`: skema penyimpanan baru,
   state root, journal/recovery, dan penyimpanan code. Cari jalur di
   mana state root pbt berbeda dari MPT klasik untuk state yang sama,
   atau data hilang setelah restart/recovery.
8. **Sync.** `trie/sync.go`: sinkronisasi trie dari peer, penanganan
   node yang hilang, dan validasi data sync. Cari jalur di mana data
   sync yang jahat bisa diterima (state root salah) atau membuat sync
   macet (DoS).

## Aturan

- JANGAN menjalankan build/test berat di sesimu (kamu di Android).
  Kalau perlu bukti, tulis perintah `go test` persisnya + file test
  yang relevan — Muse yang menjalankan di VM (geth sudah bisa di-build
  di sana, `go test ./trie/...` jalan).
- Tandai tiap temuan "checked" (kamu baca kodenya langsung) atau
  "inferred" (penalaran dari kode tanpa menjalankan).
- JANGAN klaim "belum pernah dilaporkan" — Muse yang cek duplikat di GitHub.

## Format jawaban (Bahasa Indonesia)

- (A) Kandidat bug — perilaku salah untuk input valid (state root
  salah, proof palsu diterima, data hilang, crash, DoS).
  Tiap kandidat: file+baris, skenario pemicu minimal, dampak.
- (B) Kandidat divergensi — geth vs spesifikasi/EELS (trie heksadesimal
  di EELS) atau vs MPT klasik (untuk pbt).
- (C) Temuan lemah / perlu cek lanjutan.
- 3 kandidat terkuat.
- 3 hal yang sudah dicek dan ternyata aman (dengan file+baris).
