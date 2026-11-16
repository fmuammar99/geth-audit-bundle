# HASIL OPUS_TASK_SOL_02 — Solidity compiler: type checker

Target: ethereum/solidity commit `41d9e65`, dibaca dari `sol-task02/solidity/` di repo `fmuammar99/solidity-audit-bundle`.
Tidak ada build/test yang dijalankan. Tag: **checked** = kode dibaca langsung; **inferred** = penalaran tanpa menjalankan.
Cek duplikat di GitHub BELUM dilakukan (tugas Muse).

## Sumber
- Dari bundle: `analysis/` (ViewPureChecker, ImmutableValidator, DeclarationTypeChecker, ConstantEvaluator, TypeChecker), `ast/`, `docs/bugs.json` (dari bundle task 01).
- Diambil dari GitHub di `41d9e65` (tidak ada di bundle): `libevmasm/SemanticInformation.cpp` (klasifikasi opcode pure/view), `docs/contracts/constant-state-variables.rst` dan `docs/contracts/functions.rst` (pembanding dokumentasi).
- **Cakupan terbatas:** pemeriksaan #2 (FunctionCall/overload), #3 (OverrideChecker), #7 (resolusi nama) dan #8 (control flow) TIDAK dibaca tuntas. Tidak ada temuan di sana karena belum diaudit, bukan karena aman.

## Ringkasan
- 1 kandidat konkret: fungsi `pure` boleh membaca immutable yang diinisialisasi literal, padahal constructor masih boleh menimpanya (sejak 0.8.21). Dampaknya rendah: pemeriksa mutability tidak sound, bukan miscompile.
- Tidak menemukan bug lain di area yang dibaca.

## Hasil 8 pemeriksaan

| # | Pemeriksaan | Hasil | Lokasi | Tag |
|---|---|---|---|---|
| 1 | Konversi implisit | Tidak dibaca tuntas (`ast/Types.cpp` sangat besar). | — | — |
| 2 | Panggilan fungsi | Tidak dibaca tuntas. Hanya `erc7201` yang dicek (tipe argumen harus `string`/literal; lihat laporan SOL_01). | `analysis/TypeChecker.cpp:2405-2427` | checked (sebagian) |
| 3 | Override | Tidak dibaca. | — | — |
| 4 | View / pure | Opcode assembly diklasifikasi lewat `SemanticInformation`: TLOAD, BLOBHASH, BLOBBASEFEE, SLOTNUM, SLOAD, dsb. = baca state (tidak boleh di pure); TSTORE, SSTORE, LOG, CALL, CREATE, SELFDESTRUCT = tulis (tidak boleh di view). Operator user-defined, modifier, akses member storage/alamat/magic ditangani. Lihat A1 untuk immutable. | `analysis/ViewPureChecker.cpp:66-123, 184-222, 334-470`; `libevmasm/SemanticInformation.cpp:551-611` | checked |
| 5 | Data location | Transient ditolak untuk constant/immutable dan untuk inisialisasi inline; transient untuk variabel lokal/parameter → `solUnimplemented`; transient butuh EVM ≥ cancun. | `analysis/DeclarationTypeChecker.cpp:386-520` | checked |
| 6 | Konstanta & immutable | `ConstantEvaluator` hanya meng-fold variabel `constant` (bukan immutable); `isPure` hanya untuk constant. `ImmutableValidator` menolak tulis immutable di fungsi non-constructor dan modifier. Lihat A1. | `analysis/ConstantEvaluator.cpp:434-439`; `analysis/TypeChecker.cpp:3859`; `analysis/ImmutableValidator.cpp:25-67` | checked |
| 7 | Resolusi nama & scope | Tidak dibaca. | — | — |
| 8 | Control flow | Tidak dibaca. | — | — |

## (A) Kandidat bug compiler

### A1 — Fungsi `pure` bisa membaca immutable yang nilainya ditentukan saat deploy
- File: `libsolidity/analysis/ViewPureChecker.cpp:194-199`.
- Masalah: pembacaan immutable dianggap `pure` bila deklarasinya punya inisialisasi literal (`varDecl->value()` bertipe `RationalNumber`), dengan komentar "Immutables that are assigned literals are pure". Sejak 0.8.21, constructor boleh menimpa immutable berkali-kali walau sudah diinisialisasi inline: `ImmutableValidator` sengaja tidak memeriksa constructor (`ImmutableValidator.cpp:40-43`), dan dokumentasi resmi memberi contoh `uint immutable decimals = 18;` yang ditimpa `decimals = decimals_;` di constructor. Akibatnya, nilai yang "pure" sebenarnya bergantung pada argumen constructor atau lingkungan deploy (mis. `block.timestamp`).
- Kontrak contoh minimal (`ImmPure.sol`):
  ```solidity
  // SPDX-License-Identifier: UNLICENSED
  pragma solidity ^0.8.21;
  contract C {
      uint immutable x = 1;              // literal → dianggap pure
      constructor(uint v) { x = v; }     // diizinkan
      function f() external pure returns (uint) { return x; }   // DITERIMA
  }
  contract D {
      uint immutable y;                  // tanpa literal
      constructor(uint v) { y = v; }
      function g() external pure returns (uint) { return y; }   // DITOLAK (seharusnya)
  }
  ```
