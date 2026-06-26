# Dockerfile Instructions

A practice collection covering every major Dockerfile instruction, each with a minimal working example. Part of the [`docker-in`](https://github.com/Naga-Sai-Prasanna/docker-in) repo — each section below corresponds to a folder of the same name in `dockerfiles/`.

## Contents

- [FROM](#from)
- [ARG](#arg)
- [RUN](#run)
- [CMD](#cmd)
- [ENTRYPOINT](#entrypoint)
- [COPY](#copy)
- [ADD](#add)
- [ENV](#env)
- [EXPOSE](#expose)
- [LABEL](#label)
- [ONBUILD](#onbuild)
- [USER](#user)
- [WORKDIR](#workdir)


---

## FROM

The `FROM` instruction sets the base image for the build. Every valid Dockerfile must start with a `FROM` instruction (aside from optional `ARG` lines used to parameterize it).

### Syntax

```dockerfile
FROM <image>[:<tag>] [AS <stage-name>]
```

### Example

```dockerfile
FROM alpine:3.19

# Multi-stage build example
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

### Usage

```bash
docker build -t from-example .
docker run --rm from-example
```

### Notes

- Always pin a specific tag (e.g. `alpine:3.19`, not just `alpine`) for reproducible builds — `latest` can change underneath you.
- Prefer smaller base images (`alpine`, `-slim` variants, or `distroless`) to reduce image size and attack surface where the base OS tooling isn't needed.
- `AS <stage-name>` enables multi-stage builds, letting you copy only the final build artifacts into a clean, minimal final image — keeping build tools out of the shipped image.

---

## ARG

The `ARG` instruction defines a build-time variable that users can pass via `docker build --build-arg`. Unlike `ENV`, `ARG` values are **not** available in the running container unless explicitly passed through to an `ENV` instruction.

### Syntax

```dockerfile
ARG <name>[=<default value>]
```

### Example

```dockerfile
FROM alpine:3.19

ARG APP_VERSION=1.0.0
RUN echo "Building version $APP_VERSION"
```

### Usage

```bash
# Use the default value
docker build -t arg-example .

# Override it at build time
docker build --build-arg APP_VERSION=2.0.0 -t arg-example .
```

### Notes

- `ARG` declared before the first `FROM` is only usable in `FROM` lines themselves (e.g. to parameterize the base image tag) and is **not** available later in the build unless re-declared after `FROM`.
- To make a build-time value available inside the running container, pass it into `ENV`:

  ```dockerfile
  ARG APP_VERSION=1.0.0
  ENV APP_VERSION=$APP_VERSION
  ```

- Avoid putting secrets in `ARG` — build-arg values are visible in the image history (`docker history`) and in build logs.

---

## RUN

The `RUN` instruction executes a command during the image build, and commits the resulting filesystem changes as a new image layer. It's how you install packages, create directories, or run any setup needed before the container starts.

### Syntax

```dockerfile
RUN ["executable", "param1", "param2"]   # exec form
RUN command param1 param2                 # shell form
```

### Example

```dockerfile
FROM alpine:3.19

RUN apk add --no-cache curl
RUN mkdir -p /usr/src/app
```

### Usage

```bash
docker build -t run-example .
docker run --rm run-example curl --version
```

### Notes

- Each `RUN` instruction creates a new image layer — chaining related commands with `&&` reduces the number of layers and keeps the image smaller:

  ```dockerfile
  RUN apk add --no-cache curl \
      && mkdir -p /usr/src/app
  ```

- Clean up package manager caches in the same `RUN` step that installs them, otherwise the cache is baked permanently into that layer even if removed in a later `RUN`:

  ```dockerfile
  RUN apk add --no-cache curl   # --no-cache avoids leaving apk's index behind
  ```

- `RUN` executes at **build time**, unlike `CMD`/`ENTRYPOINT`, which define what runs at **container start time**.

---

## CMD

The `CMD` instruction sets the default command that runs when a container starts from the image. Unlike `RUN`, which executes during the build, `CMD` only takes effect at `docker run` time — and it's overridden if a command is supplied on the `docker run` command line.

### Syntax

```dockerfile
CMD ["executable", "param1", "param2"]   # exec form (preferred)
CMD command param1 param2                 # shell form
```

### Example

```dockerfile
FROM alpine:3.19

CMD ["echo", "Hello from CMD"]
```

### Usage

```bash
docker build -t cmd-example .

# Runs the default CMD
docker run cmd-example

# Overrides CMD entirely
docker run cmd-example echo "this overrides CMD"
```

### Notes

- Only the **last** `CMD` in a Dockerfile takes effect — earlier ones are ignored.
- Prefer the exec form (`["executable", "arg"]`) over the shell form — the shell form runs the command through `/bin/sh -c`, which adds an extra process layer and doesn't forward signals (like `SIGTERM`) cleanly to the actual process.
- `CMD` is commonly paired with `ENTRYPOINT`: `ENTRYPOINT` sets the fixed executable, and `CMD` supplies default arguments to it.

---

## ENTRYPOINT

The `ENTRYPOINT` instruction sets the fixed executable that always runs when the container starts, making the image behave like an executable itself. Arguments passed to `docker run` (or supplied via `CMD`) are appended to the `ENTRYPOINT` command rather than replacing it.

### Syntax

```dockerfile
ENTRYPOINT ["executable", "param1", "param2"]   # exec form (preferred)
ENTRYPOINT command param1 param2                 # shell form
```

### Example

```dockerfile
FROM alpine:3.19

ENTRYPOINT ["echo", "Hello"]
CMD ["World"]
```

### Usage

```bash
docker build -t entrypoint-example .

# Runs: echo Hello World
docker run entrypoint-example

# Runs: echo Hello Everyone  (overrides only the CMD portion)
docker run entrypoint-example Everyone
```

### Notes

- Unlike `CMD`, `ENTRYPOINT` is **not** replaced by arguments on the `docker run` command line — those arguments are appended instead. To override `ENTRYPOINT` itself, pass `--entrypoint` explicitly:

  ```bash
  docker run --entrypoint sh entrypoint-example
  ```

- A common pattern: use `ENTRYPOINT` for the fixed command/binary, and `CMD` for default arguments that the user can still override.
- Prefer the exec form for the same signal-handling reasons as `CMD`.

---

## COPY

The `COPY` instruction copies files or folders from the build context into the image filesystem. It's the simpler, more predictable counterpart to `ADD` — no archive extraction, no remote URL support.

### Syntax

```dockerfile
COPY <src> <dest>
```

### Example

```dockerfile
FROM alpine:3.19

WORKDIR /usr/src/app
COPY app.js .
COPY config/ ./config/
```

### Usage

```bash
docker build -t copy-example .
docker run --rm copy-example ls /usr/src/app
```

### Notes

- `COPY` is the recommended default for bringing local files into an image — use `ADD` only when you specifically need its extra behavior (archive auto-extract or remote URLs).
- Supports a `--chown` flag to set file ownership during the copy, useful when running as a non-root user:

  ```dockerfile
  COPY --chown=node:node . .
  ```

- Supports multi-stage build copying, pulling files from an earlier build stage:

  ```dockerfile
  COPY --from=builder /app/dist ./dist
  ```

---

## ADD

The `ADD` instruction copies files, folders, or remote URLs from the build context into the image filesystem. It behaves like `COPY`, but with two extra abilities: it can fetch files from a URL, and it automatically extracts local `.tar`, `.tar.gz`, `.tar.bz2`, etc. archives into the destination.

### Syntax

```dockerfile
ADD <src> <dest>
```

### Example

```dockerfile
FROM alpine:3.19

# Copy a local file
ADD app.tar.gz /usr/src/app/

# Add a file from a remote URL
ADD https://example.com/file.txt /usr/src/app/file.txt
```

### Usage

```bash
docker build -t add-example .
docker run --rm add-example ls /usr/src/app
```

### Notes

- Prefer `COPY` over `ADD` for plain local file copies — it's more predictable and explicit, since `ADD`'s auto-extract and URL-fetch behavior can cause unexpected results (e.g. accidentally extracting an archive you meant to copy as-is).
- Use `ADD` only when you specifically need its archive-extraction or remote-URL behavior.

---

## ENV

The `ENV` instruction sets environment variables that persist in the built image and are available to any process running inside the resulting container.

### Syntax

```dockerfile
ENV <key>=<value>
ENV <key>=<value> <key2>=<value2>
```

### Example

```dockerfile
FROM alpine:3.19

ENV APP_ENV=production
ENV PORT=8080

CMD ["sh", "-c", "echo Running in $APP_ENV on port $PORT"]
```

### Usage

```bash
docker build -t env-example .
docker run env-example

# Override at runtime
docker run -e APP_ENV=development env-example
```

### Notes

- `ENV` values are baked into the image and visible via `docker inspect`, unlike `ARG`, which is build-time only.
- Values set with `ENV` can be overridden at `docker run` time with `-e KEY=value`, without rebuilding the image.
- Useful for setting defaults that application code reads via `process.env` (Node.js), `os.environ` (Python), etc.

---

## EXPOSE

The `EXPOSE` instruction documents which port(s) a container listens on. It's purely informational — it does **not** actually publish the port to the host. Publishing still requires `-p`/`-P` on `docker run` or `ports:` in Compose.

### Syntax

```dockerfile
EXPOSE <port>[/<protocol>]
```

### Example

```dockerfile
FROM nginx:alpine

EXPOSE 80
EXPOSE 443/tcp
```

### Usage

```bash
docker build -t expose-example .

# EXPOSE alone does nothing visible from outside the container
docker run -d expose-example

# Port must still be explicitly published to be reachable from the host
docker run -d -p 8080:80 expose-example
```

### Notes

- `EXPOSE` serves as documentation for anyone reading the Dockerfile, and is also used by `docker run -P` (capital P), which auto-publishes every `EXPOSE`d port to a random host port.
- Default protocol is `tcp` if not specified; use `EXPOSE <port>/udp` for UDP services.

---

## LABEL

The `LABEL` instruction adds metadata to an image as key-value pairs — useful for documenting ownership, version, source repo, or any other descriptive info, queryable later via `docker inspect`.

### Syntax

```dockerfile
LABEL <key>="<value>"
LABEL <key1>="<value1>" <key2>="<value2>"
```

### Example

```dockerfile
FROM alpine:3.19

LABEL maintainer="prasanna@example.com"
LABEL version="1.0.0"
LABEL description="Example image demonstrating the LABEL instruction"
```

### Usage

```bash
docker build -t label-example .
docker inspect --format='{{json .Config.Labels}}' label-example
```

### Notes

- Labels are purely metadata — they have no effect on how the container runs.
- Multiple `LABEL` instructions are all retained (unlike `CMD`/`ENTRYPOINT`, where only the last one counts) — though combining related labels into one instruction reduces the number of image layers.
- Common conventions include `org.opencontainers.image.*` labels for standardized metadata (source, revision, licenses, etc.).

---

## ONBUILD

The `ONBUILD` instruction registers a trigger instruction that doesn't run during this image's own build — instead, it runs later, when **another** image uses this one as its `FROM` base.

### Syntax

```dockerfile
ONBUILD <INSTRUCTION>
```

### Example

**Base image (`Dockerfile.base`):**

```dockerfile
FROM alpine:3.19

ONBUILD COPY . /usr/src/app
ONBUILD RUN echo "Running ONBUILD step from base image"
```

**Child image, built from the base above:**

```dockerfile
FROM onbuild-example-base
# COPY and RUN from ONBUILD fire automatically here, even though
# this Dockerfile never mentions them
```

### Usage

```bash
# Build the base image first
docker build -f Dockerfile.base -t onbuild-example-base .

# Build a child image — the ONBUILD triggers fire during THIS build
docker build -t onbuild-example-child .
```

### Notes

- `ONBUILD` instructions are invisible to anyone just reading the child Dockerfile — they execute silently, inherited from the parent. This can make builds harder to reason about, which is why `ONBUILD` is less commonly used in modern Dockerfiles compared to alternatives like multi-stage builds.
- Common historical use case: base "framework" images (e.g. old versions of language runtime images) that automatically copied and built application code placed in any child image.

---

## USER

The `USER` instruction sets the user (and optionally group) that subsequent instructions, and the final container process, run as. By default, containers run as `root`, which is a security risk for production images.

### Syntax

```dockerfile
USER <user>[:<group>]
```

### Example

```dockerfile
FROM alpine:3.19

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

USER appuser

CMD ["whoami"]
```

### Usage

```bash
docker build -t user-example .
docker run --rm user-example
# Output: appuser
```

### Notes

- Running containers as a non-root user is a security best practice — if an attacker compromises the application, they're limited to that user's permissions rather than root inside the container.
- `USER` affects all instructions after it in the Dockerfile (e.g. a later `RUN` will execute as that user too), as well as the default user for the running container.
- Some base images already include a non-root user you can reuse (e.g. `node` images include a `node` user) instead of creating your own.

---

## WORKDIR

The `WORKDIR` instruction sets the working directory for any subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, or `ADD` instructions in the Dockerfile, and for the container's process when it starts.

### Syntax

```dockerfile
WORKDIR <path>
```

### Example

```dockerfile
FROM alpine:3.19

WORKDIR /usr/src/app

COPY app.sh .
RUN chmod +x app.sh

CMD ["./app.sh"]
```

### Usage

```bash
docker build -t workdir-example .
docker run --rm workdir-example
```

### Notes

- If the directory specified doesn't exist, `WORKDIR` creates it automatically.
- Using `WORKDIR` is preferred over `RUN cd <path>` — each `RUN` instruction executes in its own shell, so a `cd` doesn't persist to the next instruction, while `WORKDIR` does.
- Can be used multiple times in a Dockerfile; relative paths are resolved against the previous `WORKDIR`.
