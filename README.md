# GPU Server Bootstrap (`setup.sh`)

This repository contains a single bootstrap script:

- `setup.sh`

It turns a fresh GPU server into a production-friendly multi-user Docker platform where each user works in an isolated private container.

## Deployment Model

After `setup.sh` finishes:

- Host SSH allows **root only**.
- Normal users do **not** access host shell directly.
- Each user gets a dedicated container (`dev-<username>`) with SSH via host high ports (`20000+`).
- User data persists on host data disk:
  - `/data/home/<username>` -> container `/root`
  - `/data/share` -> container `/share`
- GPU defaults:
  - `--gpus all`
  - `NVIDIA_DRIVER_CAPABILITIES=all` (compute + rendering support)
- Shared memory defaults:
  - `--shm-size 256g`
- Container restart policy:
  - `unless-stopped`

## Preconditions

`setup.sh` assumes:

1. You run it as `root`.
2. Data disk is mounted at `/data`.
3. Server has full outbound network access.
4. Host NVIDIA driver is installed and `nvidia-smi` works.
5. OS is apt-based (Ubuntu/Debian family).

## Quick Start

```bash
chmod +x setup.sh
sudo ./setup.sh
```

## What `setup.sh` Does

1. Installs required packages (`docker.io`, `jq`, `curl`, `gpg`, etc.).
2. Installs NVIDIA Container Toolkit and configures Docker runtime.
3. Detects a **working CUDA base image automatically**:
   - Tries multiple CUDA tags.
   - Tries multiple registry mirrors.
   - Verifies each candidate with `docker run --gpus all ... nvidia-smi -L`.
4. Writes host SSH policy (`AllowUsers root`).
5. Prepares runtime directories under `/data`.
6. Writes `/root/private-dev-image.Dockerfile` using the detected CUDA base.
7. Builds `private-dev:ssh-gpu`.
8. Generates operation scripts in `/root`:
   - `/root/creater_user.sh`
   - `/root/delete_user.sh`
   - `/root/rebuild_user_container.sh`
   - `/root/rebuild_user_container_keep_data.sh`
   - `/root/user_storage_usage.sh`
   - `/root/blame_gpu_use.sh`
9. Writes metadata file:
   - `/data/docker-private-users/platform.env`

## Day-2 Operations (Generated Scripts)

### Create a user container

```bash
/root/creater_user.sh <username> <pubkey_or_pubkey_file> <server_host>
```

### Delete a user container

```bash
/root/delete_user.sh <username>
/root/delete_user.sh <username> --purge-home
```

### Rebuild (reset mode)

Resets non-mounted container layer, keeps `/root` and `/share` mounts:

```bash
/root/rebuild_user_container.sh <username>
```

### Rebuild (keep-data mode)

Preserves mounted data and container writable-layer data (snapshot + recreate):

```bash
/root/rebuild_user_container_keep_data.sh <username> --shm-size 256g
```

### Storage accounting

```bash
/root/user_storage_usage.sh
```

### GPU process attribution

```bash
/root/blame_gpu_use.sh
```

## Post-Install Validation

```bash
docker images | grep private-dev:ssh-gpu
sshd -T | egrep 'allowusers|permitrootlogin'
nvidia-smi -L
```

Optional: create one test user and verify login + GPU inside container.

## Idempotency and Safety Notes

- Running `setup.sh` again is supported and generally safe (idempotent-oriented).
- Rebuild scripts terminate running jobs in the target container.
- Keep-data rebuild keeps user data but still restarts container runtime state.
- This model does not enforce per-user disk quota by default.

## Scope

This project focuses on practical single-host multi-user operations with low overhead.  
If you need strict tenant isolation, fairness scheduling, or enterprise policy controls, add orchestration/governance layers on top.

