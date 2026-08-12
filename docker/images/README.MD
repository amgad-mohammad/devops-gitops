# Docker Images

## What Is an Image?

A Docker image contains everything an application needs to run: code,
runtime, libraries, and environment configuration.

Images are **read-only**. That's what makes them reusable — anyone can pull
the same image and spin up their own container on top of it without
affecting anyone else's copy.

---

## Docker UFS: Union File System

This is *how* images are actually stored — as a stack of **layers**.

- When you **run** a container, Docker adds one more layer on top: a thin,
  **writable** layer just for that container.
- When you **remove** the container, only that writable layer (and its
  changes) disappears. The underlying image layers are untouched.
- This is exactly why you need **volumes or bind mounts** for anything you
  want to survive the container being removed (see the Docker Volumes
  notes) — the container's own writable layer is not a safe place to keep
  data long-term.

**Modifying an image directly:**
You *can* save changes made inside a running container (e.g. installing
`curl` or Jenkins) as a new image using `docker commit`. You give the
result a new image name/tag — the original base image is untouched and
still available to everyone else.

```bash
docker commit -m "Added curl" my-container my-image:v2
```

⚠️ In practice, `docker commit` is rarely used. It has no record of *what*
changed or *why* — the only way to inspect it afterward is `docker diff`,
and that's not something you can hand to a teammate as documentation. If
you make several changes this way, nobody has a clear, repeatable recipe of
what was done.

This is exactly the problem a **Dockerfile** solves: a plain-text,
version-controllable recipe for building an image, instead of a black-box
`commit`.

---

## Dockerfile

A Dockerfile is a plain text file with a defined set of instructions,
executed top to bottom. Each instruction that changes the filesystem
(`RUN`, `COPY`, `ADD`) creates a new image layer.

### Key instructions

| Instruction | What it does |
|---|---|
| `FROM` | Must be the first instruction. Sets the base image to build on top of (e.g. `node:alpine`) — this is where your needed runtime/libraries come from. |
| `WORKDIR` | Sets the working directory inside the image for every instruction that follows. Creates the directory if it doesn't exist (like `mkdir -p` then `cd`). |
| `COPY` | Copies files from your **local build context** into the image. `COPY . .` = copy everything from the source folder into the current `WORKDIR`. |
| `ADD` | Does what `COPY` does, plus two extras: it can fetch a file from a **remote URL**, and it will **auto-extract** a local compressed archive (e.g. `.tar.gz`) into the destination. Note: files fetched from a remote URL are *not* auto-extracted, only local archives are. Docker's own guidance: prefer `COPY` unless you specifically need one of `ADD`'s extra behaviors. |
| `RUN` | Executes a command at build time (e.g. `npm install`). Each `RUN` instruction creates its own layer. |
| `CMD` | The **default command/arguments** for the container. Easily overridden — anything you pass after `docker run my-image ...` replaces it. |
| `ENTRYPOINT` | The **fixed command** that always runs. Harder to override (needs `docker run --entrypoint ...`). |
| `EXPOSE` | **Documentation only** — tells anyone reading the image that it's designed to listen on that port. It does **not** actually publish the port; you still need `-p` when running the container. |

### A correction worth calling out: chaining RUN commands

To combine multiple shell commands into a **single** `RUN` instruction
(so they produce **one** layer instead of several), chain them with `&&`,
not a single `&`:

```dockerfile
RUN apt-get update && apt-get install -y curl && apt-get clean
```

(A single `&` runs a command in the *background* — not what you want here.)

### CMD vs ENTRYPOINT

If you define neither, the container will start but run nothing useful —
you need at least a `CMD` or an `ENTRYPOINT`.

| Instruction(s) present | Behavior |
|---|---|
| `CMD` only | The command runs by default, and is fully replaced if you pass arguments to `docker run`. |
| `ENTRYPOINT` only | The command always runs; anything you pass to `docker run` is appended as arguments to it. |
| `ENTRYPOINT` + `CMD` | `CMD` supplies the **default arguments** to `ENTRYPOINT`. `docker run` arguments override just the `CMD` part, not `ENTRYPOINT` itself. |

Example — `ENTRYPOINT` fixes the interpreter, `CMD` gives it a default file
to run, and you can still override *which* file at runtime:

```dockerfile
ENTRYPOINT ["node"]
CMD ["src/index.js"]
```

```bash
docker run my-image                # runs: node src/index.js
docker run my-image src/other.js   # runs: node src/other.js
```

### Building the image

```bash
docker build -t my_image .
```

The `.` is the **build context** — the folder Docker sends to the build
process. Docker looks for a file named `Dockerfile` inside that context by
default (use `-f path/to/Dockerfile` to point elsewhere).

```bash
docker image ls        # or: docker images
docker run -d -p 3000:3000 my_image
```

---

## Publishing & Distributing Docker Images

A **Docker Registry** is where images are stored so they can be pulled and
pushed — think of it as a cloud host for images. Docker's own registry is
**Docker Hub**, hosting many public images (Node.js, Java, Kafka, etc.).

⚠️ By default, images you push to Docker Hub go into a **public**
repository unless you explicitly create a private one (free accounts get a
limited number of private repos) — it's the opposite of "private by
default."

**Logging in:**

```bash
docker login                       # Docker Hub (the default registry)
docker login REGISTRY_URL          # any other registry
```

**Anatomy of an image reference:**
When you run:

```bash
docker pull ubuntu
```

Docker expands that to:

```
docker pull docker.io/library/ubuntu:latest
```

- `docker.io` — the registry
- `library` — the namespace (Docker's own official images live here)
- `ubuntu` — the image name
- `latest` — the tag (defaults to `latest` if omitted)

For your own images, the namespace is your username:

```bash
docker pull docker.io/$YOUR_USERNAME/my-image
```

**Tagging and pushing:**

```bash
docker tag my_image:latest $YOUR_USERNAME/my_image:latest
docker push $YOUR_USERNAME/my_image:latest
```

You can drop `docker.io/` when pushing/pulling from Docker Hub specifically
— it's the default registry Docker assumes.

**Running it (from anywhere, once pulled):**

```bash
docker run -d -p 3000:3000 $YOUR_USERNAME/my_image:latest
```

### Other registries

If you want to push somewhere other than Docker Hub:

- Amazon **Elastic Container Registry** (ECR)
- Google **Artifact/Container Registry** (GCR)
- Azure **Container Registry** (ACR)
- **GitHub Container Registry** (GHCR)

The general flow: create a repository on that registry, authenticate
against it, then reference its URL on push/pull:

```bash
docker login ghcr.io
docker tag my_image ghcr.io/$YOUR_USERNAME/my_image:latest
docker push ghcr.io/$YOUR_USERNAME/my_image:latest
```

⚠️ Cloud registries (ECR, GCR, ACR) usually authenticate through their own
CLI (`aws ecr get-login-password`, `gcloud auth configure-docker`, etc.)
rather than a plain username/password `docker login`.