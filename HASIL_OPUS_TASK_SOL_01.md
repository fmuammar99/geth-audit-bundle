# HASIL OPUS_TASK_SOL_01 — Solidity compiler: codegen storage / SSTORE

Target: ethereum/solidity commit `41d9e65`, dibaca dari repo `fmuammar99/solidity-audit-bundle`.
Tidak ada build/test yang dijalankan. Tag: **checked** = kode dibaca langsung; **inferred** = penalaran tanpa menjalankan.
Cek duplikat di GitHub BELUM dilakukan (tugas Muse).

## Catatan sumber
- Bundle berisi `libsolidity/` (analysis, ast, codegen, codegen/ir, formal, interface, parsing), `libyul/`, dan `docs/bugs.json`.
- `libevmasm/` TIDAK ada di bundle. Untuk satu pemeriksaan saya mengambil `libevmasm/SemanticInformation.cpp` dari GitHub di commit `41d9e65`.
- `ConstantEvaluator` ada di `libsolidity/analysis/ConstantEvaluator.cpp` (bukan di `codegen/` seperti tertulis di task).
- **Penting untuk Muse:** `bugs.json` di commit ini sudah memuat perbaikan sampai 0.8.38, sedangkan solc di VM adalah 0.8.28. Banyak bug yang sudah ditambal (mis. `TransientStorageClearingHelperCollision`, `LostStorageArrayWriteOnSlotOverflow`, `YulOptimizerKnowledgeBaseStaleBookkeeping`) masih ada di 0.8.28. Hasil uji dengan 0.8.28 TIDAK mewakili commit target. Untuk bukti, pakai solc yang di-build dari `41d9e65` (atau nightly terdekat).

## Ringkasan
- (A) Tidak ada bug codegen yang terkonfirmasi.
- (B) Tidak ada divergensi legacy vs `--via-ir` yang terkonfirmasi; ada 3 area yang layak diuji diferensial.
- Beberapa area rawan (transient vs storage, layout, erc7201, UnusedStoreEliminator) sudah dicek dan aman.

## Hasil 8 pemeriksaan

| # | Pemeriksaan | Hasil | Lokasi | Tag |
|---|---|---|---|---|
| 1 | Storage layout | `computeOffsets` benar: pindah slot bila `byteOffset + storageBytes > 32`; tipe multi-slot mulai di slot baru; assert bila melewati 2^256. Urutan inheritance dari `linearizedBaseContracts` dibalik (paling dasar dulu); constant & immutable dikecualikan. Basis layout kustom (`layout at`) hanya untuk storage; transient selalu mulai dari 0. Legacy dan IR mendaftarkan slot dari fungsi yang sama. | `ast/Types.cpp:142-175, 2150-2182`; `ast/ASTUtils.cpp:127-139`; `codegen/ContractCompiler.cpp:571-576`; `codegen/ir/IRGenerator.cpp:1121-1126` | checked |
| 2 | SSTORE / TSTORE codegen | Legacy: `GenericStorageItem<IsTransient>` memilih SSTORE/TSTORE lewat `s_storeInstruction` (`LValue.h:177-178`); masking packed-slot benar. IR: lvalue `Storage` vs `TransientStorage` memanggil helper dengan lokasi yang benar (`IRGeneratorForStatements.cpp:3118-3150, 3228-3254`). Nama helper memuat prefix `transient_`, jadi tidak bertabrakan (pola bug `TransientStorageClearingHelperCollision`). | `codegen/LValue.cpp:305-469`; `codegen/YulUtilFunctions.cpp:2825-2990, 4376-4392` | checked |
| 3 | `delete` storage | Legacy struct: dibersihkan per anggota, mapping dilewati (`LValue.cpp:478-497`); value type packed: hanya byte-nya yang di-nol-kan. Elemen `bytes` storage: hanya 1 byte di-nol-kan (`LValue.cpp:570-583`). IR: `storageSetToZeroFunction` dengan lokasi eksplisit (`IRGeneratorForStatements.cpp:715-735`). Loop pembersihan memakai counter (`lt(i, slotCount)`), bukan perbandingan slot absolut, sesuai fix `LostStorageArrayWriteOnSlotOverflow`. | lihat kolom hasil; `YulUtilFunctions.cpp:1861` | checked |
| 4 | ABI encoder storage → memory | Copy byte array storage → memory (legacy): short array (≤31) membuang byte panjang lewat `DIV/MUL 0x100`; long array per slot; loop berhenti berdasarkan alamat memori. Konversi `bytes storage` → `bytesN` (IR) untuk short array membaca slot langsung lalu memasking byte di luar panjang, termasuk byte panjang. | `codegen/ArrayUtils.cpp:240-330`; `codegen/YulUtilFunctions.cpp:3704-3760` | checked |
| 5 | IR vs legacy | Legacy `copyArrayToStorage` sekarang mendelegasikan ke helper Yul yang sama dengan IR (`copyArrayToStorageFunction`); urutan argumen stack cocok dengan signature `(slot, value[, len])`. Self-assign storage→storage ditangani (`if eq(slot, value) leave`). | `codegen/ArrayUtils.cpp:44-104`; `codegen/YulUtilFunctions.cpp:1967-2060` | checked |
| 6 | Yul optimizer & storage | `UnusedStoreEliminator` hanya menjadikan `SSTORE` (bukan `TSTORE`) kandidat penghapusan. Operasi `TransientStorage` dari `SemanticInformation` tidak bisa "menutupi" atau "membaca" store lain karena `knownUnrelated`/`knownCovered` menolak lokasi berbeda di awal. `movableRelativeTo` memisahkan efek storage dan transient. | `libyul/optimiser/UnusedStoreEliminator.cpp:155-195, 250-365`; `libyul/optimiser/Semantics.h:69-104`; `libevmasm/SemanticInformation.cpp` (TSTORE/TLOAD → `Location::TransientStorage`) | checked |
| 7 | ConstantEvaluator | `erc7201(id)` compile-time = `keccak256(uint256(keccak256(id)) - 1)` lalu byte terendah di-nol-kan; sesuai ERC-7201. Legacy dan IR memakai fungsi yang sama untuk literal; untuk string runtime keduanya memakai helper Yul `erc7201` yang sama, dan urutan argumen stack legacy cocok. | `analysis/ConstantEvaluator.cpp:457-489`; `ast/ASTUtils.cpp:141-158`; `codegen/ExpressionCompiler.cpp:931-971`; `codegen/ir/IRGeneratorForStatements.cpp:1432-1458`; `codegen/YulUtilFunctions.cpp:1301-1316` | checked |
| 8 | Cross-check bugs.json | Lihat kolom "Cek bugs.json" di (C). | `docs/bugs.json` (67 entri) | checked |

