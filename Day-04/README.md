# 🐳 Day 4 — Custom Dockerfile, Build Context & Docker Commands

<div align="center">

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![CKA](https://img.shields.io/badge/CKA-Prep-F59E0B?style=for-the-badge&logo=kubernetes&logoColor=white)
![Day](https://img.shields.io/badge/Day-04-10B981?style=for-the-badge)

</div>

---

## 📌 Topics Covered

- [Custom Dockerfile Names](#-custom-dockerfile-names)
- [Understanding Build Context](#-understanding-build-context)
- [CMD vs ENTRYPOINT vs RUN — Full Comparison](#-cmd-vs-entrypoint-vs-run--full-comparison)
- [Container Management Commands](#-container-management-commands)
- [Image Management Commands](#-image-management-commands)
- [Summary](#-summary)

---

## 📄 Custom Dockerfile Names

By default Docker looks for a file named `Dockerfile`. But you can use any name with the `-f` flag — useful when you need multiple Dockerfiles in the same project.

### Why use custom names?

| Scenario | Example |
|---|---|
| **Environment-specific builds** | `Dockerfile.dev` for development, `Dockerfile.prod` for production |
| **Complex projects** | Different services needing distinct Dockerfiles |
| **Team clarity** | Clearly named files communicate their purpose immediately |

### Command syntax

```bash
docker build -t <image-name> -f <path-to-dockerfile> <build-context>
```

### Flag breakdown

| Flag | Purpose |
|---|---|
| `-t <image-name>` | Assigns a name and optional tag to the built image |
| `-f <path-to-dockerfile>` | Points to your custom-named Dockerfile |
| `<build-context>` | The directory Docker uses to find files during the build |

### Examples

```bash
# Build using a custom Dockerfile name in the current directory
docker build -t my-app -f Dockerfile.dev .

# Build using a custom Dockerfile in a different location
docker build -t my-app -f /path/to/Dockerfile.prod /path/to/build-context

# Build development image
docker build -t my-app:dev -f Dockerfile.dev .

# Build production image
docker build -t my-app:prod -f Dockerfile.prod .
```

---

## 📦 Understanding Build Context

The **build context** is the directory Docker sends to the Docker daemon when you run `docker build`. Every file inside this directory becomes accessible during the build.

```
docker build -t my-app -f Dockerfile.dev .
                                          ↑
                                   build context
                                (current directory)
```

### How it works

```
  Your Machine                          Docker Daemon
  ─────────────                         ─────────────
  /my-project/
  ├── Dockerfile
  ├── app.py           ── sent ──▶      Build context received
  ├── requirements.txt                  COPY, ADD can access
  └── config/                           these files
```

### Key rules

| Rule | Detail |
|---|---|
| **Accessibility** | Only files inside the build context can be accessed by `COPY` and `ADD` |
| **Files outside context** | Cannot be referenced — Docker will throw an error |
| **Optimisation** | Use `.dockerignore` to exclude files you don't need |

### `.dockerignore` example

```
# Exclude these from the build context
node_modules
*.log
.git
.env
__pycache__
```

> 💡 A smaller build context = faster transfer to the Docker daemon = faster builds. Always use `.dockerignore` in production projects.

---

## 🔀 CMD vs ENTRYPOINT vs RUN — Full Comparison

All three look like they "run commands" but they serve completely different purposes.

### When does each execute?

```
  Image Build Time                      Container Start Time
  ────────────────                      ────────────────────
  RUN executes here          →          CMD executes here
  (result baked into image)             (default, overwritable)
                             →          ENTRYPOINT executes here
                                        (fixed, append-only)
```

### Full comparison table

| Aspect | `RUN` | `CMD` | `ENTRYPOINT` |
|---|---|---|---|
| **When it runs** | At build time | At container start | At container start |
| **Purpose** | Modify the image filesystem | Default runtime command | Fixed runtime command |
| **Creates a layer** | ✅ Yes | ❌ No — metadata only | ❌ No — metadata only |
| **Override at runtime** | Cannot — baked into image | ✅ Fully overwritten | 🔒 Cannot — args append to it |
| **Shell form** | ✅ Supported | ✅ Supported | ❌ Not recommended |
| **Exec form** | ✅ Supported | ✅ Supported | ✅ Supported |
| **Multiple allowed** | ✅ Yes — each creates a layer | ❌ Only last one counts | ❌ Only last one counts |
| **Use case** | Install packages, set up environment | Default command users can swap | Fixed entrypoint that always runs |

---

### `RUN` — build time only

```dockerfile
RUN apt-get update && apt-get install -y python3
```

Executes during `docker build`. Result is permanently baked into the image as a new layer. Cannot be changed at runtime.

---

### `CMD` — default runtime command, fully overwritable

```dockerfile
CMD ["python", "app.py"]
```

```bash
# Uses CMD — runs python app.py
docker run my-image

# Overwrites CMD entirely — runs python other.py instead
docker run my-image python other.py
```

> Only the **last** `CMD` instruction in a Dockerfile is used. All previous ones are ignored.

---

### `ENTRYPOINT` — fixed runtime command, args append to it

```dockerfile
ENTRYPOINT ["ping", "-c", "4"]
```

```bash
# Appends google.com — runs: ping -c 4 google.com
docker run my-image google.com

# Appends yahoo.com — runs: ping -c 4 yahoo.com
docker run my-image yahoo.com

# Only way to fully override ENTRYPOINT
docker run --entrypoint /bin/bash my-image
```

> `ENTRYPOINT` supports **exec form only** as best practice. Shell form is not recommended because it wraps the command in `/bin/sh -c`, breaking signal handling.

---

### `ENTRYPOINT` + `CMD` together — the production pattern

```dockerfile
ENTRYPOINT ["ping", "-c", "4"]
CMD ["google.com"]
```

```bash
# Uses both — runs: ping -c 4 google.com
docker run my-image

# Overwrites CMD only — runs: ping -c 4 yahoo.com
docker run my-image yahoo.com
```

```
ENTRYPOINT ["ping", "-c", "4"]   +   CMD ["google.com"]
      ↓                                      ↓
  always runs                         default argument
  cannot be replaced                  freely overwritable
```

> 💡 This is the recommended pattern for production Dockerfiles. ENTRYPOINT defines the tool. CMD defines the default input.

---

### Shell form vs Exec form — quick recap

| | Shell form | Exec form |
|---|---|---|
| **Syntax** | `CMD python app.py` | `CMD ["python", "app.py"]` |
| **Runs via** | `/bin/sh -c` | Directly — no shell |
| **PID 1** | Shell | Your process |
| **Signal handling** | ❌ Broken — shell intercepts signals | ✅ Clean — process receives signals directly |
| **Recommended** | ❌ No | ✅ Yes |

---

## 🛠️ Container Management Commands

### Inspect & monitor

```bash
# List all containers including stopped ones
docker ps -a

# Inspect full metadata of a container or image
docker inspect <container-id>
docker inspect <image-id>

# List running processes inside a container
docker top <container-id>
```

### Start, stop & restart

```bash
# Stop a specific container
docker stop <container-id>

# Start a stopped container
docker start <container-id>

# Restart a container
docker restart <container-id>

# Stop all running containers
docker stop $(docker ps -q)
```

### Remove containers

```bash
# Remove a specific container (must be stopped first)
docker rm <container-id>

# Remove all stopped containers at once
docker container prune
```

---

## 🖼️ Image Management Commands

### Dangling images

A **dangling image** is an image that is no longer tagged or associated with any container. They appear as `<none>` in `docker images` and waste disk space.

```bash
# See dangling images
docker images -f dangling=true
```

### Remove images

```bash
# Remove a specific image
docker rmi <image-id>

# Remove all unused images including dangling ones
docker image prune -a
```

> ⚠️ `docker image prune -a` removes **all** images not currently used by a running container — not just dangling ones. Use with care.

---

### Full command reference

| Command | What it does |
|---|---|
| `docker ps -a` | List all containers including stopped |
| `docker inspect <id>` | Full metadata of a container or image |
| `docker top <id>` | Processes running inside a container |
| `docker stop <id>` | Stop a running container |
| `docker start <id>` | Start a stopped container |
| `docker restart <id>` | Restart a container |
| `docker stop $(docker ps -q)` | Stop all running containers |
| `docker rm <id>` | Remove a stopped container |
| `docker container prune` | Remove all stopped containers |
| `docker rmi <id>` | Remove a specific image |
| `docker image prune -a` | Remove all unused images |

---

## ✅ Summary

| Concept | Key takeaway |
|---|---|
| **Custom Dockerfile** | Use `-f` flag to specify any filename — useful for dev/prod separation |
| **Build context** | The directory sent to Docker daemon — files outside it are inaccessible |
| **`.dockerignore`** | Exclude unnecessary files from build context for faster builds |
| **`RUN`** | Executes at build time — creates a new layer — baked into the image |
| **`CMD`** | Executes at container start — metadata only — fully overwritten at runtime |
| **`ENTRYPOINT`** | Executes at container start — metadata only — runtime args append to it |
| **Exec form** | Always preferred — process becomes PID 1, handles signals correctly |
| **`ENTRYPOINT` + `CMD`** | Best production pattern — fixed command + swappable default argument |
| **Dangling images** | Untagged, unused images — clean with `docker image prune -a` |
| **`docker inspect`** | Most detailed view of any container or image metadata |

---

<div align="center">

**← [Day 3](../Day-03/README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Back to Index](../README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Day 5 →](../Day-05/README.md)**

---

</div>