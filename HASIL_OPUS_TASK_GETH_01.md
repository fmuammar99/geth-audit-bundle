# HASIL OPUS_TASK_GETH_01 — Geth EVM: gas & state gas Amsterdam (EIP-8037/8038)

Target: go-ethereum commit `fac2e99`. Semua temuan dari membaca kode; TIDAK ada test/build yang dijalankan.
Tag: **checked** = kode dibaca langsung; **inferred** = penalaran dari kode tanpa dijalankan.
Cek duplikat di GitHub BELUM dilakukan (tugas Muse).

## Sumber yang dipakai
- Dari bundle: `geth/core/vm/*`, `geth/params/*`, `geth/core/state_processor.go`, `eips/eip-8037.md`, `eips/eip-8038.md`, `eels/vm/gas.py`.
- TIDAK ada di bundle, diambil dari GitHub pada commit `fac2e99`: `core/state_transition.go`, `core/state/statedb.go`, `core/state/state_object.go`.
  - `core/types/authorization.go` tidak ada di commit itu (404); struct 7702 ada di `core/types/tx_setcode.go`.
- EELS (execution-specs HEAD, fork amsterdam): `vm/instructions/system.py`, `vm/instructions/storage.py`, `vm/interpreter.py`, `fork.py`, `vm/eoa_delegation.py`, `state_tracker.py`. `gas.py` di HEAD identik dengan `eels/vm/gas.py` di bundle.

## Ringkasan
- (A) Tidak ada selisih gas geth-vs-EIP yang terkonfirmasi.
- (B) 2 kandidat divergensi geth-vs-EELS, hanya di kasus pinggir (praktis mustahil di mainnet; bisa muncul di fixture test / genesis devnet).

## Hasil 8 pemeriksaan

| # | Pemeriksaan | Hasil | Lokasi | Tag |
|---|---|---|---|---|
| 1 | Konstanta | Semua cocok: CPSB 1530; 120/64/23 byte → 183.600 / 97.920 / 35.190; COLD_ACCOUNT 3000; ACCOUNT_WRITE 9000; CALL_VALUE 11.300; STORAGE_WRITE 10.000; clear refund 11.616; CREATE_ACCESS 12.000; access list 2900/2000; ExecutionPerAuthBaseCost 7816 (=1616+3000+3000+200); MaxTxTotalGas 2^32−1. SSTORE slot baru cold = 2100+10.000 eksekusi + 97.920 state = 110.020. Tidak ada nilai lama di jalur Amsterdam. | `params/protocol_params.go:31-32,107,113-124,222-226` | checked |
| 2 | Reservoir | Ada di `core/vm/gascosts.go` (bukan `gas.go`). Reservoir hanya bila `tx.gas > 2^24` (`state_transition.go:535`). Refill LIFO `min(s, Spilled)` (`gascosts.go:157`). Merge: `d = min(reservoir, spill)` (`gascosts.go:286`). Rollback tidak pakai baseline tersimpan tapi rumus `StateGas + UsedStateGas − Spilled` (`gascosts.go:210,233`); sama dengan baseline EELS karena invarian R+U−Sp = R₀ dijaga. | — | checked (invarian: inferred) |
| 3 | SSTORE | 13 baris tabel EIP-8038/8037 cocok dengan EELS `storage.py`. Sentry `gas_left ≤ 2300` hanya gas eksekusi (sesuai spek). Refill sesudah charge (EELS: sebelum) — tanpa dampak karena akses 100 sudah dijamin sentry. | `core/vm/gas_table.go:680-741` | checked |
| 4 | CREATE/CREATE2 | Konstan 12.000; charge 183.600 bersyarat (precheck → `Empty`) sebelum 63/64; code deposit: cek ukuran → 6/word eksekusi → 1530/byte state; initcode ≤ 131.072. Urutan sama dengan EELS `generic_create`/`process_create`. | `core/vm/evm.go:550-568,664-677`; `core/vm/instructions.go:642-680` | checked |
| 5 | CALL family | Charge akun baru hanya CALL ber-value ke akun `Empty`, sesudah gas eksekusi dan sebelum `callGas` (63/64 dari gas sisa setelah spill) — sama dengan EELS. CALLCODE/DELEGATECALL/STATICCALL tanpa state gas. Refill saat gagal/precheck ditolak. Tidak ada jalur lolos tanpa charge. | `core/vm/operations_acl.go:471-551`; `core/vm/instructions.go:775` | checked |
| 6 | Refund cap | Cap = `gasUsedBeforeRefund/5` (eksekusi + state). Sumber refund Amsterdam hanya SSTORE; SELFDESTRUCT tanpa refund. Konsisten. | `core/state_transition.go:1224-1233` | checked |
| 7 | EIP-7702 | chain_id, nonce, tanda tangan, code divalidasi sebelum charge (`:1098`). Otorisasi gagal hanya bayar intrinsic 7816, tanpa state 35.190. 35.190 sekali per authority, hanya bila belum ter-delegasi di awal tx. OOG → semua otorisasi di-revert + halt (setara `restore_state_gas_to_entry` EELS). | `core/state_transition.go:1136-1210` | checked |
| 8 | SYSTEM_MAX_SSTORES = 16 | Bukan pembatas, hanya bekal reservoir: 30M eksekusi + 16×97.920 = 1.566.720 state. Store ke-17 tetap lolos, diambil dari 30M eksekusi (spill). Sesuai EIP ("margin") dan EELS. | `core/state_processor.go:358-370`; EELS `fork.py:756-761` | checked |

