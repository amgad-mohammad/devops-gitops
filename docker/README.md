# What Is Docker?

## The Linux Foundation

The **Linux kernel** is the core layer that talks to the OS and hardware.
It's the same kernel across every Linux distribution (Red Hat, Ubuntu,
CentOS, etc.) — each distro just adds its own applications and tooling on
top of it. This is why Docker can run on any Linux OS: it's built directly
on kernel features, not on a specific distro.

## Docker Didn't Invent Containers

Containers already existed before Docker — Linux had containerization
capabilities via two kernel features:

- **Namespaces** — isolate what a process can *see* (its own filesystem,
  process list, network stack, hostname, etc.), so each container thinks
  it's the only thing running on the machine.
- **cgroups (control groups)** — limit and account for what a process can
  *use* (CPU, memory, disk I/O), so one container can't starve the others.

Together these gave you Linux Containers (LXC) — but configuring them by
hand was complex and error-prone.

**Docker's contribution** wasn't inventing containers — it was building a
much simpler, developer-friendly way to package and run applications on
top of those same kernel features (images, a CLI, a straightforward
workflow) — created by Docker, Inc.

## How It Works on Linux

On a Linux machine, you install the **Docker Engine** (the background
server/daemon) and the **Docker CLI**. You type commands in the CLI, which
send requests to the Docker Engine, which then talks to the Linux kernel
(namespaces + cgroups) to actually create and manage containers.

By default, Docker runs as **root only**. To let a non-root user run Docker
commands without `sudo`, you add that user to the `docker` group:

```bash
sudo usermod -aG docker $USER
```

## Docker on Windows & Mac

Neither Windows nor macOS ships the Linux kernel, and Docker's isolation
mechanisms (namespaces, cgroups) are Linux kernel features — they don't
exist natively on those OSes. So Docker Desktop runs a lightweight Linux
environment underneath to make it work:

- **Windows** — via **WSL2** (Windows Subsystem for Linux), which runs a
  real, lightweight Linux kernel inside a managed virtual environment.
- **macOS** — via a lightweight Linux **VM** (built on Apple's
  virtualization framework).

Either way, your containers are still Linux containers running on an
actual Linux kernel — Docker Desktop just handles spinning that kernel up
for you so it feels native.

## How to Download & Install It

### Linux

On Linux you don't need Docker Desktop — install the **Docker Engine**
directly, straight from Docker's official package repository (recommended
over any curl-script "convenience install," which Docker itself says is
only meant for quick evaluation/dev use):

```bash
# Example for Ubuntu/Debian — see docs.docker.com/engine/install for your specific distro
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Then add yourself to the `docker` group (see above) so you don't need
`sudo` for every command. (A Docker Desktop app also exists for Linux via
`.deb`/`.rpm`/AUR packages if you'd rather have the GUI.)

### Windows

1. Go to **docker.com** and download **Docker Desktop for Windows**.
2. Run the installer — it will offer to enable **WSL 2** for you if it
   isn't already set up (this is the Linux kernel piece from above).
3. Restart Windows if prompted, then launch Docker Desktop from the Start
   menu and wait for the engine to start.

### macOS

1. Go to **docker.com** and download **Docker Desktop for Mac** — pick the
   build that matches your chip: **Apple Silicon** (M1/M2/M3/M4) or
   **Intel**. Check via Apple menu → About This Mac if you're unsure.
2. Open the `.dmg` and drag Docker to your Applications folder.
3. Launch Docker Desktop from Applications — it spins up the Linux VM
   underneath automatically. Once it's running, `docker` works from the
   Terminal.

Docker Desktop is free for personal use, education, small businesses, and
non-commercial open source; larger commercial use requires a paid plan.

---