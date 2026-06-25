# Dual-Speed

**More network lanes. More NAS speed.**

## Common Use Case

You have:

- a workstation or server motherboard with two network ports
- a NAS reachable through two IP addresses
- a large folder of model files, media files, backups, datasets, or project artifacts
- a normal public mount path such as `/your-public-nas-mount-here`

`dual-speed` can split multi-file transfers across both paths and keep the visible workflow simple. The config format already uses a `paths` array so the next development stage can extend this from two NICs to three or more NICs.

## When It Helps

Best fit:

- multiple large files
- model folders or datasets
- NAS upload and download jobs
- dual-NIC motherboards or workstations with two network paths to the same NAS
- users who want link-aggregation-like transfer gains without configuring switch LAG groups or OS bonding

## Roadmap

Next development stage: multi-NIC support.

Planned work:

- support three or more configured NIC paths as a first-class workflow
- improve file assignment for N-way balancing
- expand benchmarks from single and dual path to N-path comparison
- add per-path health checks and automatic path exclusion
- optionally add chunk-splitting for single very large files
- optionally add NAS-side archive extraction for many tiny files when SSH/API is configured

Following stage: multi-service and multi-user scheduling.

Planned work:

- run a local transfer scheduler daemon
- accept jobs from multiple users or services
- queue jobs instead of letting every process saturate the NAS independently
- assign per-job priority, bandwidth caps, and concurrency limits
- split available NIC paths fairly across active jobs
- prevent small interactive transfers from being blocked by large background jobs
- expose job status, pause, resume, cancel, and retry controls
- persist job state so transfers can resume after reboot or network interruption
- support policy profiles such as interactive, background, backup, and bulk-model-sync

The goal is to provide link-aggregation-like throughput for bulk work while keeping predictable performance for many simultaneous users or automated services.

Less useful:

- one single file, unless future chunk-splitting is added
- tiny one-off files
- many tiny files unless the NAS can extract archives locally

## Strategy

The default `auto` strategy avoids overhead:

- one file uses single-path copy
- multi-file transfers use dual path at 256 MB or more
- 16 files or more use dual path
- smaller transfers use single-path copy

You can force a mode with `--strategy single` or `--strategy dual`.

## Setup

Requirements:

- Linux client with Python 3.10 or newer
- NFS client tools, usually provided by `nfs-common` or `nfs-utils`
- `iproute2` tools for route setup
- sudo access for route and mount preparation

Optional Python virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

The project currently uses only the Python standard library, so `requirements.txt` is intentionally minimal.

Copy the example config:

```bash
cp examples/dual-speed.config.example.json dual-speed.config.json
```

Edit the placeholders:

```json
{
  "logical_root": "/your-public-nas-mount-here",
  "mount_root": "/mnt/.dual-speed",
  "export_path": "/your-nas-export-here",
  "nfs_version": "3",
  "paths": [
    {
      "name": "a",
      "nas_ip": "your-nas-ip-a",
      "iface": "your-client-nic-a",
      "src_ip": "your-client-ip-a"
    },
    {
      "name": "b",
      "nas_ip": "your-nas-ip-b",
      "iface": "your-client-nic-b",
      "src_ip": "your-client-ip-b"
    }
  ]
}
```

Install locally:

```bash
install -m 0755 bin/dual-speed ~/.local/bin/dual-speed
install -m 0755 bin/dual-speed-testcases ~/.local/bin/dual-speed-testcases
```

## Commands

Check status:

```bash
dual-speed --config dual-speed.config.json status
```

Prepare routes and hidden mounts:

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

Dry run without changing files:

```bash
dual-speed --config dual-speed.config.json upload /local/source /your-public-nas-mount-here/target --dry-run --no-prepare
```

Speed test:

```bash
sudo dual-speed --config dual-speed.config.json speedtest --size-mb 1024
```

Benchmark:

```bash
sudo DUAL_SPEED_BIN=./bin/dual-speed ./bin/dual-speed-testcases benchmark --nas-dir /your-public-nas-mount-here/.dual_speed_benchmark
```

Validation test cases:

```bash
dual-speed-testcases --scale quick
```

## Operating Guidelines

Use `dual-speed` for large NAS transfers, especially model folders, datasets, backups, media folders, or other large multi-file jobs. For tiny one-off files, ordinary `cp` or `rsync` is usually simpler and faster.

Keep public paths pointed at the normal NAS mount, such as `/your-public-nas-mount-here`. Hidden internal mount paths are implementation details and should not be used directly in day-to-day transfer commands.

`upload` and `download` default to `--strategy auto`:

- one file uses single-path copy
- multi-file transfers use dual path at 256 MB or more
- 16 files or more use dual path
- smaller transfers use single-path copy to avoid overhead

Use `--strategy dual` to force both links for benchmarks or known-large jobs. Use `--strategy single` for tiny or latency-sensitive transfers.

Prefer `--verify size` for normal large transfers. Use `--verify sha256` when integrity matters more than speed. Resume is enabled by default and skips existing same-size destination files.

After important transfers, record the total size, elapsed time, combined MB/s, verification mode, and whether both configured NICs showed traffic. If traffic only appears on one NIC, fall back to ordinary NAS transfer and check routing, mounts, and NFS version before relying on acceleration.

## Small Files And Compression

Many tiny files are often limited by metadata operations rather than bandwidth. Packing can help only when extraction runs on the NAS host itself through SSH or an API. Packing and then extracting through NFS from the client usually wastes network I/O because the archive is read back and the extracted files are written over the network again.

Compression is workload-dependent. Do not compress model files, archives, images, audio, video, or random-like data by default. Consider compression for large text, CSV, JSON, logs, or zero-heavy data only when a probe shows enough size reduction to offset compression time.

## Notes

NFSv3 is used because NFSv4 clients can merge multiple NAS IPs into a single session, defeating dual-path routing in some setups.

Commands that modify routes or mounts normally require sudo.

## License

MIT
