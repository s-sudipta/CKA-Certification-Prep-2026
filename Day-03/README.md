# 🐳 Day 3 — Docker Flags, Deep Dive into Dockerfile & Exposing Containers

<div align="center">

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![CKA](https://img.shields.io/badge/CKA-Prep-F59E0B?style=for-the-badge&logo=kubernetes&logoColor=white)
![Day](https://img.shields.io/badge/Day-03-10B981?style=for-the-badge)

</div>

---

## 📌 Topics Covered

- [Important Docker Flags](#-important-docker-flags)
- [Dockerfile Instructions — Deep Dive](#-dockerfile-instructions--deep-dive)
- [Example Dockerfile — Flask App](#-example-dockerfile--flask-app)
- [Shell Form vs Exec Form](#-shell-form-vs-exec-form)
- [Container Inspection & Debugging Commands](#-container-inspection--debugging-commands)
- [Cleanup Commands](#-cleanup-commands)
- [Summary](#-summary)

---

## 🚩 Important Docker Flags

### The command

```bash
docker run -d -p 8080:80 --name my-nginx-cont nginx
```

Breaking it down:

| Flag | Value | What it does |
|---|---|---|
| `-d` | — | **Detached mode** — runs the container in the background, terminal stays free |
| `-p` | `8080:80` | **Port mapping** — maps port `80` inside the container to port `8080` on your host |
| `--name` | `my-nginx-cont` | **Container name** — assigns a human-readable name instead of a random one |
| `nginx` | — | The image to run |

Once running, open `http://localhost:8080` in your browser — you'll see the Nginx default page.

### Port mapping explained

```
  Your Machine                    Container
  ────────────                    ─────────
  localhost:8080   ──────────▶    port 80 (Nginx)

  Host Port : Container Port
     8080   :      80
```

> 💡 The format is always `-p <host-port>:<container-port>`. Traffic hitting your machine on port 8080 gets forwarded into the container on port 80.

---

## 📄 Dockerfile Instructions — Deep Dive

### `FROM`

Defines the base image — every Dockerfile must start with this.

```dockerfile
FROM ubuntu:latest
FROM python:3.9-slim
```

---

### `RUN`

Executes a command **at build time**. Creates a new image layer with the result.

```dockerfile
RUN apt-get update && apt-get install -y nginx
```

> 💡 Combine multiple commands into one `RUN` to reduce layers and image size.

---

### `COPY`

Copies files or directories from your host machine into the image.
Simple, explicit — does nothing extra.

```dockerfile
COPY app.py /app/app.py
```

---

### `ADD`

Like `COPY` but with two extra abilities:
- Automatically **extracts `.tar.gz`** archives
- Can fetch files from a **URL**

```dockerfile
ADD app.tar.gz /app
```

> 💡 Prefer `COPY` for simple file copying. Use `ADD` only when you specifically need tar extraction or URL support.

---

### `WORKDIR`

Sets the working directory inside the container for all subsequent instructions.
Creates the directory if it does not exist.

```dockerfile
WORKDIR /app
```

---

### `EXPOSE`

Documents which port the containerised application listens on.

```dockerfile
EXPOSE 5000
```

> ⚠️ `EXPOSE` does **not** actually open or publish the port. It is documentation only. To actually publish the port, use the `-p` flag when running the container:
> ```bash
> docker run -p 5000:5000 my-app
> ```

---

### `CMD`

Specifies the **default command** to run when the container starts.
Can be overridden at runtime by passing a command after the image name.

```dockerfile
CMD ["python", "app.py"]
```

```bash
# Override CMD at runtime
docker run my-app python other_script.py
```

---

### `ENTRYPOINT`

Defines the command that **always runs** when the container starts.
Cannot be overridden easily — arguments passed via `docker run` are **appended** to it.

```dockerfile
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

```bash
# To fully override ENTRYPOINT, use --entrypoint flag
docker run --entrypoint /bin/bash my-nginx-cont
```

### `CMD` vs `ENTRYPOINT` — when to use which

| | `CMD` | `ENTRYPOINT` |
|---|---|---|
| **Purpose** | Default command — easily replaceable | Fixed command — always executes |
| **Override** | `docker run <image> <new-command>` replaces it | `docker run <image> <args>` appends to it |
| **Use case** | Flexible containers where the command may change | Containers with a fixed, specific purpose |
| **Force override** | — | `--entrypoint` flag required |

---

## 🐍 Example Dockerfile — Flask App

```dockerfile
# Use a lightweight Python image as the base
FROM python:3.9-slim

# Set the working directory inside the container
WORKDIR /app

# Copy the application file from the host to the container
ADD app.py /app/app.py

# Install the necessary Python library
RUN pip install flask

# Expose the port where the app will listen
EXPOSE 5000

# Specify the default command to run the application
CMD ["python", "app.py"]
```

### What each instruction does here

```
FROM python:3.9-slim     ← base layer (pulled from registry)
WORKDIR /app             ← ✅ new layer — creates /app directory
ADD app.py /app/app.py   ← ✅ new layer — copies file into image
RUN pip install flask    ← ✅ new layer — installs flask, modifies filesystem
EXPOSE 5000              ← ✅ new layer — records port metadata
CMD ["python", "app.py"] ← ❌ no layer — metadata only, runtime instruction
```

---

## 🔀 Shell Form vs Exec Form

Both `CMD` and `ENTRYPOINT` support two syntax forms. Understanding the difference matters for signal handling and process behaviour.

### Shell Form

```dockerfile
CMD echo "Hello World"
CMD python app.py
```

Runs the command through `/bin/sh -c`. The shell becomes **PID 1** — not your actual process.

### Exec Form

```dockerfile
CMD ["echo", "Hello World"]
CMD ["python", "app.py"]
```

Runs the command **directly** without a shell. Your actual process becomes **PID 1**.

### Comparison

| Feature | Shell Form | Exec Form |
|---|---|---|
| **Syntax** | `CMD command` | `CMD ["executable", "arg1", "arg2"]` |
| **Execution** | Runs via `/bin/sh -c` | Runs directly — no shell |
| **PID 1** | Shell becomes PID 1 | Your process becomes PID 1 |
| **Signal handling** | Shell intercepts signals — process may not receive `SIGTERM` | Process receives signals directly — clean shutdown |
| **Environment variables** | ✅ Supports `$VAR` expansion | ❌ No shell expansion |
| **Complex commands** | ✅ Supports `&&`, pipes, chaining | ❌ Not suitable for complex shell logic |
| **Best for** | Commands needing shell features | Production containers — recommended default |

### Why PID 1 matters

```
Shell Form:                          Exec Form:
─────────────────────────────        ─────────────────────────────
PID 1: /bin/sh                       PID 1: python app.py
  └── PID 2: python app.py

docker stop sends SIGTERM            docker stop sends SIGTERM
  └── Shell receives it                └── python receives it directly
  └── May NOT forward to python       └── App shuts down cleanly ✅
  └── Container force-killed ❌
```

> 💡 **Always use Exec Form in production.** It ensures your application receives shutdown signals directly and can exit gracefully.

---

## 🔍 Container Inspection & Debugging Commands

### View container logs

```bash
docker logs <container-name>
docker logs my-nginx-cont
```

Prints everything the container has written to stdout and stderr. Essential for debugging.

---

### Execute a command inside a running container

```bash
# Run a single command inside the container
docker exec <container-name> ls
docker exec <container-name> ls -a

# Open an interactive bash shell inside the container
docker exec -it <container-name> bash
```

| Flag | Meaning |
|---|---|
| `-i` | Interactive — keeps stdin open |
| `-t` | Allocates a pseudo-TTY (terminal) |
| `bash` | The shell to open inside the container |

```
  Your terminal
       │
       │  docker exec -it my-nginx-cont bash
       ▼
  root@a3f1c2d:/app#   ← you are now inside the container
```

> 💡 If `bash` is not available in the container (e.g. Alpine-based images), use `sh` instead:
> ```bash
> docker exec -it <container-name> sh
> ```

---

## 🧹 Cleanup Commands

### Stop, remove containers and images — step by step

```bash
# Step 1: Stop all running containers
docker stop $(docker ps -aq)

# Step 2: Remove all containers (running + stopped)
docker rm $(docker ps -aq)

# Step 3: Remove all images
docker rmi $(docker images -q)
```

### How the subcommands work

| Subcommand | What it returns |
|---|---|
| `docker ps -aq` | IDs of **all** containers (running + stopped) — `-a` all, `-q` IDs only |
| `docker images -q` | IDs of **all** local images — `-q` IDs only |

```
docker stop $(docker ps -aq)
               │
               └─▶ docker ps -aq returns: a3f1c2 b4d2e1 c5f3g2
                   docker stop a3f1c2 b4d2e1 c5f3g2
```

---

### Nuclear option — remove everything at once

```bash
docker system prune -a
```

This removes in one command:
- All stopped containers
- All unused images (not just dangling)
- All unused networks
- All build cache

```
WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all images without at least one container associated to them
  - all build cache

Are you sure you want to continue? [y/N]
```

> ⚠️ `docker system prune -a` is destructive and irreversible. Use with caution — especially on shared or production machines. Always confirm what you're removing before proceeding.

### Cleanup command reference

| Command | What it removes |
|---|---|
| `docker stop $(docker ps -aq)` | Stops all running containers |
| `docker rm $(docker ps -aq)` | Removes all containers |
| `docker rmi $(docker images -q)` | Removes all local images |
| `docker system prune -a` | Removes everything unused in one shot |

---

## ✅ Summary

| Concept | Key takeaway |
|---|---|
| `-d` flag | Run container in background — detached mode |
| `-p host:container` | Publish a container port to the host |
| `--name` | Assign a readable name to a container |
| `EXPOSE` | Documentation only — does not publish the port |
| `ADD` vs `COPY` | Use `COPY` by default — use `ADD` only for tar extraction or URLs |
| `CMD` vs `ENTRYPOINT` | `CMD` is overridable — `ENTRYPOINT` always runs |
| Shell form vs Exec form | Always prefer Exec form in production for correct signal handling |
| `docker logs` | View container stdout / stderr output |
| `docker exec -it` | Open an interactive shell inside a running container |
| `docker system prune -a` | Remove all unused containers, images, networks and cache |

---

<div align="center">

**← [Day 2](../Day-02/README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Back to Index](../README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Day 4 →](../Day-04/README.md)**

---

</div>