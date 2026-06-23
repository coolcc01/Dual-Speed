# Dual-Speed

`dual-speed` is a multi-path NAS transfer tool that starts with the common dual-NIC case. It is especially useful for workstation and server motherboards with dual onboard NICs, dual 1GbE or 2.5GbE ports, add-in NICs, or any setup where a client can reach a NAS through two or more separate IP paths.

Instead of fighting with LACP, bonding, MPTCP, SMB multichannel, or switch-side link aggregation, `dual-speed` takes the simple route: split suitable file workloads at the application layer and send each group through a different mounted NAS path. For many real NAS copy jobs, this achieves the practical throughput gain people want from link aggregation without requiring complex network configuration.

The tool keeps one public NAS path for users, while internally splitting large multi-file upload and download workloads across two hidden NFSv3 mounts. This avoids confusing users with multiple mount points while still using both network links when it helps.

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

Benchmark:

```bash
sudo DUAL_SPEED_BIN=./bin/dual-speed ./bin/dual-speed-testcases benchmark --nas-dir /your-public-nas-mount-here/.dual_speed_benchmark
```

## Notes

NFSv3 is used because NFSv4 clients can merge multiple NAS IPs into a single session, defeating dual-path routing in some setups.

Packing many tiny files can help only if extraction runs on the NAS itself through SSH or an API. Packing and then extracting through NFS from the client usually wastes network I/O.

## License

MIT
