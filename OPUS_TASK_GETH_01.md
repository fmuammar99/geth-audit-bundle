# OPUS_TASK_GETH_01 — Geth EVM: gas & state gas Amsterdam (EIP-8037/8038)

Konteks: ini task pertama audit Geth. Kamu mulai dari KOSONG (tidak ada repo
lokal). Clone dulu, baca kode, jawab dalam Bahasa Indonesia.
Target: https://github.com/ethereum/go-ethereum — commit `fac2e99`
("accounts/keystore: fix the race between key scan and delete (#35846)").

## Langkah 0 — clone (jalankan di sesimu)

```
git clone --depth 1 https://github.com/ethereum/go-ethereum.git geth
git clone --depth 1 https://github.com/ethereum/EIPs.git eips
git clone --depth 1 https://github.com/ethereum/execution-specs.git execution-specs
```

Referensi angka (EIP-8037/8038, dari EELS):
state gas slot baru 97.920, akun baru 183.600, auth base 35.190,
code deposit 1.530/byte, SSTORE slot baru cold 110.020, slot lama warm
10.100 / cold 12.100, CREATE ke alamat kosong 195.600, refund clear
11.616, refund cap 20%, reservoir hanya bila tx.gas > 2^24 (16.777.216),
SYSTEM_MAX_SSTORES_PER_CALL = 16.

## Yang dicek (8 pemeriksaan)

1. **Konstanta state gas.** Cari di `geth/core/vm/` (gas_table.go,
   gascosts.go, operations_acl.go) semua angka state gas. Bandingkan satu
   per satu dengan tabel EIP-8037/8038 di `eips/EIPS/eip-8037.md` dan
   `eip-8038.md`. Angka mana yang beda, hilang, atau pakai nilai lama?
2. **Reservoir.** Cari implementasi reservoir di `geth/core/vm/gas.go`
   (dan `core/vm/evm.go` / `core/state/` bila perlu). Bandingkan dengan
   EELS `execution-specs/src/ethereum/forks/amsterdam/vm/gas.py`:
   reservoir hanya bila tx.gas > 2^24? Kredit dibatasi spill? Repay =
   min(reservoir, spill)? Restore pakai baseline?
3. **SSTORE.** Baca `sstore` di `geth/core/vm/instructions.go`:
   jalur charge/refund untuk slot dirty vs clean, restorasi slot
   (set lalu clear dalam satu tx), refund clear 11.616 dan cap 20%.
   Ada jalur yang charge-nya kurang / refund-nya lebih?
4. **CREATE/CREATE2.** Charge ke alamat kosong (195.600?), code deposit
   per byte, batas initcode. Bandingkan dengan EIP-8037.
5. **CALL family.** `call`, `callCode`, `delegateCall`, `staticCall` di
   instructions.go: aturan 63/64, charge akun baru, transfer value,
   memory expansion. Ada jalur yang lolos tanpa charge akun baru?
6. **Refund cap.** Cari di mana refund total di-cap (20% per EIP-8038,
   dulu 50%). Apakah cap dipakai konsisten di semua jalur refund
   (SSTORE clear, SELFDESTRUCT, CREATE)?
7. **EIP-7702.** Setup delegation: `core/types/authorization.go` dan
   pemakaian di `core/vm/` / `core/state_processor.go`. Apakah charge
   auth base 35.190 diterapkan? Apakah otorisasi yang gagal tetap
   ke-charge? Apakah nonce/chain_id dicek sebelum charge?
8. **SYSTEM_MAX_SSTORES_PER_CALL = 16.** Cari enforcement-nya (system
   contracts / precompile?). Apa yang terjadi pada store ke-17 dalam
   satu call — error, charge, atau diam-diam lolos?

## Aturan

- JANGAN menjalankan test berat / build penuh di sesimu (kamu di Android).
  Kalau perlu test, tulis perintah `go test` persisnya — Muse yang
  menjalankan di VM.
- Tandai tiap temuan "checked" (kamu baca kodenya langsung) atau
  "inferred" (penalaran dari kode tanpa menjalankan).
- JANGAN klaim "belum pernah dilaporkan" — Muse yang cek duplikat di GitHub.

## Format jawaban (Bahasa Indonesia)

- (A) Kandidat bug — selisih geth-vs-EIP/EELS atau salah hitung gas.
  Tiap kandidat: file+baris, apa yang salah, dampak (curi gas / gas
  gratis / DoS / split), cara memicu.
- (B) Kandidat consensus-split / divergensi (geth berperilaku beda dari
  EELS/client lain untuk input yang sama).
- (C) Temuan lemah / perlu cek lanjutan.
- 3 kandidat terkuat.
- 3 hal yang sudah dicek dan ternyata aman (dengan file+baris).
