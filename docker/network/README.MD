# Docker Networks

## Why Networks Exist

Docker containers run in an isolated environment. If you need to reach a
container from the **host**, or connect it to **another container**, you
can't do that directly — there needs to be a virtual network that handles
it.

By default, Docker creates a virtual network called `bridge` (driver type:
`bridge`). Every container gets an IP address and runs inside that
network's gateway/subnet.

The **default bridge network has no DNS-based service discovery** — you can
only reach another container by its **IP address**, not its name.

When you create your **own** network (of type bridge), Docker gives it a
built-in DNS resolver along with the usual IP/gateway setup. Now containers
on that network can reach each other **by container name**, not just IP.
This is why real projects almost always use a user-defined network instead
of the default one.

```bash
# Create a network
docker network create MY_NETWORK

# Attach a running container to a network
docker network connect MY_NETWORK MY_CONTAINER

# Create a new container already attached to a network
docker run --network MY_NETWORK IMAGE_NAME
```

---

## Types of Networks

Docker supports several network types, each controlling how a container
talks to the host, other containers, and the outside world.

1. Bridge
2. Host
3. Macvlan
4. None

---

### 1. Bridge Network

The default network type. Docker creates a private internal network
(backed by a virtual switch, `docker0` on Linux) and gives each container
its own IP inside it. Containers can reach the outside world (outbound
internet) through NAT, and — if you're on a **user-defined** bridge —
can reach each other **by name** thanks to Docker's built-in DNS.

```bash
docker network create my-bridge
docker run -d --network my-bridge --name web nginx
```

**Real-world use case:**
The standard choice for most single-host setups — e.g. a web app container
and a database container that need to talk to each other by name (this is
exactly what `docker compose` sets up for you automatically).

⚠️ Best practice: avoid the *default* bridge network for real projects
(no DNS, weaker isolation) — always create your own bridge network instead.

---

### 2. Host Network

The container uses the **host's network stack directly** — no isolation,
no separate IP. It's as if the process were running natively on the host.
Containers using host networking can reach each other (and be reached) via
`localhost:port`, since everything shares the same network namespace.

```bash
docker run --network host IMAGE_NAME
```

**Real-world use case:**
Performance-sensitive tools where you want to avoid the small overhead of
Docker's NAT/port-mapping layer — e.g. a network monitoring or
packet-capture tool, or a high-throughput proxy like HAProxy.

⚠️ **Linux only.** On Docker Desktop for Windows/Mac, containers run inside
a lightweight VM, so `host` networking gives you the *VM's* network, not
your actual machine — `localhost` won't point to your Mac/Windows host as
you'd expect.

---

### 3. Macvlan Network

Gives each container its own **MAC address and IP address** taken directly
from your physical network's range — Docker doesn't create a virtual
bridge here. The container appears on the LAN as if it were its own
physical device, fully reachable from other machines on that network.

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 my-macvlan

docker run -d --network my-macvlan --ip 192.168.1.130 IMAGE_NAME
```

**Real-world use case:**
Legacy applications or IoT/home-lab devices that need to be discoverable
directly on your LAN by other physical devices — e.g. running Pi-hole or
Home Assistant so it shows up on the network like any other appliance.

⚠️ Kernel limitation: by default the **host itself cannot talk to** a
macvlan container directly (you'd need an extra macvlan sub-interface on
the host to bridge that gap).

---

### 4. None Network (Mode)

Full isolation. The container gets no network interfaces besides loopback
— no inbound connections, no outbound connections, nothing.

```bash
docker run --network none IMAGE_NAME
```

**Real-world use case:**
Running something you deliberately don't want touching a network at all —
e.g. analyzing a potentially malicious file/binary in a sandbox, or a
pure batch/compute job that only reads local files and has no business
making network calls.

---

## Which One Should I Use?

Ask these in order:

```
Need full isolation (no network at all)?
│
├─ Yes → None      (fully sandboxed, no in/out traffic)
│
└─ No  → Need host-level performance (share host's network directly)?
         │
         ├─ Yes → Host     (max performance, no isolation)
         │
         └─ No  → Need its own IP visible on the physical LAN?
                  │
                  ├─ Yes → Macvlan   (acts like a real device on your LAN)
                  │
                  └─ No  → Bridge (default)   (normal container-to-container setup)
```

- **None** — you deliberately don't want any network access (e.g. sandboxing
  untrusted code).
- **Host** — you need raw performance and don't need isolation (e.g. a
  packet-capture tool).
- **Macvlan** — the container needs to look like a real device on your LAN
  (e.g. a home-lab appliance like Pi-hole).
- **Bridge** — the default, everyday choice: containers that need to talk
  to each other (and the internet) but don't need any of the above.

---

## Quick Comparison

| Type    | Isolated from host? | Container-to-container DNS? | Reachable from outside LAN? | Typical use |
|---------|----------------------|-------------------------------|-------------------------------|-------------|
| Bridge (user-defined) | Yes | Yes | Only via published ports | Multi-container apps (app + DB) |
| Host    | No (shares host network) | N/A (same as host) | Same as host | Performance-critical tools |
| Macvlan | Yes (own IP/MAC) | No | Yes, natively | LAN-visible devices, legacy apps |
| None    | Full isolation | No | No | Sandboxing, no-network jobs |