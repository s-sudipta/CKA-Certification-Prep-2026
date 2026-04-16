# 🐳 Day 2 — Write Your First Dockerfile & Push to Docker Hub

<div align="center">

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![CKA](https://img.shields.io/badge/CKA-Prep-F59E0B?style=for-the-badge&logo=kubernetes&logoColor=white)
![Day](https://img.shields.io/badge/Day-02-10B981?style=for-the-badge)

</div>

---

## 📌 Topics Covered

- [Understanding docker pull](#-understanding-docker-pull)
- [Pulling from Other Registries](#-pulling-from-other-registries)
- [Write Your First Dockerfile](#-write-your-first-dockerfile)
- [Dockerfile Instructions — Layer vs No Layer](#-dockerfile-instructions--which-ones-create-a-layer)
- [Build, Tag, Login & Push to Docker Hub](#-build-tag-login--push-to-docker-hub)
- [Run a Container & Verify](#-run-a-container--verify)
- [Summary](#-summary)

---

## 🔍 Understanding `docker pull`

When you run:

```bash
docker pull ubuntu:latest
```

Docker internally expands this to the **fully qualified image name**:

```bash
docker pull docker.io/library/ubuntu:latest
```

| Part | Meaning |
|---|---|
| `docker.io` | Default registry — Docker Hub |
| `library` | Official images namespace on Docker Hub |
| `ubuntu` | Image name |
| `latest` | Tag (version) |

> 💡 If no registry is specified, Docker always assumes `docker.io`. If no tag is specified, Docker assumes `latest`.

---

## 🌐 Pulling from Other Registries

The `docker pull` command works the same way across all registries — the only difference is the fully qualified path.

### AWS Elastic Container Registry (ECR)

```bash
# Format
docker pull <account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>

# Example
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
```

### Google Container Registry (GCR)

```bash
# Format
docker pull gcr.io/<project-id>/<image-name>:<tag>

# Example
docker pull gcr.io/my-gcp-project/my-app:v1.0
```

### Key Points

- The pull mechanism is identical across all registries
- For **private registries** (ECR, GCR), you must authenticate before pulling
- Docker Hub is the only registry that works without specifying the full URI

---

## 📄 Write Your First Dockerfile

Create a file named `Dockerfile` (no extension) in your project directory:

```dockerfile
FROM ubuntu:latest
RUN echo "Hello, Docker!"
```

### What each instruction does:

| Instruction | Purpose |
|---|---|
| `FROM ubuntu:latest` | Sets Ubuntu as the base image |
| `RUN echo "Hello, Docker!"` | Executes the echo command during image build |

---

## 🏗️ Dockerfile Instructions — Which Ones Create a Layer?

This is one of the most important concepts to understand about how Docker images work internally.

### How Docker layers work

Every Docker image is made of **stacked layers**. Some Dockerfile instructions create a **new layer** (a filesystem snapshot), while others only add **metadata** to the image without creating a new layer.

```
┌──────────────────────────────────────────────────────┐
│  Instructions that CREATE a new layer                │
│  → Each adds to the image's filesystem               │
│  → Increases image size                              │
│  → Gets its own cache entry                          │
│  → Shows "created just now" in docker inspect        │
├──────────────────────────────────────────────────────┤
│  Instructions that DO NOT create a layer             │
│  → Only add metadata to the image                    │
│  → Do not increase image size                        │
│  → No filesystem change                              │
└──────────────────────────────────────────────────────┘
```

---

### ✅ Instructions that CREATE a new layer

| Instruction | What it does |
|---|---|
| `RUN` | Executes a command and saves the result as a new layer |
| `COPY` | Copies files from host into the image — new layer |
| `ADD` | Like COPY but also supports URLs and tar extraction — new layer |
| `WORKDIR` | Sets working directory — creates it if it doesn't exist, adds a layer |
| `VOLUME` | Creates a mount point — adds a layer |
| `EXPOSE` | Documents a port — adds a layer |

---

### ❌ Instructions that DO NOT create a new layer

| Instruction | What it does |
|---|---|
| `ARG` | Defines a build-time variable — metadata only |
| `ENV` | Sets environment variables — metadata only |
| `LABEL` | Adds key-value metadata to the image |
| `ONBUILD` | Trigger instruction for child images — metadata only |
| `USER` | Sets the user for subsequent instructions — metadata only |
| `CMD` | Default runtime command — metadata only |
| `ENTRYPOINT` | Configures the container's main process — metadata only |

---

### 🔬 The `CMD` vs `RUN` Layer Experiment

This is the best way to understand the difference concretely.

#### Case 1 — Using `CMD` (no new layer created)

```dockerfile
FROM ubuntu:latest
CMD echo "Hello, Docker!"
```

When you build this and inspect the image:

```bash
docker build -t test-cmd .
docker inspect test-cmd
```

```
"Created": "2024-01-15T10:00:00Z"   ← Same date as ubuntu:latest base image
```

**Why?** `CMD` only adds metadata — it tells Docker what to run when the container starts but does **not** create any new filesystem layer. So the image timestamp stays the same as the base `ubuntu:latest` image.

---

#### Case 2 — Using `RUN` (new layer created)

```dockerfile
FROM ubuntu:latest
RUN echo "Hello, Docker!"
```

When you build this and inspect the image:

```bash
docker build -t test-run .
docker inspect test-run
```

```
"Created": "2024-04-16T08:35:22Z"   ← TODAY's date — a new layer was created NOW
```

**Why?** `RUN` executes the command during the build process and saves the resulting filesystem state as a **new layer**. This means Docker had to do real work at build time, so the image gets a new timestamp reflecting when it was built.

---

#### Visual comparison

```
Case 1: FROM ubuntu:latest + CMD echo "..."
─────────────────────────────────────────────────────────
  ubuntu:latest layer   (created: Jan 15, 2024)
  CMD metadata          (no new layer — same date as above)

  docker inspect → Created: Jan 15, 2024  ← base image date


Case 2: FROM ubuntu:latest + RUN echo "..."
─────────────────────────────────────────────────────────
  ubuntu:latest layer   (created: Jan 15, 2024)
  ─────────────────────────────────────────────
  RUN layer             (created: Apr 16, 2024)  ← NEW layer added TODAY

  docker inspect → Created: Apr 16, 2024  ← build date
```

---

#### Full layer example — Node.js app

```dockerfile
FROM node:18-alpine          # base layer (pulled from registry)

WORKDIR /app                 # ✅ creates a layer — makes the /app directory

COPY package*.json ./        # ✅ creates a layer — copies files into image filesystem
RUN npm install              # ✅ creates a layer — installs packages, modifies filesystem

COPY . .                     # ✅ creates a layer — copies source code

EXPOSE 3000                  # ✅ creates a layer — records port metadata in filesystem

ENV NODE_ENV=production      # ❌ no layer — metadata only
LABEL version="1.0"          # ❌ no layer — metadata only
ARG BUILD_DATE               # ❌ no layer — build-time variable, not persisted

CMD ["node", "index.js"]     # ❌ no layer — runtime instruction, metadata only
```

Resulting image layers:

```
┌─────────────────────────────────┐
│  Layer 6  │  COPY . .           │
├─────────────────────────────────┤
│  Layer 5  │  RUN npm install    │
├─────────────────────────────────┤
│  Layer 4  │  COPY package*.json │
├─────────────────────────────────┤
│  Layer 3  │  EXPOSE 3000        │
├─────────────────────────────────┤
│  Layer 2  │  WORKDIR /app       │
├─────────────────────────────────┤
│  Layer 1  │  node:18-alpine     │  ← base
└─────────────────────────────────┘

  ENV, LABEL, ARG, CMD → not shown as layers (metadata only)
```

> 💡 **Why does this matter?** Fewer layers = smaller image. Understanding which instructions create layers helps you write optimised Dockerfiles. For example, combining multiple `RUN` commands into one reduces the layer count:
> ```dockerfile
> # Bad — 3 separate layers
> RUN apt-get update
> RUN apt-get install -y curl
> RUN apt-get clean
>
> # Good — 1 layer
> RUN apt-get update && apt-get install -y curl && apt-get clean
> ```

---

## 🚀 Build, Tag, Login & Push to Docker Hub

### Step 1 — Build the image

```bash
docker build -t my-first-image .
```

| Part | Meaning |
|---|---|
| `docker build` | Build command |
| `-t my-first-image` | Name (tag) for the image |
| `.` | Build context — current directory where Dockerfile lives |

---

### Step 2 — Tag the image for Docker Hub

```bash
docker tag my-first-image <your-username>/my-first-image:v1.0
```

| Part | Meaning |
|---|---|
| `my-first-image` | The locally built image name |
| `<your-username>` | Your Docker Hub username |
| `my-first-image` | Repository name on Docker Hub |
| `v1.0` | Version tag |

Docker Hub automatically prefixes `docker.io` internally:

```bash
# These two are equivalent
docker tag my-first-image sudiptasaha/my-first-image:v1.0
docker tag my-first-image docker.io/sudiptasaha/my-first-image:v1.0
```

---

### Step 3 — Login to Docker Hub

Before pushing, you must authenticate:

```bash
docker login --username <your-username>
```

Docker will prompt for your password (or Personal Access Token if 2FA is enabled):

```
Password: ********
Login Succeeded
```

Or pass credentials in one line (not recommended for security — avoid in scripts):

```bash
docker login --username <your-username> --password <your-password>
```

> 💡 **Best practice:** Use a **Personal Access Token (PAT)** instead of your password. Generate one at `hub.docker.com → Account Settings → Security → Access Tokens`. This is safer and can be scoped and revoked independently.

To logout when done:

```bash
docker logout
```

---

### Step 4 — Push the image to Docker Hub

```bash
docker push <your-username>/my-first-image:v1.0
```

Docker expands this to:

```bash
docker push docker.io/<your-username>/my-first-image:v1.0
```

After pushing, your image is publicly available at:

```
https://hub.docker.com/r/<your-username>/my-first-image
```

---

### Full flow — end to end

```
  📄 Dockerfile (in current directory)
       │
       │  docker build -t my-first-image .
       ▼
  🖼️  Local Image: my-first-image
       │
       │  docker tag my-first-image <username>/my-first-image:v1.0
       ▼
  🏷️  Tagged Image: <username>/my-first-image:v1.0
       │
       │  docker login --username <username>
       │  docker push <username>/my-first-image:v1.0
       ▼
  📡  Docker Hub: hub.docker.com/r/<username>/my-first-image
       │
       │  docker pull <username>/my-first-image:v1.0   (any machine)
       │  docker run <username>/my-first-image:v1.0
       ▼
  📦  Running Container
```

---

## ▶️ Run a Container & Verify

### Run the container

```bash
docker run my-first-image
```

### List all containers (including stopped)

```bash
docker ps -a
```

Output:

```
CONTAINER ID   IMAGE            COMMAND                  CREATED        STATUS                    
a3f1c2d4e5b6   my-first-image   "/bin/sh -c 'echo H…"   5 seconds ago  Exited (0) 4 seconds ago
```

> The container exits immediately after running `echo "Hello, Docker!"` because there is no long-running process — this is expected behaviour.

---

## ✅ Summary

| Concept | Key takeaway |
|---|---|
| `docker pull` | Defaults to `docker.io/library/<image>:latest` if no registry or tag is specified |
| Other registries | Same pull mechanism — just use the fully qualified URI (ECR, GCR, ACR…) |
| `CMD` vs `RUN` | `CMD` = metadata only, no layer, no new timestamp. `RUN` = new layer, new timestamp |
| Layer-creating instructions | `RUN`, `COPY`, `ADD`, `WORKDIR`, `VOLUME`, `EXPOSE` |
| Non-layer instructions | `ARG`, `ENV`, `LABEL`, `ONBUILD`, `USER`, `CMD`, `ENTRYPOINT` |
| `docker build -t` | Builds image from Dockerfile in current directory |
| `docker tag` | Renames / versions an image for pushing to a registry |
| `docker login` | Authenticate with Docker Hub before pushing (use PAT, not password) |
| `docker push` | Uploads tagged image to Docker Hub |
| `docker run` | Creates and starts a container from an image |
| `docker ps -a` | Lists all containers — running and stopped |

---

<div align="center">

**← [Day 1](../Day-01/README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Back to Index](../README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Day 3 →](../Day-03/README.md)**

---

</div>