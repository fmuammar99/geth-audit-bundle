# HASIL OPUS_TASK_GETH_TRIE — Geth: audit Merkle Patricia Trie

Target: go-ethereum commit `fac2e99`. Kode dibaca dari `geth-task02/geth/trie/` (plus `core/state/` untuk pemanggil).
Tidak ada build/test yang dijalankan. Tag: **checked** = kode dibaca langsung; **inferred** = penalaran tanpa menjalankan.
Cek duplikat di GitHub BELUM dilakukan (tugas Muse). Tidak ada file tambahan yang diambil dari GitHub untuk task ini.

## Ringkasan
- (A) Tidak ada bug yang terkonfirmasi memberi state root salah atau proof palsu diterima.
- Kode yang paling baru (partisi StackTrie, `UpdateBatch` paralel, `pbt/`) adalah tempat temuan berada; semuanya kandidat lemah atau bergantung pada pemanggil yang tidak ada di bundle.
- Bagian lama (proof, range proof, committer, decoding) sesuai pola upstream yang sudah lama di-fuzz.
- **Belum dibaca tuntas:** `iterator.go` (traversal/seek) dan `sync.go` di luar `ProcessNode`/`ProcessCode`.

## Hasil 8 pemeriksaan

| # | Pemeriksaan | Hasil | Lokasi | Tag |
|---|---|---|---|---|
| 1 | Encoding node | Decode menolak node embedded ≥ 32 byte, referensi string selain 0/32 byte, dan jumlah elemen selain 2/17. Decoding compact key longgar (bit flag tak terpakai tidak dicek), tetapi node selalu diikat hash, jadi tidak bisa dipalsukan terhadap root nyata. | `trie/node.go:248-345` | checked |
| 2 | Hashing | Cache hash dipakai hanya bila node tidak dirty; insert/delete membuat flag baru. `UpdateBatch` paralel menandai root dirty dan memproses anak root di posisi berbeda; tracer memakai mutex. Lihat C2. | `trie/trie.go:489-559, 806-870`; `trie/tracer.go:36-112` | checked |
| 3 | Committer | Node dihapus dicatat dari `opTracer` lalu difilter dengan `prevalueTracer.HasList` (hanya node yang benar-benar ada di DB). Node embedded yang dulunya tersimpan dicatat sebagai terhapus (`store`, hash == nil). Commit paralel memakai NodeSet per anak lalu `MergeDisjoint`. | `trie/committer.go:28-138`; `trie/trie.go:786-856` | checked |
| 4 | Proof | `VerifyProof`, `proofToPath`, `unsetInternal`, `unset`, `hasRightElement`, `VerifyRangeProof`: urutan monoton, larangan prefix, tanpa nilai kosong, dua edge proof, rebuild + bandingkan root. Ada `panic` untuk node tak terduga (C4) — tidak terjangkau dengan root state yang sah. | `trie/proof.go:111-593` | checked |
| 5 | Iterator | Belum dibaca tuntas. | `trie/iterator.go` | — |
| 6 | StackTrie / partial | Urutan key wajib naik (`stacktrie.go:96`). Root selalu di-hash, node lain di-embed bila < 32 byte (`:418`). `PartialStackTrie` + `MountPartitionRoot`: lihat C1. | `trie/stacktrie.go`; `trie/stacktrie_partial.go`; `trie/node.go:359-401` | checked |
| 7 | pbt | Hash leaf mengikat stem+sub+value penuh; hash branch mengikat prefix ber-panjang (2 byte panjang + bit). Split/collapse menjaga path absolut anak; jalur hapus/insert dilacak di `deletes`. Root kosong = `EmptyBinaryHash` = 0x00…, konsisten dengan `emptyNode.hashAt`. Root pbt memang tidak sama dengan MPT (komitmen berbeda, blake3 + trie biner), jadi perbandingan root pbt vs MPT tidak relevan. Lihat C3, C5. | `trie/pbt/node.go`, `trie/pbt/trie.go`, `trie/pbt/bits.go`, `trie/pbt/key.go` | checked |
| 8 | Sync | `ProcessNode`/`ProcessCode` TIDAK memverifikasi `keccak(data) == hash`; validasi diserahkan ke pemanggil (`eth/protocols/snap`, tidak di bundle). Lihat C6. | `trie/sync.go:400-448` | checked |

## (A) Kandidat bug
Tidak ada yang terkonfirmasi.

## (B) Kandidat divergensi
Tidak ada yang terkonfirmasi untuk MPT vs spesifikasi (trie hex EELS). Untuk pbt tidak ada spesifikasi pembanding: desainnya (code dikunci per code-hash, chunk nol tidak disimpan, blake3) bukan EIP-7864, jadi tidak bisa dibandingkan per byte.

## (C) Temuan lemah / perlu cek lanjutan

