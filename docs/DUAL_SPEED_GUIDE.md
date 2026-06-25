# Dual Speed Guide

Purpose: accelerate large NAS transfers while keeping one public NAS path.

## Tool

Use:

```bash
dual-speed
```

The tool uses hidden internal NFSv3 mounts:

- `/mnt/.dual-speed/via-your-nas-ip-a`
- `/mnt/.dual-speed/via-your-nas-ip-b`
- more paths can be added in the config as future multi-NIC support matures

Do not expose these paths to users. User-facing NAS paths remain your public NAS mount path.

## Network

Configured paths:

- `your-client-ip-a/your-client-nic-a` to `your-nas-ip-a`
- `your-client-ip-b/your-client-nic-b` to `your-nas-ip-b`

NFSv3 is required for this acceleration. NFSv4.2 was tested and failed to use both links because the Linux NFSv4 client merged the two NAS IPs into one server/session.

## Quick Checks

```bash
dual-speed --config dual-speed.config.json status
```

Prepare routes and hidden mounts:

```bash
sudo dual-speed --config dual-speed.config.json prepare --remount
```

Run speed test:

```bash
sudo dual-speed --config dual-speed.config.json speedtest --size-mb 1024
```

Expected result:

- Upload combined: about `180-200 MB/s`
- Download combined: about `200 MB/s`

Both configured NICs should show roughly equal byte deltas.

## Upload

Upload a local file or directory to the NAS:

```bash
sudo dual-speed --config dual-speed.config.json upload /path/to/source /your-public-nas-mount-here/path/to/dest --verify size
```

`upload` and `download` default to `--strategy auto`. Auto uses normal single-path copy for small transfers and dual-path copy for larger multi-file transfers. Current default threshold:

- single when there is only `1` file, even if it is large
- dual when there are at least `2` files and total size is at least `256 MB`
- dual when file count is at least `16`
- otherwise single

Force a mode when needed:

```bash
dual-speed --config dual-speed.config.json upload /path/to/source /your-public-nas-mount-here/path/to/dest --strategy single
sudo dual-speed --config dual-speed.config.json upload /path/to/source /your-public-nas-mount-here/path/to/dest --strategy dual
```

Dry run:

```bash
dual-speed --config dual-speed.config.json upload /path/to/source /your-public-nas-mount-here/path/to/dest --dry-run --no-prepare
```

Resume is enabled by default and skips same-size destination files. Use `--verify sha256` only when correctness matters more than speed.

## Download

Download from NAS to local disk:

```bash
sudo dual-speed --config dual-speed.config.json download /your-public-nas-mount-here/path/to/source /local/dest --verify size
```

Dry run:

```bash
dual-speed --config dual-speed.config.json download /your-public-nas-mount-here/path/to/source /local/dest --dry-run --no-prepare
```

## Operating Guidelines

Use this tool for large NAS transfers, especially model folders, datasets, many files, or large multi-file jobs.

Do not use it for tiny one-off files. Plain `cp` is fine for small files.

Commands that modify routes or mounts normally require sudo.

After a transfer, report:

- total size and elapsed time
- combined MB/s
- whether both network paths showed traffic
- verify mode used

## Test Cases

Run the quick validation suite:

```bash
dual-speed-testcases --scale quick
```

Run the larger suite:

```bash
dual-speed-testcases --scale full
```

The suite covers:

- uneven file sizes
- odd number of files
- nested directories
- empty files
- text, JSON, CSV, log, markdown, image-like, archive-like, model-like files
- highly compressible data and random/already-compressed data
- upload, download, resume, and checksum roundtrip validation
- auto strategy threshold behavior

The script also prints a compression probe. Use compression only when it clearly reduces total size enough to offset compression time. For model files such as `.gguf` or `.safetensors`, archives, images, audio, and random-like data, compression usually wastes time. For huge text, CSV, logs, JSON, or zero-heavy data, compression may help.

## Small Files And Packing

Many tiny files are limited by metadata operations, not bandwidth. Packing can help only when the archive is extracted on the NAS host itself.

Recommended rule:

- Single file: use single-path copy unless a future chunk-split mode is added.
- Multi-file and at least `256 MB`: use dual path.
- At least `16` files: use dual path.
- At least `1000` files with average file size below about `1 MB`: consider packing.
- At least `10000` files: strongly consider packing.

Do not pack and then extract from this client through NFS unless the final output can remain an archive. Client-side extraction over NFS reads the archive back over the network and writes all extracted files back again.

NAS-side extraction requires SSH or an API on the NAS. Once remote execution is configured, a future `pack-upload-extract` mode can:

1. create a local `.tar.zst` archive,
2. transfer it with `dual-speed`,
3. run `tar --use-compress-program=zstd -xf ...` on the NAS,
4. verify and remove the archive.

## Future: Multi-Service Scheduling

When many users or services transfer files at the same time, raw parallel copy is not enough. A future scheduler should:

- queue transfer jobs from multiple users or services
- assign priorities and bandwidth caps
- limit per-job concurrency
- split available NIC paths fairly across active jobs
- prevent small interactive transfers from waiting behind bulk jobs
- persist job state for pause, resume, retry, and reboot recovery

This turns dual-speed from a single-command transfer accelerator into a small NAS transfer coordinator.

## Cleanup

The tool writes temporary files using `.part` suffix and atomically renames on success. Failed transfers can be resumed.

Hidden mount points can stay mounted. To reset them:

```bash
sudo umount /mnt/.dual-speed/via-your-nas-ip-a
sudo umount /mnt/.dual-speed/via-your-nas-ip-b
```
