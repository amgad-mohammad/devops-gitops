# Docker Volumes

## Why Volumes Exist

By default, a Docker container is **not persistent**. When you stop, remove,
or replace a container, any data written inside it is lost.

Data baked into an **image** is also effectively read-only at runtime — you
shouldn't rely on modifying a running container's filesystem as a way to
share changes with other users of that image.

This is a problem for anything that needs to persist, like a database. If
the container is removed and a new one takes its place, the data must still
exist on disk.

**Docker volumes** solve this. They are managed by Docker itself and stored
on the host machine's filesystem. On Linux, they live at:

```
/var/lib/docker/volumes
```

---

## Types of Mounts

Docker supports three ways to persist or share data between the host and a
container:

1. Anonymous Volumes
2. Named Volumes
3. Bind Mounts

---

### 1. Anonymous Volumes

The simplest type. You specify a path *inside* the container, but give it no
name — Docker creates one automatically and assigns it a long, random ID.

- Managed entirely by Docker (stored under `/var/lib/docker/volumes`)
- Typically used by a single container; not meant to be shared
- Easy to lose track of, since the name is auto-generated

```bash
docker run -d -v /path/inside/container IMAGE_NAME
```

**Real-world use case:**
A one-off container that needs *some* writable scratch space during its
run, but where you don't care about the data afterward and won't reuse it
in another container — e.g. a temporary build cache for a CI job container
that gets torn down right after.

---

### 2. Named Volumes

Like anonymous volumes, but you explicitly name them — making them easier to
find, reuse, and share across multiple containers.

- Managed by Docker, same storage location as anonymous volumes
- Can be attached to multiple containers at once (shared)
- The recommended way to persist data like databases

```bash
docker volume create VOLUME_NAME
docker run -d -v VOLUME_NAME:/path/inside/container IMAGE_NAME
```

**Real-world use case:**
**Databases.** This is the textbook example. Run a Postgres or MySQL
container with its data directory (e.g. `/var/lib/postgresql/data`) mapped
to a named volume. You can stop, remove, upgrade, or recreate the container
entirely — as long as you reattach the same named volume, all your tables
and rows are still there. It's also how you'd let a backup container read
the same data as your live database container, since named volumes can be
shared.

```bash
docker volume create pgdata
docker run -d --name postgres-db -v pgdata:/var/lib/postgresql/data postgres
```

---

### 3. Bind Mounts

Not technically a "Docker volume" — it's a direct link between a path on the
**host** filesystem and a path inside the container.

- **Bidirectional**: changes made on the host appear in the container, *and*
  changes made in the container appear on the host
- Not managed by Docker — you're responsible for the host path
- Useful for local development (e.g. mounting your source code into a
  container for live reload)

```bash
docker run -d -v $HOST_PATH:/path/inside/container IMAGE_NAME
```

**Real-world use cases:**

- **Local development with live reload.** Mount your project's source code
  folder on the host into the container (e.g. a Node.js or Python app). Edit
  a file in your IDE, and the change is instantly visible inside the running
  container — no rebuild needed.

  ```bash
  docker run -d -v $(pwd):/app node:18
  ```

- **Serving/hosting a website.** Mount your HTML/CSS/JS files (or an Nginx
  config) from the host straight into a web server container, so you can
  edit content on the host and see it reflected immediately by the running
  server.

  ```bash
  docker run -d -v $(pwd)/site:/usr/share/nginx/html nginx
  ```

- **Reading host logs or config from a container**, or the reverse — a
  container writes application logs to a host path (e.g. `/var/log/myapp`)
  so log-shipping tools running directly on the host (like Filebeat) can
  pick them up without needing to reach into the container.

⚠️ Note: bind mounts tie your setup to a specific host path, which makes
them less portable — this is why they're recommended mainly for local
development rather than production deployments.

---

## Quick Comparison

| Type              | Named? | Shared across containers? | Managed by Docker? | Direction        |
|-------------------|--------|-----------------------------|---------------------|-------------------|
| Anonymous Volume  | No     | No (typically single-use)   | Yes                 | Container ↔ Volume |
| Named Volume      | Yes    | Yes                          | Yes                 | Container ↔ Volume |
| Bind Mount        | N/A    | Yes (any container pointed at the path) | No | Host ↔ Container (both ways) |

| Type              | Best real-world fit |
|-------------------|----------------------|
| Anonymous Volume  | Temporary scratch space for a short-lived, single-use container (e.g. a CI build cache) |
| Named Volume      | Databases and any data that must survive container recreation and possibly be shared (e.g. Postgres/MySQL data dir) |
| Bind Mount        | Local development with live reload, hosting/serving files straight from the host, feeding host-side logs to a container or vice versa |