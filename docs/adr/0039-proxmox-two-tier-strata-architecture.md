# ADR-0039: Proxmox VE Two-Tier Split (GPU LXC + Docker App VM) & Strata Primary Architecture

- **Status:** Accepted (Phase 1 & Phase 2 Complete)
- **Date:** 2026-11-08 (Updated)
- **Related:** [ADR-0006](0006-hybrid-deployment-model.md) (hybrid deploy), [ADR-0032](0032-qwen3.8-flash-next-strata-evaluation.md) / [ADR-0033](0033-flash-next-strata-in-llama-swap.md) (Strata engine)

## Context
This fork adapts the reference bare-metal `/srv/ai` server architecture to a Proxmox VE (`pve`, `192.168.1.10`) platform with the following hardware and progression:
- **Host CPU & Memory:** AMD Ryzen 7 5800X3D (8C/16T, 96 MB 3D V-Cache), 64 GB DDR4-3600.
- **Storage:**
  - `nvme0n1`: Samsung SSD 970 EVO Plus 1TB (Proxmox ZFS `rpool`, ZFS ARC capped to 3.0 GB max / 1.0 GB min via `/etc/modprobe.d/zfs.conf`).
  - `nvme1n1`: Samsung SSD 990 EVO Plus 1TB (dedicated fast PCIe 4.0 x4 model storage drive).
- **GPUs:**
  - **Initial (Phases 1–3):** NVIDIA GeForce RTX 3080 12GB (`sm_86` Ampere).
  - **Target Upgrade (Phases 4–5):** 2× Tesla V100 SXM2 32GB (`sm_70` Volta, 64 GB aggregate HBM2) mounted on a single PCIe x16 NVLink carrier board (bifurcated to x8/x8 on the host, with **300 GB/s onboard NVLink traces** between the SXM2 modules).

### Live Inspection Findings (2026-11-07)
1. **Why `Strata-data` Must Move Off the ZFS Root Volume onto `ext4` (`nvme1n1`):**
   - Strata `0.1.57` (`Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS`, 128K context, `dflash2` drafter) is currently installed under `/root/.local/share/strata` inside CT 100 (`192.168.1.11`), consuming 114 GB of the 150 GB ZFS root subvolume (`rpool/data/subvol-100-disk-0`).
   - Inspection of `/root/.local/state/strata/launch/iq3_xxs-128k.engine.log` revealed a critical interaction between `--mmap-experts`, ZFS, and the 3 GB ZFS ARC cap:
     - Strata's file tier expects the 36.0 GiB cold expert pack (`experts.bin`) to be cached in the 44.3 GiB available Linux page cache (`MemAvailable`).
     - Because `Strata-data` currently resides on ZFS (`rpool`), Linux page cache (`buff/cache`) is bypassed in favor of ZFS ARC (which is capped at 3 GB). As a result, Strata logged:
       `file tier I/O this request: the OS read 9742.1 MB from storage (8709 major faults) for 18828.0 MB of expert reads; page cache held 0% of the decode's reads (EWMA)`.
   - Formatting `nvme1n1` (Samsung 990 EVO Plus 1TB) as **`ext4`** (`/mnt/ai-data` on `pve`, bind-mounted into CT 100 at `/srv/ai/models`) and relocating `Strata-data` there allows the Linux kernel to cache the entire 36.0 GiB expert pack directly in the 44+ GiB Linux page cache, eliminating major page faults during decode and reclaiming 114 GB on `rpool`.
2. **Container Configuration Gaps Found on CT 100:**
   - `ulimit -l` (`max locked memory`) inside CT 100 is currently `8192` KB (8 MB); adding `lxc.prlimit.memlock: unlimited` to `/etc/pve/lxc/100.conf` unlocks pinned host memory (`cudaMallocHost` / `mlock`).
   - CT 100 is currently assigned `cores: 6` out of the 5800X3D's 8 physical cores (`16` threads), leaving `strata aux cpus: none spare`. Increasing CT 100 to `cores: 8` (or `12`) gives Strata's AVX2 `CPU_IQ_ALL_EXACT1` worker pool and `strata-vision` full core coverage.
   - `/root/.local/state/strata/launch/iq3_xxs-128k.engine.log` showed `strata-vision` falling back to CPU (`ggml_cuda_init: failed to initialize CUDA: no CUDA-capable device is detected` because `"gpu": false` in the launch config hides CUDA from the vision subprocess).

## Decision
1. **Two-Tier Guest Split (Avoiding Nested Docker-in-LXC):**
   - **CT 100 (`ai-server`, `192.168.1.11`, unprivileged LXC, 48–50 GB RAM ceiling):** Dedicated GPU appliance running Strata (primary inference engine), `llama-swap`, ComfyUI, and `comfyui-mcp` natively via systemd.
   - **VM 101 (`ai-app`, `192.168.1.12`, KVM Debian 12, 3.5 GB static RAM with `balloon: 0`, `cpu: host`):** Dedicated Docker Compose appliance running LiteLLM (`:4000`), Open WebUI (`:3000`), `mcpo` (`:8000`), and SearXNG.
2. **Engine Tiering:**
   - **Primary Engine:** Strata (`IQ3_XXS` on RTX 3080 12GB today; dual V100 SXM2 with 300 GB/s NVLink in Phase 4).
   - **Secondary/Dense Engine on V100s:** Volta-optimized `llama-superfast-v100` (`BC=8` FlashAttention tiling and Volta WMMA kernels), avoiding vanilla upstream `llama.cpp`'s ~40% Volta prompt-processing penalty.
3. **Storage & Networking:**
   - Format `/dev/nvme1n1` as `ext4` (`LABEL=ai-data`), mount at `/mnt/ai-data` on `pve`, bind-mount to `/srv/ai/models` in CT 100, and relocate `Strata-data` (114 GB) there via symlink so Strata's `mmap-experts` uses the Linux page cache.
   - Run Tailscale as a subnet router on `pve` (`192.168.1.10`) advertising `192.168.1.0/24`.
   - Restrict CT 100 backend router ports (`:9090`, `:9000`/`:9001`) to `127.0.0.1/8` and `192.168.1.12/32` (VM 101).

## Consequences
- Reclaims 114 GB from the ZFS root pool (`rpool`) and unlocks the 44 GiB Linux page cache for Strata's 36 GiB `experts.bin` file tier.
- Isolates Docker Compose on a standard KVM kernel (VM 101) with native `overlay2` while preserving near-zero-overhead GPU execution in CT 100.

## Verified (2026-11-08)
- **KVM Virtualization:** AMD SVM enabled in 5800X3D BIOS; VM 101 running Debian 13 (`6.12.115-cloud-amd64`) with 3.5 GB static RAM and 2 vCPUs (`cpu: host`).
- **Docker Compose Stack:** LiteLLM (`:4000`), Open WebUI (`:3000`), SearXNG (`127.0.0.1:8888`), `mcpo` (`:8000`), `autoheal`, and `docker-socket-proxy` all healthy and running.
- **End-to-End Inference:** LiteLLM gateway routes client chat completions over private bridge (`192.168.1.12:4000` -> `192.168.1.11:8080`) to Strata, achieving 30.2 tok/s prompt prefill, 22.2 tok/s generation, and 100% DFlash2 speculative draft acceptance on RTX 3080.
- **Web Interface:** Open WebUI accessible on `http://192.168.1.12:3000` with models `coding`, `chat`, `flash`, and `qwen3.8-flash-next-iq3_xxs`.