## (A) Kandidat bug — salah hitung gas geth vs EIP
Tidak ada yang terkonfirmasi.

## (B) Kandidat consensus split / divergensi

### B1 — Storage lama tidak dihapus saat CREATE ke alamat "storage-only"
- File: `core/vm/evm.go:612-619` (fungsi `create`), `core/state/statedb.go:676` (`CreateContract`).
- Masalah: alamat dengan storage tapi nonce 0 dan code kosong. Geth hanya memanggil `CreateAccount` bila `!Exist(address)`; `CreateContract` hanya set flag `newContract` → storage lama tetap. EELS menjalankan `destroy_storage` (`interpreter.py:350`). Cek tabrakan di kedua klien hanya nonce/code (cek storage EIP-7610 sudah dihapus). Cek tabrakan geth tidak di-gate per fork → berlaku juga pra-Amsterdam (EELS Osaka juga `destroy_storage`).
- Dampak: state root beda; SLOAD di initcode → nilai lama (geth) vs 0 (EELS); gas SSTORE beda (original ≠ 0 → tanpa 97.920 state gas). Mainnet butuh tabrakan alamat (praktis mustahil); relevan untuk genesis devnet / fixture EEST.
- Cara memicu: genesis berisi akun X = `CreateAddress2(D, salt, keccak(init))` dengan storage `{0:1}`, nonce 0, code kosong, balance 0. Tx memanggil kontrak D yang menjalankan `CREATE2(salt, init)`, init = `SLOAD(0)` lalu `SSTORE(1, hasil)`. Bandingkan post-state geth vs EELS.
- Tag: checked (kode), inferred (dampak).

### B2 — Charge NEW_ACCOUNT authority 7702: `Empty` vs `account_exists`
- File: `core/state_transition.go:1180`.
- Masalah: geth pakai `st.state.Empty(authority)` (tidak ada ATAU kosong per EIP-161). EELS pakai `not account_exists(...)` (`eoa_delegation.py:295`), yang hanya mengecek ada/tidaknya leaf. (EELS sendiri memakai `is_account_alive` untuk CALL/CREATE/SELFDESTRUCT.)
- Dampak: authority yang ada di trie tapi kosong → geth menagih 183.600 state gas lebih → gasUsed/receipt/state beda. Teks EIP condong ke geth, tapi tetap divergensi. Di mainnet akun kosong sudah dibersihkan (EIP-161); realistisnya di fixture/genesis.
- Cara memicu: genesis dengan akun A `{balance 0, nonce 0, code ""}`; kirim SetCode tx dengan otorisasi valid dari A; bandingkan gasUsed.
- Tag: checked (kode), inferred (dampak).

