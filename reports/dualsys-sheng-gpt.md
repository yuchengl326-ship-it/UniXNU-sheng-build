# Dual-system partition adaptation on sheng (macOS 27.3)

Host and device evidence for the fork-level dual_sys changes. All commands ran on
2026-11-22 against sheng (Xiaomi Pad 6S Pro 12.4) in TWRP 3.7.1_12-0, kernel
5.15.144-android13-8-00017-gdaa558ea5602-ab11990578, using the UniXNU dual_sys
toolkit at revision 2bbb39dcee0cada3b04e41f0256d678e80667c5b with the fork
patches in `patches/dual-sys-sheng-vendor-gpt.patch`.

## Disk geometry

- `/dev/block/sda`: logical sector 4096 bytes, 262,173,360,128 bytes
  (64,007,168 logical sectors). GPT usable range LBA 6..64,007,160.
- 29 occupied entries; `userdata` is entry 29 at LBA 3,094,592..64,007,160
  (232.363 GiB); `super` is entry 28 at 211,008..3,094,591 (11 GiB).

## Vendor GPT layout

Primary header (LBA 1) and backup header (LBA 64,007,167) both parse and pass
their CRC, and both declare count 32, width 128, first 6, last 64,007,160 and
table CRC 3475259004:

- primary header: here=1, other=64,007,167, table=2
- backup header: here=64,007,167, other=1, table=64,007,161

The standard position for a 16 KiB entry array before the backup header is
`total - 1 - blocks = 64,007,163`. sheng instead keeps the backup array at LBA
64,007,161; the bytes there equal the primary table byte for byte and
64,007,162..64,007,166 are zero. The upstream `gpt.header` therefore rejected
the disk with `non-contiguous GPT tables are unsupported`, and
`partition.py inspect` could not run.

Fork adaptation (keeps the vendor topology instead of normalising it):

- `gpt.header` accepts a backup `table` LBA anywhere with
  `last < table <= total - 1 - blocks`, so both the vendor position and the
  standard position pass.
- `read_gpt` reads the backup header at `total - 1`, then the entry array at the
  header's own `table` LBA; the logical backup region is array-then-header, which
  keeps `GPT.split` and the byte comparisons unchanged.
- `partition.py` exports `BACKUP_TABLE_LBA`, `BACKUP_HEADER_LBA` and
  `TABLE_BLOCKS`; `twrp.sh` compares and writes the array and the header
  separately. On a standard disk the two writes are adjacent, so the emitted
  bytes are identical to the upstream single write.

A synthetic standard layout (table moved to 64,007,163, header field and CRC
rewritten) also passed the patched reader and split, confirming no regression.

## F2FS superblock differences

`f2fs-shrink` reads 8192 bytes from the decrypted mapping (`/dev/block/dm-8`)
and compares the superblock copies at offsets 1024 and 5120. On sheng the copies
differ in exactly 48 bytes inside the 256-byte field at offset 0x684 of the
packed superblock, which records the release of the kernel that last mounted
`/data`:

- copy 0: `5.15.144-android13-8-00017-gdaa558ea5602-ab11990578` (TWRP)
- copy 1: `5.15.195-MorizukiNeko-260620LTSC` (previous recovery)

Every geometry field and both `block_count` values agree. The fork masks
0x684..0x783 in both copies before the equality check and keeps the geometry
fields and `block_count` strict. After the online resize both copies were
rewritten and the helper's `VERIFIED: both F2FS superblocks fit the planned
partition` passed.

## Apply, interruption and resume

`partition.py apply` performed the online F2FS shrink (60,912,569 blocks to
48,329,145, partition limit 48,329,408), then `umount /data`. TWRP left a second
mount of dm-8 at `/sdcard`, so `require_unmounted` stopped the script before any
GPT write with `STOP: userdata or a bind mount is still mounted`.
`stage` recorded `F2FS resize verified; GPT still original`, and the GPT was
verified byte-identical to the planned originals in the resume step.

Resume from the same bundle:

1. `umount /sdcard` (exit 0), then `f2fs-shrink offline /dev/block/sda29
   249497882624 48329145 48329408 /dev/block/dm-8` printed
   `VERIFIED: unmounted filesystem fits the planned userdata partition`.
2. `resume_gpt.sh` compared the live GPT with the bundle's before images
   (`BEFORE_OK`), wrote the backup array (4 sectors at LBA 64,007,161) and the
   backup header (LBA 64,007,167), verified the readback (`BACKUP_OK`), wrote
   the primary region (6 sectors at LBA 0) and verified both copies
   (`GPT_DONE`).

After `adb reboot recovery`, `partition.py verify --plan` printed
`PASS: GPT copies, kernel partition ranges and mounted F2FS size match the
plan.` with:

- userdata entry 29: LBA 3,094,592..51,423,999 (184.362 GiB), start 24,756,736
  in 512-byte units, F2FS 48,329,145 blocks, limit 48,329,408
- unixnu entry 30: LBA 51,424,000..64,006,911 (48.000 GiB), 51,539,607,552
  bytes, `/dev/block/sda30`

## Backups

| file | bytes | sha256 |
| --- | --- | --- |
| metadata.img | 67,108,864 | f1bbd0e0291910c833b8ed4c72c42a0d57f08d4df866f121591ce8dcec9b328f |
| gpt_head_raw.bin (LBA 0..39) | 163,840 | 85b009aeec66604386da72fd3bdf2595b62a07c2674a9036d7ec10e92b226279 |
| gpt_tail_raw.bin (LBA 64,007,158..167) | 40,960 | 6992397a28f8a14d8db02cdbdb943d260e9b2d00661d4ac0d70a978e93234960 |

The apply bundle keeps `gpt-primary.before/after` and `gpt-backup.before/after`
plus the helper and `plan.json`, so the write can be re-compared at any time.
