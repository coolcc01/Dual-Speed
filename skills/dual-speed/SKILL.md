---
name: dual-speed
description: Use when copying, uploading, downloading, syncing, moving, or benchmarking large files or model folders to or from a NAS with two available network paths. This skill uses the dual-speed NFSv3 transfer tool to split suitable multi-file work across two NICs while keeping one user-facing NAS path.
---

# Dual Speed

Use `dual-speed` for large NAS transfers, especially model folders, many-file copies, or large datasets. It is designed for dual-NIC hosts today and can evolve toward three or more configured NIC paths. Use plain `cp` for tiny one-off files.

Public paths remain the normal user-facing NAS mount. Do not expose or ask the user to use hidden internal mount paths.

## Commands

Check status:

```bash
dual-speed --config dual-speed.config.json status
```

Prepare hidden mounts and routes:

```bash
sudo dual-speed --config dual-speed.config.json prepare --remount
```

Upload:

```bash
sudo dual-speed --config dual-speed.config.json upload /local/source /your-public-nas-mount-here/target --verify size
```

Download:

```bash
sudo dual-speed --config dual-speed.config.json download /your-public-nas-mount-here/source /local/target --verify size
```

Dry run without sudo:

```bash
dual-speed --config dual-speed.config.json upload /local/source /your-public-nas-mount-here/target --dry-run --no-prepare
```

Speed test:

```bash
sudo dual-speed --config dual-speed.config.json speedtest --size-mb 1024
```

Validation test cases:

```bash
dual-speed-testcases --scale quick
```

## Rules

- If sudo requires a password, give the exact command for the user to run and continue from their pasted output.
- `upload` and `download` default to `--strategy auto`: one file uses single path; multi-file transfers use dual path at 256 MB or 16 files and above; smaller transfers use ordinary single-path copy to avoid overhead.
- Use `--strategy dual` to force acceleration for benchmarks or known-large jobs. Use `--strategy single` for tiny or latency-sensitive jobs.
- Prefer `--verify size` for normal large transfers. Use `--verify sha256` only when integrity matters more than speed.
- Resume is enabled by default and skips existing same-size files.
- After a transfer, report total size, elapsed time, combined MB/s, verification mode, and whether both configured NICs showed traffic.
- If dual-path traffic fails, fall back to ordinary NAS transfer and mention that acceleration was unavailable.
- Do not compress model files, archives, images, audio, or random-like data by default. Consider compression only for large text, CSV, JSON, logs, or zero-heavy data, and only when the compression probe shows enough size reduction.
- For many tiny files, packing helps only if extraction runs on the NAS via SSH/API. Do not pack and then extract through NFS from this client; that usually wastes network I/O.
- Future multi-service support should use a scheduler daemon with job queues, priorities, bandwidth caps, pause/resume/cancel, and fair path allocation across users or services.

Reference guide: `docs/DUAL_SPEED_GUIDE.md`.