## (C) Temuan lemah / perlu cek lanjutan

| # | Lokasi | Catatan | Tag |
|---|---|---|---|
| C1 | `core/state_transition.go:1033` vs EELS `gas.py:1216` | `UsedStateGas < 0` → geth return error (blok invalid); EELS clamp ke 0. Tidak ditemukan jalur yang membuat negatif; bila ada → split langsung. | inferred |
| C2 | `core/vm/gascosts.go:210-214, 233-237` | Reservoir negatif di-clamp 0 hanya dengan `log.Warn`; rollback pakai rumus, bukan baseline tersimpan. Mutasi `StateGas` tanpa `UsedStateGas` di masa depan → divergensi diam-diam. | checked |
| C3 | `core/vm/instructions.go:775`; `core/state_transition.go:910` | Refill CALL menghitung ulang `Empty(toAddr)` setelah call; EELS pakai flag `new_account_charged`. Sekarang setara (revert snapshot memulihkan state), tapi rapuh. | inferred |
| C4 | `core/state_transition.go:826-856, 874-899` | Spill charge penerima/CREATE tingkat tx disimpan di budget tx, bukan frame teratas (EELS: di `state_gas_spilled` meter teratas). Tidak ditemukan skenario di mana `repay_state_gas_spill` berbeda; layak di-fuzz. | inferred |
| C5 | `core/state_transition.go:1184-1187` | AUTH_BASE 35.190 tidak dikembalikan bila delegasi di-set lalu di-clear dalam tx yang sama. Sama dengan EELS; EIP-2780 tidak tersedia untuk verifikasi. | checked (vs EELS) |
| C6 | `core/state_processor.go:366` | `msg.GasLimit` system call = 30M, bukan 30M + 16×97.920 seperti rumus EIP. EELS juga 30M; tidak teramati dari opcode. Rendah. | checked |

## 3 kandidat terkuat
1. B1 — storage lama tidak dihapus saat CREATE ke alamat storage-only (`core/vm/evm.go:612-619`).
2. B2 — `Empty` vs `account_exists` untuk NEW_ACCOUNT authority 7702 (`core/state_transition.go:1180`).
3. C1 — hard error vs clamp untuk `UsedStateGas` negatif (`core/state_transition.go:1033`).

## 3 hal yang sudah dicek dan aman
1. Tabel SSTORE `core/vm/gas_table.go:680-741` identik dengan EELS `storage.py:68-158` (charge 97.920 hanya 0→0→x, refill 0→x→0, refund 11.616 / +10.000, `SubRefund` tidak bisa underflow).
2. Matematika reservoir `core/vm/gascosts.go:109-131,157-162,270-290` ekuivalen dengan `charge_state_gas_from_meter`, `credit_state_gas_refund`, `repay_state_gas_spill`.
3. Settlement & refund cap `core/state_transition.go:1032-1095` sama dengan `settle_transaction_gas`; split reservoir `min(MaxTxGas − intrinsic, evmGas)` sama dengan `allocate_evm_gas`.

## Perintah untuk Muse (di VM)
```
cd geth
go test ./core/vm/ -run '8037|8038|SStore|Create' -count=1 -v
go test ./core/ -run 'Amsterdam|7702|SetCode|StateTransition' -count=1 -v
# B1/B2: buat blockchain test dari genesis di atas, lalu bandingkan dengan EELS:
go test ./tests/ -run TestExecutionSpecBlocktests -count=1
```
Catatan: nama test di regex `-run` adalah tebakan; sesuaikan dengan nama test yang ada.

## Yang perlu dilakukan agent berikutnya
1. Cek duplikat B1 dan B2 di issue/PR go-ethereum dan execution-specs.
2. Tulis fixture blockchain test untuk B1 dan B2, jalankan di geth dan EELS, bandingkan post-state dan gasUsed.
3. Konfirmasi apakah penghapusan cek storage EIP-7610 di geth disengaja dan apakah EELS lama (pra-Amsterdam) juga `destroy_storage` — untuk menilai dampak B1 pada fork sebelum Amsterdam.
