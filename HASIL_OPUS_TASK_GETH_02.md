# HASIL OPUS_TASK_GETH_02 — Geth: txpool & state/trie

Target: go-ethereum commit `fac2e99`. Kode dibaca dari `geth-task02/geth/` di repo bundle.
Tidak ada build/test yang dijalankan. Tag: **checked** = kode dibaca langsung; **inferred** = penalaran tanpa menjalankan.
Cek duplikat di GitHub BELUM dilakukan (tugas Muse).

## Sumber
- Dari bundle: `core/txpool/` (validation, legacypool, blobpool), `core/state/` (statedb, journal, state_object, statedb_eip_7928, reader_eip_7928), `core/state_processor.go`, `core/types/tx_blob.go`.
- Diambil dari GitHub di `fac2e99` (tidak ada di bundle): `core/state_processor_parallel.go` — jalur eksekusi paralel berbasis BAL yang dipanggil dari `Process()`.
- **Tidak dicek mendalam:** `trie/` (encoding, proof, iterator) dan validasi header blob gas. Saya tidak menemukan apa-apa di sana karena memang belum dibaca tuntas, bukan karena aman.

## Ringkasan
- (A) 1 kandidat DoS txpool: pool penuh membuang tx orang lain sebelum mengecek price-bump penggantian (legacypool).
- (B) 1 kandidat divergensi: EIP-8246 SELFDESTRUCT dengan saldo > 0 tidak menghapus storage awal-blok (hanya di kasus pinggir akun "storage-only").
- (C) 6 temuan lemah.

## Hasil 8 pemeriksaan

| # | Pemeriksaan | Hasil | Lokasi | Tag |
|---|---|---|---|---|
| 1 | Validasi tx | Cek tipe per fork, ukuran, initcode, cap gas (Osaka 2^24; Amsterdam 2^32−1), intrinsic (EIP-2780, dengan `from`/`value`), floor, `max(intrinsic, floor) ≤ 2^24` di Amsterdam, tip ≥ minimum, setcode ≥1 auth, nonce ≠ MaxUint64. Konsisten dengan `state_transition.go`. Celah kecil: tx ber-value ke akun kosong lolos pool walau gas tidak cukup untuk 183.600 state gas — tx tetap valid (OOG saat eksekusi, tetap bayar), jadi bukan bug. | `core/txpool/validation.go:63-172` | checked |
| 2 | Admission & eviction | Legacypool: lihat A1. Blobpool: eviction berbasis `Datacap` + heap prioritas, batas 16 tx/akun, buffer gapped global 128. | `legacypool.go:660-791`; `blobpool.go:2100-2290` | checked |
| 3 | Nonce | Tx ber-gap masuk queue (legacy) atau buffer gapped (blob). Penggantian butuh bump ≥ `PriceBump` untuk fee cap DAN tip (blob juga blob fee cap). Akun ter-delegasi 7702 dibatasi 1 tx in-flight. Tidak menemukan jalur tx dieksekusi dua kali; nonce dicek ulang terhadap state. | `legacypool.go:597-657, 795-819`; `list.go:303-343`; `blobpool.go:1648-1761` | checked |
| 4 | StateDB snapshot/revert | Semua entri journal punya revert yang mengembalikan nilai lama (balance, nonce, code, storage dengan origin, transient, refund, log, access list). `getStateObject` menghormati `stateObjectsDestruct`. Lihat B1 untuk EIP-8246. | `core/state/journal.go:466-673`; `statedb.go:601-613, 852-910` | checked |
| 5 | Trie | Tidak dicek mendalam. | `trie/` | — |
| 6 | Pemrosesan blok / BAL | Sekuensial dan paralel memakai cek gas 2D yang sama (`CheckGasAmsterdam` lalu `ChargeGasAmsterdam`) dan `CumulativeGasUsed` dari pool global. Jalur paralel membaca state lewat overlay BAL dari header; kebenarannya bergantung pada pembandingan BAL hasil eksekusi dengan BAL header (validator, tidak ada di bundle). `ApplyBlockAccessList` menghapus akun yang jadi kosong dan hanya men-stage slot yang berubah. | `core/state_processor.go:68-168`; `core/state_processor_parallel.go:49-69, 210-300, 384-490`; `core/state/statedb_eip_7928.go:37-211`; `reader_eip_7928.go:226-303` | checked (validator: tidak dicek) |
| 7 | Blob tx | Blob fee cap ≥ 1, hash commitment cocok, jumlah ≤ `BlobTxMaxBlobs`, sidecar versi 1, jumlah proof = commitment × `CellProofsPerBlob`, verifikasi KZG per cell. Tx tanpa blob ditolak di `ValidateCells` ("blobless") dan di `state_transition.preCheck`. `tx.Cost()` memasukkan blob gas × blob fee cap untuk cek saldo. | `core/txpool/validation.go:177-238`; `blobpool.go:2059-2079`; `core/types/tx_blob.go:306` | checked |
| 8 | Divergensi vs spek/EELS | B1 (EIP-8246 storage). Tidak menemukan divergensi lain di area yang dibaca. | — | checked/inferred |

