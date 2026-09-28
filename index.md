# Lab GPU Server

## Workstation specs

| Component | Spec |
|---|---|
| Hostname | `rtx-pro-6000-blackwell` |
| GPU | 1 × NVIDIA RTX PRO 6000 Blackwell Workstation (98 GB VRAM) |
| CPU | Intel Core Ultra 9 285K (24 cores) |
| RAM | 64 GB |
| Storage | 4 TB NVMe SSD |

---

# Accessing the GPU server

This guide explains how to connect to the GPU server. The recommended connection method is remote SSH over the Tailscale VPN.

New users should read it once top to bottom before starting, then follow the steps in order. Throughout this guide, replace `<username>` with the account name provided.

> Currently the VPN runs under Shaun's account. For future maintenance, it will be migrated to one admin account for easier handover.

## Step 0: Request access

Before anything else, message Shaun (shaun.liew@singaporetech.edu.sg) with the email address you want to use. Shaun will then:

1. Create a user account for you on the server.
2. Send a Tailscale VPN invitation to that email address.

Access is not possible until both are done. If you do not have an account or an invite yet, contact Shaun first.

## Before you start

- Install the Tailscale VPN app: <https://tailscale.com/download>
- Join the Tailscale network using the invitation email Shaun sent.
- Make sure Tailscale is switched on on your laptop before connecting.

No SSH key needs to be created. Tailscale handles that automatically. As long as the VPN is on, the connection will work.

## Step 1: SSH into the server

Open a terminal and run the following, using the username and password Shaun provided:

```bash
ssh <username>@rtx-pro-6000-blackwell
```

You may be asked to log in to Tailscale first. Once authorized, the server login appears.

A successful login looks like this:

```text
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.17.0-23-generic x86_64)
 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
...
<username>@rtx-pro-6000-blackwell:~$
```

When the `<username>@rtx-pro-6000-blackwell:~$` prompt appears, you are logged in.

## Step 2: One-time container setup

Each user works in their own container, mounted to their own workspace folder. This setup is done once. After that, day-to-day use is just Step 3.

Create a personal workspace folder under `/workspace`:

```bash
mkdir -p /workspace/<username>
```

If this returns a permission error, ask Shaun to create the folder, since standard accounts may not have write access to `/workspace`.

Create the container from the PyTorch image (already on the server, no need to pull it):

```bash
docker run -dit \
  --name <username> \
  --gpus all \
  --restart unless-stopped \
  --shm-size=16g \
  -v /workspace/<username>:/workspace \
  nvcr.io/nvidia/pytorch:26.02-py3 \
  /bin/bash
```

What the flags do:

| Flag | Purpose |
|---|---|
| `--name <username>` | Unique, easy-to-identify container name (one per user). |
| `--gpus all` | Grants GPU access. |
| `--restart unless-stopped` | Keeps the container running across reboots, so setup only happens once. |
| `--shm-size=16g` | Raises shared memory, which PyTorch dataloaders need. |
| `-v /workspace/<username>:/workspace` | Mounts your personal folder into the container as `/workspace`. |

## Step 3: Enter the container

Every time you connect, enter your container with:

```bash
docker exec -it <username> /bin/bash
```

The prompt changes to something like:

```text
<username>@rtx-pro-6000-blackwell:~$ docker exec -it <username> /bin/bash
root@2e915b05cfe5:/workspace#
```

Once `/workspace` appears in the prompt, you are inside your container and ready to work.

## The one rule that matters

**Never run code or install packages in the host environment.** Code and packages must only be run and installed inside your container.

The host has a fragile NVIDIA driver setup, and installing things there can take the GPU down for everyone. Each container is isolated and safe to work in.

To check where you are, look at the prompt:

| Environment | Prompt |
|---|---|
| Container (safe) | `root@<random-id>:/workspace#` |
| Host (do not install here) | `<username>@rtx-pro-6000-blackwell:~$` |

## Step 4: Check things are working

Inside the container, confirm GPU access:

```bash
nvidia-smi
```

The GPU should be listed, for example:

```text
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.58.03              Driver Version: 595.58.03      CUDA Version: 13.2     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|=========================================+========================+======================|
|   0  NVIDIA RTX PRO 6000 Blac...    Off |   00000000:02:00.0  On |                  Off |
| 30%   33C    P8             13W /  300W |     503MiB /  97887MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

Anything saved under `/workspace` inside the container lives in your personal folder on the host, so it persists even if the container restarts. The PyTorch image already includes PyTorch, CUDA, and common libraries. Install additional packages inside the container as needed (for example with conda or pip).

## Step 5 (recommended): VS Code Remote-SSH

Working in VS Code is much more comfortable than a raw terminal. There are two parts: connecting to the host, then attaching to the container.

### Connect to the host

1. Install the **Remote - SSH** extension (`ms-vscode-remote.remote-ssh`).
2. Add this to `~/.ssh/config` on your laptop, replacing `<username>`:

   ```text
   Host blackwell
       HostName rtx-pro-6000-blackwell
       User <username>
   ```

3. Open the Command Palette (`Cmd/Ctrl + Shift + P`), run **Remote-SSH: Connect to Host**, and select `blackwell`.

### Attach to the container

1. Install the **Dev Containers** extension (`ms-vscode-remote.remote-containers`).
2. Open the Command Palette and run **Dev Containers: Attach to Running Container**.
3. Select `<username>` from the list.
4. Open the `/workspace` folder.

You can now edit and run code directly inside the container.

## Quick recap

1. Message Shaun with the email to use, and wait for the account and Tailscale invite.
2. Turn Tailscale on and connect.
3. `ssh <username>@rtx-pro-6000-blackwell`
4. One time only: create `/workspace/<username>` and the `<username>` container.
5. Each session: `docker exec -it <username> /bin/bash`
6. Work only inside the container. Never install anything on the host.

Each login is tested before handover, so this should work as described. For any problems, contact Shaun.
