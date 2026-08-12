# Docker Compose

## What It Is

A tool for defining and running **multi-container** applications from a
single YAML file — common in microservice architectures, where an app is
split into several services (e.g. an Order Service, a Payment Service, a
Gateway Service) that need to run and talk to each other together.

⚠️ Correction: it's not "a service developed in Python" anymore.
- **`docker-compose`** (hyphenated) was the original standalone tool,
  written in **Python** — it's now legacy/deprecated.
- **`docker compose`** (space, no hyphen) is the current version, a **Go**
  plugin built directly into the Docker CLI. This is what you should be
  using today.

With a `docker-compose.yml` (or `compose.yaml`) file, you define each
service, which one gets exposed to the outside world (e.g. a gateway on a
published port) and which stay internal-only, plus any **networks** and
**volumes** they share — Docker creates and wires up all the containers for
you. No manual `docker run` per container.

---

## Install

```bash
# Modern way — comes bundled with Docker Desktop / Docker Engine, no separate install needed.
# If you specifically need the legacy standalone binary:
brew install docker-compose
```

---

## Example `docker-compose.yml`

```yaml
services:
  auth:
    restart: always          # restart this container automatically if it stops/crashes
    build: ./auth            # build the image from the Dockerfile in ./auth
    depends_on:
      - db                   # start "db" before this service
    environment:              # env vars passed into the container
      DB_HOST: db
      DB_USER: root
      DB_PASSWORD: my-secret-pw
      DB_NAME: authdb
    networks:
      - app-net               # attach to the app-net network (so it can reach other services by name)

  ui:
    restart: always
    build: ./UI
    depends_on:
      - auth                  # wait for auth...
      - weather                # ...and weather to start first
    environment:
      AUTH_HOST: auth
      AUTH_PORT: 8080
      WEATHER_HOST: weather
      WEATHER_PORT: 5000
    ports:
      - "3000:3000"           # publish container port 3000 to host port 3000 (this is the exposed service)
    networks:
      - app-net

  weather:
    restart: always
    build: ./weather
    environment:
      APIKEY: 6d92c50bdamsh81137f3b87ace1fp1d53eejsnfe818b9dbc83
    networks:
      - app-net                # internal only — no "ports:", so not reachable from outside

  db:
    restart: always
    image: mysql:8.0.25         # use a prebuilt image instead of building one
    environment:
      MYSQL_ROOT_PASSWORD: my-secret-pw
      MYSQL_DATABASE: authdb
    networks:
      - app-net
    volumes:
      - ./db-data:/var/lib/mysql   # bind mount: persist MySQL's data folder on the host

networks:
  app-net:
    driver: bridge            # user-defined bridge network → services can reach each other by name

volumes:
  db-data:                    # declares db-data as a named volume (unused here since db uses a bind mount above,
                               # but declaring it lets other services reference it as a named volume if needed)
```

⚠️ Small note on the example: `db` mounts `./db-data:/var/lib/mysql` as a
**bind mount** (host path on the left), while the `volumes:` section at the
bottom declares `db-data` as a **named volume**. Both aren't used together
here — if the intent was a named volume, the service should instead use
`db-data:/var/lib/mysql` (no `./`).

---

## Do You Need a Dockerfile Per Service?

It depends on whether a service is **built** or just **pulled**:

- Services using `build: ./folder` (your own microservices, e.g. `auth`,
  `ui`, `weather`) need a **Dockerfile** inside that folder — Compose has
  no other way to turn your source code into an image.
- Services using `image: ...` (third-party infrastructure you didn't
  write — a database, Redis, Nginx, etc., like `db` above) need **no**
  Dockerfile at all. You're just pulling an existing image.

## Project Structure

`docker-compose.yml` lives at the **root** of the project, one level above
all the service folders. This matters because every `build: ./auth`-style
path in the compose file is resolved **relative to where the compose file
sits**. Move the compose file somewhere else and those paths break.

```
project/
├── docker-compose.yml    ← must sit here, at the root
├── auth/
│   └── Dockerfile         ← your code, needs one
├── ui/
│   └── Dockerfile         ← your code, needs one
├── weather/
│   └── Dockerfile         ← your code, needs one
```

Each `build: ./auth` means "look in the `auth` folder, right next to me,
for a Dockerfile and build from there." The `db` service, by contrast, has
no folder or Dockerfile at all — it just references `image: mysql:8.0.25`.

---

```bash
docker compose up -d      # create + start all services, in the background
docker compose ps         # show status of the running services
docker compose down       # stop and remove all services (containers + default network)
```

(You'll still see `docker-compose` with a hyphen in older tutorials — same
idea, just the legacy command.)