## (A) Kandidat bug

### A1 — Pool penuh: tx lain dibuang dulu, cek price-bump penggantian belakangan (eviction tanpa admisi)
- File: `core/txpool/legacypool/legacypool.go:700-760` (discard) vs `:761-767` dan `:785` (cek penggantian di pending / queue).
- Masalah: saat pool penuh, `add()` hanya mengecek `priced.Underpriced(tx)` (lebih murah dari tx termurah di pool), lalu langsung `Discard` + `removeTx` tx termurah milik orang lain. Baru setelah itu `list.Add(tx, PriceBump)` mengecek bump terhadap tx lama dengan nonce sama; bila gagal → `ErrReplaceUnderpriced`, tetapi tx korban sudah dibuang. Hal sama untuk penggantian di queue (`enqueueTx` → `queue.add` → `ErrReplaceUnderpriced`, `queue.go:133-137`).
- Skenario pemicu minimal: pool penuh. Penyerang punya 1 tx pending (nonce N, harga P). Ia mengirim berulang tx nonce N dengan harga sedikit di atas tx termurah pool tetapi < P×1,1 (atau fee cap/tip ≤ P). Setiap kiriman membuang tx termurah orang lain lalu ditolak; penyerang tidak membayar apa pun dan tx aslinya tetap.
- Pembatas: `changesSinceReorg ≤ GlobalSlots/4` per siklus reorg (`legacypool.go:712-715`), jadi laju pembuangan dibatasi per siklus, tetapi siklus berulang terus.
- Dampak: DoS murah terhadap tx berharga rendah milik orang lain (dibuang dari pool tanpa penyerang membayar). Tidak ada dana hilang; tx korban bisa dikirim ulang.
- Tag: checked (kode), inferred (dampak). Pola ini mungkin sudah lama ada (Muse cek duplikat).
- Usulan uji (tulis di `core/txpool/legacypool/legacypool_test.go`): isi pool sampai `GlobalSlots+GlobalQueue` dengan tx murah dari banyak akun; tambahkan tx pending penyerang dengan harga tinggi; kirim pengganti dengan harga di antara keduanya; pastikan `ErrReplaceUnderpriced` dan hitung tx korban yang hilang.
  ```
  cd geth && go test ./core/txpool/legacypool/ -run TestReplacementEvictsOthers -count=1 -v
  ```
  (nama test baru; perlu ditulis Muse.)

## (B) Kandidat divergensi geth vs spesifikasi/EELS

### B1 — EIP-8246: SELFDESTRUCT dengan saldo > 0 mempertahankan storage root awal-blok
- File: `core/state/statedb.go:870-886` (`finaliseAmsterdam`).
- Masalah: untuk akun yang di-selfdestruct dengan saldo > 0, geth membuat objek baru `newObject(s, addr, obj.origin)` lalu hanya mengganti saldo. `origin` adalah akun di awal blok (diperbarui saat commit, `state_object.go:464-478`), jadi `Root` storage awal-blok ikut terbawa dan alamat tidak dimasukkan ke `stateObjectsDestruct`. EIP-8246 menyatakan "all storage is cleared". Cabang saldo = 0 menghapus akun dan storage dengan benar (`:883-886`), jadi geth sendiri tidak konsisten.
- Kapan terjadi: hanya bila alamat punya storage di awal blok tetapi nonce 0 dan code kosong (akun "storage-only"), lalu dibuat lewat CREATE/CREATE2 dan initcode-nya SELFDESTRUCT dengan saldo > 0. Di mainnet butuh tabrakan alamat (praktis mustahil); relevan untuk genesis devnet / fixture EEST. Akar masalahnya sama dengan B1 di laporan GETH_01 (storage lama tidak dihapus saat CREATE).
- Dampak: state root beda dari EELS untuk kasus itu.
- Tag: checked (kode), inferred (dampak).
- Usulan uji (di `core/state/statedb_test.go` atau blockchain test Amsterdam): pre-state akun X dengan storage `{1:1}`, nonce 0, code kosong; tx CREATE2 ke X dengan endowment > 0 dan initcode `SELFDESTRUCT(X)`; cek `GetState(X, 1)` setelah `IntermediateRoot` → seharusnya 0.
  ```
  cd geth && go test ./core/state/ -run TestEIP8246SelfdestructClearsOriginStorage -count=1 -v
  ```
  (nama test baru; perlu ditulis Muse.)