- Perilaku yang dihasilkan (inferred): `C.f` lolos sebagai `pure`; `D.g` ditolak dengan error "Function declared as pure, but this expression ... reads from the environment or state". Dua pola yang secara semantik sama diperlakukan berbeda.
- Yang seharusnya: pembacaan immutable yang bisa ditimpa di constructor diperlakukan `view` (seperti `D.g`), atau penimpaan immutable yang punya inisialisasi literal ditolak.
- Dampak: rendah. Tidak ada miscompile — kedua pipeline tetap membaca nilai immutable yang benar dan `pure` tidak mengubah codegen pemanggilan secara berbahaya. Masalahnya pada jaminan bahasa: kode/alat (mis. ABI `stateMutability: pure`, analisis statis, auditor) mengira `f()` konstanta, padahal hasilnya berbeda per deployment.
- Cek bugs.json: tidak ada entri tentang mutability immutable/pure. Muse perlu cek issue GitHub (mungkin dianggap desain).
- Perintah (solc 0.8.39-develop di VM):
  ```
  solc --bin ImmPure.sol
  solc --via-ir --bin ImmPure.sol
  ```
  Harapan saat ini: hanya error untuk `D.g`. Untuk bukti runtime, deploy `C` dengan `v = 42` lalu panggil `f()` → mengembalikan 42 meski ditandai `pure`.
- Tag: checked (kode + dokumentasi), inferred (output compiler).

## (B) Kandidat divergensi
- **B1 (sama akarnya dengan A1):** dokumentasi `functions.rst:227-233` tidak menyebut immutable sebagai "membaca state", dan `constant-state-variables.rst` menyatakan "There are no restrictions on reading immutable variables" — tetapi compiler mewajibkan `view` untuk immutable tanpa literal dan mengizinkan `pure` untuk yang ber-literal. Dokumentasi dan compiler tidak konsisten. Tag: checked.
- Tidak ada divergensi legacy vs `--via-ir` yang ditemukan; semua yang saya baca ada di tahap analisis yang sama untuk kedua pipeline.

## (C) Temuan lemah / perlu cek lanjutan

| # | Lokasi | Catatan | Cek bugs.json | Tag |
|---|---|---|---|---|
| C1 | `analysis/ImmutableValidator.cpp:25-37` | Hanya fungsi dan modifier yang dikunjungi; penulisan immutable di dalam inisialisasi state variable lain (mis. `uint z = (x = 5);`) tidak diperiksa di sini. Secara semantik itu terjadi saat konstruksi, jadi kemungkinan sengaja; perlu konfirmasi. | Tidak ada entri. | inferred |
| C2 | `analysis/ViewPureChecker.cpp:66-73` | Hanya builtin Yul yang punya `instruction` yang diklasifikasi. Builtin tanpa opcode langsung (mis. `datasize`, `loadimmutable`) tidak diperiksa; di inline assembly Solidity builtin itu biasanya tidak tersedia, jadi kemungkinan aman. | Tidak ada entri. | inferred |
| C3 | Pemeriksaan #1, #2, #3, #7, #8 | Belum diaudit. Prioritas berikut: overload resolution + named arguments (ada bug `MisorderedNamedParametersInRequireWithCustomErrors` baru di area ini), OverrideChecker untuk state variable `transient public`, dan ControlFlowAnalyzer untuk `try/catch` + modifier. | `MisorderedNamedParametersInRequireWithCustomErrors` (fixed 0.8.37) | — |

## 3 kandidat terkuat
1. **A1** — `pure` bisa membaca immutable ber-literal yang ditimpa di constructor (`ViewPureChecker.cpp:194-199`).
2. **B1** — dokumentasi vs compiler soal membaca immutable di `pure`/`view`.
3. **C1** — immutable ditulis di inisialisasi state variable lain (perlu konfirmasi apakah sengaja).

## 3 hal yang sudah dicek dan aman
1. **Klasifikasi opcode assembly untuk pure/view**: `libevmasm/SemanticInformation.cpp:551-611` mencakup opcode baru (TLOAD/TSTORE, BLOBHASH, BLOBBASEFEE, SLOTNUM), dipakai di `ViewPureChecker.cpp:115-121`.
2. **Aturan transient**: `DeclarationTypeChecker.cpp:474-487` menolak transient untuk constant/immutable dan inisialisasi inline; `:510-512` menolak transient untuk variabel lokal/parameter.
3. **Constant folding tidak menyentuh immutable**: `ConstantEvaluator.cpp:434-439` dan `TypeChecker.cpp:3859` hanya memperlakukan variabel `constant` sebagai konstanta compile-time, jadi A1 tidak berubah menjadi miscompile.

## Yang perlu dilakukan agent berikutnya
1. Jalankan `ImmPure.sol` di atas dengan solc 0.8.39-develop (legacy dan via-ir) untuk mengonfirmasi A1.
2. Cek issue GitHub ethereum/solidity tentang "immutable pure" untuk duplikat/desain.
3. Lanjutkan audit pemeriksaan #1, #2, #3, #7, #8 yang belum tersentuh.