## (A) Kandidat bug compiler — codegen salah untuk source valid
Tidak ada yang terkonfirmasi.

## (B) Kandidat divergensi — `--via-ir` vs legacy, atau solc vs dokumentasi
Tidak ada yang terkonfirmasi. Area yang layak diuji diferensial ada di (C1)–(C3).

## (C) Temuan lemah / perlu cek lanjutan

| # | Lokasi | Catatan | Cek bugs.json | Tag |
|---|---|---|---|---|
| C1 | `codegen/ArrayUtils.cpp:44-104` | Legacy `copyArrayToStorage` kini memakai helper Yul IR (ada komentar TODO "kept temporarily for backward compatibility"). Ini perubahan perilaku legacy terhadap rilis lama; perlu uji diferensial legacy vs via-ir untuk penyusutan array packed (`uint128[]`), `bytes` calldata/memori → storage, dan array struct storage → storage. | Mirip `DynamicArrayCleanup` (≤0.7.3), `DirtyBytesArrayToStorage` (≤0.8.15), `EmptyByteArrayCopy` (≤0.7.4), `LostStorageArrayWriteOnSlotOverflow` (≤0.8.32) — semuanya sudah diperbaiki; kandidat hanya bernilai bila ada perilaku BARU yang salah. | inferred |
| C2 | `codegen/LValue.cpp:478-497` vs IR `clearStorageStructFunction` (`YulUtilFunctions.cpp` ~1930-1950) | `delete` struct dibuat dengan dua implementasi berbeda (legacy per anggota lewat `StorageItem`, IR lewat helper Yul). Struct dengan anggota packed + static array + struct bersarang + mapping layak diuji diferensial. | Tidak ada entri yang persis; terkait `TransientStorageClearingHelperCollision` (≤0.8.34, viaIR) untuk kelas "helper clearing". | inferred |
| C3 | `codegen/ExpressionCompiler.cpp:943-971` | `erc7201(s)` dengan `s` runtime dari storage/calldata: legacy memakai `packedEncode` ke free memory tanpa menaikkan free memory pointer lalu langsung hash. Analisis saya benar, tapi layak diuji legacy vs IR vs nilai konstanta. | Tidak ada entri terkait erc7201. | checked (kode), inferred (dampak) |
| C4 | `libyul/optimiser/DataFlowAnalyzer.cpp` / `LoadResolver.cpp` | Tidak saya baca tuntas. Fix `YulOptimizerKnowledgeBaseStaleBookkeeping` (≤0.8.38) baru masuk; area ini (cache basis variabel yang di-reassign) layak diperiksa ulang untuk varian lain, termasuk `tload`/`tstore` dan `keccak256`. | `YulOptimizerKnowledgeBaseStaleBookkeeping` (fixed 0.8.38), `StorageWriteRemovalBeforeConditionalTermination` (fixed 0.8.17). | belum dicek |

## 3 kandidat terkuat
(Semuanya lemah; tidak ada yang terkonfirmasi sebagai bug.)
1. C1 — legacy `copyArrayToStorage` yang kini lewat helper Yul (`codegen/ArrayUtils.cpp:44-104`).
2. C2 — `delete` struct storage: implementasi legacy vs IR berbeda (`codegen/LValue.cpp:478-497`).
3. C4 — varian lain dari bug cache basis variabel di `DataFlowAnalyzer`/`LoadResolver` (belum dicek).