## (C) Temuan lemah / perlu cek lanjutan

| # | Lokasi | Catatan | Tag |
|---|---|---|---|
| C1 | `core/txpool/validation.go:283-295`; `blobpool.go:2112-2124` | Urutan cek: nonce-gap (`ErrNonceTooHigh`) dikembalikan SEBELUM cek saldo. Blobpool menyimpan tx ber-gap ke buffer tanpa cek saldo/delegasi (dicek ulang saat promosi). | checked |
| C2 | `blobpool.go:2118, 2584-2592` | Buffer gapped global hanya 128 dan penuh = tx gapped baru ditolak. Allowance = log10(nonce+1), jadi akun dengan nonce ≥ 9 dan saldo 0 bisa mengisi buffer; entri kedaluwarsa setelah 1 menit lalu bisa diisi ulang. Biaya penyerang: blob + bukti KZG valid per tx (biaya CPU juga di korban). Dampak: tx blob gapped milik orang lain ditolak sementara. | inferred |
| C3 | `blobpool.go:2265-2283` | Setelah loop eviction `drop()`, tx yang baru ditambahkan bisa ikut tergusur, tetapi tetap diumumkan (`discoverFeed`/`insertFeed`) bila memenuhi syarat. Hanya mengumumkan tx yang sudah tidak ada. | inferred |
| C4 | `core/state_processor_parallel.go:384-490`; `reader_eip_7928.go:270-277` | Overlay BAL membaca slot yang tidak tercatat di BAL dari state awal-blok. Untuk akun yang storage-nya dihapus di tengah blok (EIP-8246) dan alamatnya punya storage awal-blok, hasil paralel bisa berbeda dari sekuensial. Kasus pinggir yang sama dengan B1. | inferred |
| C5 | `core/txpool/validation.go:177-199` | `ValidateTransaction` sendiri tidak menolak tx blob tanpa blob; penolakan baru terjadi di `ValidateCells` / `preCheck`. Aman sekarang, tetapi bergantung pada urutan pemanggil. | checked |
| C6 | `trie/`, validator BAL | Belum dibaca. Pembandingan BAL hasil eksekusi vs header ada di `core/block_validator.go` (tidak di bundle) — itu satu-satunya hal yang membuat jalur paralel aman terhadap BAL palsu. Layak dicek. | — |

## 3 kandidat terkuat
1. **A1** — eviction tanpa admisi di legacypool (`legacypool.go:700-767`).
2. **B1** — EIP-8246 dengan saldo > 0 tidak menghapus storage awal-blok (`statedb.go:870-886`).
3. **C2** — buffer gapped blobpool bisa diisi akun bersaldo nol (`blobpool.go:2118, 2584-2592`).

## 3 hal yang sudah dicek dan aman
1. **Revert journal lengkap**: `core/state/journal.go:466-673` — tiap entri (balance, nonce, code, storage + origin, transient, refund, log, access list) dikembalikan; `getStateObject` tidak membaca ulang akun yang sudah dihancurkan di blok ini (`statedb.go:610-613`).
2. **Price bump penggantian**: legacypool `list.go:303-325` dan blobpool `blobpool.go:1728-1758` mewajibkan fee cap DAN tip (dan blob fee cap) naik ≥ `PriceBump`, dengan cek "lebih besar" untuk harga level wei.
3. **Cek gas blok 2D sama di sekuensial dan paralel**: `core/state_processor_parallel.go:246-255` memakai `CheckGasAmsterdam(min(gas, MaxTxGas), gas)` lalu `ChargeGasAmsterdam`, sama dengan `preCheck`/`settleGas` di jalur sekuensial; worker berhenti bila total melampaui gas limit (`:417-420`).

## Perintah untuk Muse (di VM)
```
cd geth
go test ./core/txpool/... -count=1
go test ./core/state/ -run 'Journal|Revert|Snapshot|BAL|7928' -count=1 -v
go test ./core/ -run 'Parallel|BAL|Amsterdam' -count=1 -v
```
Nama di regex `-run` adalah tebakan; sesuaikan dengan nama test yang ada.

## Yang perlu dilakukan agent berikutnya
1. Tulis dan jalankan test A1 dan B1 di atas.
2. Baca `core/block_validator.go` di `fac2e99` untuk memastikan BAL hasil eksekusi dibandingkan dengan BAL header (C6).
3. Audit `trie/` (proof, iterator, path compression) — belum dicek.
4. Cek duplikat A1, B1, C2 di issue/PR go-ethereum.