| # | Lokasi | Catatan | Dampak bila terpicu | Tag |
|---|---|---|---|---|
| C1 | `trie/stacktrie_partial.go:89-91`; `trie/stacktrie.go:418`; `trie/node.go:365-381` | Root partisi selalu di-hash (aturan root StackTrie), dan `MountPartitionRoot` kasus branch selalu mereferensi via hash. Padahal di trie penuh, node di kedalaman 1 yang encoding-nya < 32 byte harus di-EMBED di branch induk. Aman untuk key 32 byte ter-hash (leaf partisi ≥ 34 byte, branch ≥ 66 byte), tetapi prasyarat ini tidak di-assert. | Bila dipakai untuk trie dengan key pendek & nilai kecil (gaya tx/receipt trie), root hasil partisi ≠ root trie penuh. | inferred |
| C2 | `trie/trie.go:489-559` | `UpdateBatch` paralel: kalau tidak ada perubahan nyata (nilai sama), root tetap ditandai dirty → commit menulis ulang root identik (hanya kerja ekstra). Cek "survivors < 2" konservatif, jadi root tidak pernah runtuh di jalur paralel. Pembaca `t.reader` dipakai dari banyak goroutine; aman bila reader thread-safe (tidak di bundle). | Kinerja; data race bila reader tidak thread-safe. | inferred |
| C3 | `trie/secure_trie.go:202, 214-234` vs `trie/pbt/trie.go:143-145, 450-465`; kontrak di `core/state/database.go:112-119` | Dokumentasi interface: `UpdateStorage` dengan nilai panjang 0 = hapus. `StateTrie` meng-RLP-encode nilai kosong menjadi `0x80` dan MENULIS-nya (juga di `UpdateStorageBatch`); `pbt` memperlakukan nilai nol sebagai hapus. StateDB saat ini hanya memanggil `UpdateStorage` untuk nilai ≠ 0 dan `DeleteStorage` untuk nol (`state_object.go:361-378`), dan `UpdateStorageBatch` tidak punya pemanggil di bundle. | Bila pemanggil baru memakai batch dengan nilai kosong: MPT menulis leaf `0x80` (root salah), pbt menghapus → divergensi antar backend. | checked (kode), inferred (dampak) |
| C4 | `trie/proof.go:352-411` | `unset` dengan `key[pos] == 16` (key yang berakhir di slot nilai branch) mengosongkan anak 0..15 lalu turun ke `valueNode` → `panic("it shouldn't happen")`. Hanya mungkin dengan key panjang bervariasi (key satu adalah prefix key lain); trie state memakai key 32 byte, dan node tidak bisa dipalsukan terhadap root sah. | Crash (DoS) hanya bila API dipakai untuk trie dengan key bervariasi. | inferred |
| C5 | `trie/pbt/trie.go:475-505` | `UpdateContractCode` dengan code kosong langsung `return nil`, sehingga leaf codeHash/delegasi tidak dihapus di sini. Ini bergantung pada `UpdateAccount` (CodeHash kosong → menulis EmptyCodeHash dan menghapus leaf delegasi) yang dipanggil untuk akun yang sama. Pembersihan delegasi 7702 dan EIP-8246 (code dihapus) bergantung pada urutan/keberadaan panggilan itu di `core/state` (jalur pbt tidak saya telusuri). | Akun tetap terlihat ter-delegasi / punya code bila `UpdateAccount` tidak dipanggil. | inferred |
| C6 | `trie/sync.go:400-448` | Tidak ada cek `keccak256(result.Data) == req.hash`; `decodeNode` hanya menempelkan hash yang diminta ke node. Keamanan sepenuhnya bergantung pada pemanggil (snap/downloader) yang memeriksa hash sebelum memanggil. | Bila ada pemanggil yang lupa memeriksa: node palsu diterima dan disimpan (state rusak). | checked |

## 3 kandidat terkuat
(Semuanya lemah; tidak ada yang terkonfirmasi.)
1. **C3** — kontrak "nilai kosong = hapus" tidak dipenuhi `StateTrie.UpdateStorage/UpdateStorageBatch` (`secure_trie.go:202, 227`), berbeda dengan pbt.
2. **C1** — partisi StackTrie selalu mereferensi root partisi via hash (`stacktrie_partial.go:89`, `node.go:372`), salah bila node < 32 byte.
3. **C6** — `Sync.ProcessNode` tidak memverifikasi hash (`sync.go:419-431`).

## 3 hal yang sudah dicek dan aman
1. **Decode referensi node**: `trie/node.go:226-248` menolak embedded node ≥ 32 byte dan referensi string selain 0/32 byte.
2. **Range proof**: `trie/proof.go:478-593` — monoton, tanpa prefix, tanpa nilai kosong, panjang edge key sama, rebuild trie dari dua edge proof lalu bandingkan root; `unsetInternal` menolak rentang kosong (`:285-290`).
3. **Commit node terhapus**: `trie/trie.go:789-801` hanya melaporkan path yang benar-benar ada di DB (via `prevalueTracer.HasList`), dan `committer.go:112-118` mencatat node yang jadi embedded sebagai terhapus, sehingga tidak ada node basi yang tertinggal di skema path.

## Perintah untuk Muse (di VM)
```
cd geth
go test ./trie/... -count=1
go test ./trie/ -run 'StackTrie|Partial|Proof|Range|UpdateBatch' -count=1 -v
go test ./trie/pbt/ -count=1 -v
```
Usulan test baru (perlu ditulis):
- **C1**: bangun trie dengan key 1–2 byte dan nilai 1 byte lewat `PartialStackTrie` per nibble + `AssembleBranch`/`MountPartitionRoot`, bandingkan dengan `NewStackTrie` biasa pada data sama (`trie/stacktrie_partial_test.go`).
- **C3**: panggil `StateTrie.UpdateStorageBatch` dengan satu nilai kosong lalu bandingkan root dengan `DeleteStorage` (`trie/secure_trie_test.go`).
- **C4**: `VerifyRangeProof` pada trie dengan key `0x01` dan `0x0102` (key prefix) sebagai edge kanan; pastikan tidak panic (`trie/proof_test.go`).

## Yang perlu dilakukan agent berikutnya
1. Audit `trie/iterator.go` (seek, iterasi saat trie berubah) — belum dicek.
2. Cari pemanggil `NewPartialStackTrie`/`MountPartitionRoot`/`UpdateStorageBatch` di `fac2e99` (di luar bundle) untuk menilai C1 dan C3.
3. Konfirmasi bahwa `eth/protocols/snap` memverifikasi hash sebelum `ProcessNode`/`ProcessCode` (C6).
4. Cek duplikat di issue/PR go-ethereum.