## 3 hal yang sudah dicek dan aman
1. **Pemisahan storage vs transient** di legacy (`codegen/LValue.h:177-178`, `codegen/LValue.cpp:305-469`) dan IR (`codegen/ir/IRGeneratorForStatements.cpp:715-735, 3118-3150`; nama helper ber-prefix `transient_` di `codegen/YulUtilFunctions.cpp:2842, 2955, 4390`).
2. **Layout storage**: `ast/Types.cpp:142-175` (packing/slot), `ast/ASTUtils.cpp:127-139` (basis `layout at` hanya untuk storage, transient = 0), registrasi slot sama di legacy dan IR (`codegen/ContractCompiler.cpp:571-576`, `codegen/ir/IRGenerator.cpp:1121-1126`).
3. **UnusedStoreEliminator vs TSTORE**: `libyul/optimiser/UnusedStoreEliminator.cpp:161, 280-283, 351-352` — hanya SSTORE yang bisa dihapus; operasi lokasi berbeda tidak pernah dianggap menutupi/membaca.

## Kontrak uji untuk Muse (diferensial legacy vs via-ir)

Simpan sebagai `T.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.28;

contract T {
    struct Inner { uint8 a; uint16[3] b; }
    struct S { uint128 x; uint64 y; Inner inner; mapping(uint => uint) m; bytes data; }

    uint128[] packed;
    bytes blob;
    S s;
    Inner[] arr;
    Inner[] arr2;
    string ns;

    // C1: shrink packed array then grow with push() -> new elements must be 0
    function c1a() external returns (uint128, uint128) {
        packed = new uint128[](5);
        for (uint i; i < 5; i++) packed[i] = uint128(i + 1);
        uint128[] memory small = new uint128[](1);
        small[0] = 7;
        packed = small;
        packed.push(); packed.push(); packed.push();
        return (packed[1], packed[3]);
    }
    // C1: bytes calldata -> storage, then push() must give 0
    function c1b(bytes calldata d) external returns (bytes1) {
        blob = d;
        blob.push();
        return blob[blob.length - 1];
    }
    // C1: struct array storage -> storage
    function c1c() external returns (uint8, uint16) {
        arr.push(Inner(1, [uint16(2), 3, 4]));
        arr.push(Inner(5, [uint16(6), 7, 8]));
        arr2 = arr;
        return (arr2[1].a, arr2[1].b[2]);
    }
    // C2: delete struct with packed members, static array, nested struct, mapping, bytes
    function c2() external returns (uint128, uint64, uint8, uint16, uint, uint) {
        s.x = 1; s.y = 2; s.inner.a = 3; s.inner.b[2] = 4; s.m[1] = 5; s.data = "abcdefghijklmnopqrstuvwxyz0123456789";
        delete s;
        return (s.x, s.y, s.inner.a, s.inner.b[2], s.m[1], s.data.length);
    }
    // C3: erc7201 runtime vs compile-time
    function c3(string calldata id) external returns (bool, bool, bool) {
        ns = id;
        string memory mem = id;
        uint k = erc7201("example.main");
        return (erc7201(id) == k, erc7201(ns) == k, erc7201(mem) == k);
    }
}
```

Hasil yang diharapkan (keduanya pipeline harus sama):
- `c1a()` → `(0, 0)`; `c1b(0x6162)` → `0x00`; `c1c()` → `(5, 8)`;
- `c2()` → `(0, 0, 0, 0, 5, 0)` (mapping tidak dihapus oleh `delete`);
- `c3("example.main")` → `(true, true, true)`.

Perintah (pakai solc yang di-build dari commit `41d9e65`; 0.8.28 hanya sebagai pembanding):
```
solc --evm-version cancun --bin-runtime --optimize T.sol -o out_legacy
solc --evm-version cancun --bin-runtime --optimize --via-ir T.sol -o out_ir
solc --evm-version cancun --ir-optimized --optimize T.sol > T.ir.yul
```
`erc7201` adalah builtin baru dan mungkin tidak ada di 0.8.28; bila kompilasi gagal di 0.8.28, hapus `c3`.
Untuk menjalankan, paling mudah lewat Foundry (`forge test`, sekali dengan `via_ir = false` dan sekali `via_ir = true` di `foundry.toml`, `solc` diarahkan ke binari hasil build) dan bandingkan nilai kembalian semua fungsi.

## Yang perlu dilakukan agent berikutnya
1. Build solc dari `41d9e65` (atau ambil nightly terdekat) dan jalankan `T.sol` di legacy dan via-ir.
2. Baca tuntas `libyul/optimiser/DataFlowAnalyzer.cpp`, `LoadResolver.cpp`, `EqualStoreEliminator.cpp` untuk varian lain dari `YulOptimizerKnowledgeBaseStaleBookkeeping` (C4).
3. Cek duplikat C1–C4 di issue/PR ethereum/solidity.
