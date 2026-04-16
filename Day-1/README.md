# 🐳 Day 1 — Docker Fundamentals

<div align="center">

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![CKA](https://img.shields.io/badge/CKA-Prep-F59E0B?style=for-the-badge&logo=kubernetes&logoColor=white)
![Day](https://img.shields.io/badge/Day-01-10B981?style=for-the-badge)

</div>

---

## 📌 Topics Covered

- [Problems Before Docker](#-problems-before-docker)
- [What is Docker?](#-what-is-docker)
- [Shipping Container Analogy](#-the-shipping-container-analogy)
- [VMs vs Docker Containers](#-virtual-machines-vs-docker-containers)
- [Key Docker Concepts](#-key-docker-concepts)
- [Docker Architecture](#-docker-architecture)
- [Docker Workflow](#-docker-workflow)

---

## 🚧 Problems Before Docker

Before Docker, the software development and deployment lifecycle was **manual, fragile, and inconsistent** across every stage.

### 1. Development
- Dependencies and runtimes were installed manually on each developer's machine
- No two machines were guaranteed to be identical
- Teams maintained lengthy setup docs just to onboard one new member

### 2. Testing
- Test environments were manually configured to *resemble* production — never truly identical
- Minor config differences caused bugs that only appeared in production and couldn't be reproduced locally

### 3. Deployment
- Applications were packaged as `.tar.gz` or `.zip` archives
- Deployment relied on manual scripts to copy files, install dependencies, and configure services
- Every release was a potential source of human error

---

### ⚠️ Core Challenges of Traditional Deployment

| Challenge | Description | Example |
|---|---|---|
| **🔗 Dependency conflicts** | Different OS, library, and config versions across environments caused inconsistency | Dev uses Python 3.4, Prod runs Python 3.1 → app breaks in production |
| **💾 Inefficient resource usage** | Multiple VMs per application increased overhead and cost | Multiple apps on one VM competing for the same CPU, RAM, and ports |
| **⚙️ Complex deployments** | Manual configuration across Dev, Test, and Prod introduced human error | A missed environment variable causes a 3AM production failure |
| **🔓 Lack of isolation** | Apps on the same VM share dependencies, ports, configs — causing conflicts | Two apps needing different versions of the same library → unexpected crashes |

> **Solution → Docker Containers** ✅

---

## 🐳 What is Docker?

Docker is an open-source platform that allows developers to **build, ship, and run applications inside containers**. A container packages the application together with everything it needs — code, runtime, libraries, environment variables, and config files — into a single portable unit.

### Before Docker vs With Docker

| | Before Docker | With Docker |
|---|---|---|
| **Environment setup** | Manual — each machine configured separately by hand | Automated — the container carries everything it needs |
| **Consistency** | "Works on my machine" was a real problem | Same container runs identically on every machine |
| **Deployment** | Error-prone scripts, manual steps, human errors | `docker run` — one command, any environment |
| **Isolation** | Apps conflict on shared infrastructure | Every container runs fully isolated |

---

## 🚢 The Shipping Container Analogy

Before standardised shipping containers, moving goods required manual handling at every port — slow, expensive, and inconsistent. The introduction of **uniform, stackable containers** changed everything. The same container moved from ship → truck → train without repacking.

**Docker does the same thing for software.**

| Aspect | 🚢 Shipping Containers | 🐳 Docker Containers |
|---|---|---|
| **Standardisation** | Uniform 20ft/40ft sizes work across all transport modes | Bundles app + dependencies into identical units that work across all environments |
| **Portability** | Move between ship, truck, and train without repacking | Run across Dev, Test, and Prod without compatibility issues |
| **Isolation** | Goods stay separate — no contamination between shipments | Each container is isolated — one crash never affects another |
| **Scalability** | Scale logistics operations based on demand | Spin up hundreds of containers in seconds to meet traffic |
| **Efficiency** | Reduces manual handling, lowers cost | Automates deployment, minimises errors, ensures consistent performance |

---

## 🖥️ Virtual Machines vs Docker Containers

### Architecture Comparison

```
┌──────────────────────────────────┐    ┌──────────────────────────────────┐
│        VIRTUAL MACHINE           │    │       DOCKER CONTAINER           │
├─────────┬──────────┬─────────────┤    ├──────┬───────┬──────┬────────────┤
│  App 1  │  App 2   │   App 3     │    │ App1 │ App 2 │ App3 │   App 4    │
│ Lib/Bin │ Lib/Bin  │  Lib/Bin    │    │ Libs │ Libs  │ Libs │   Libs     │
│ GuestOS │ GuestOS  │  GuestOS    │    └──────┴───────┴──────┴────────────┘
├─────────┴──────────┴─────────────┤    ┌──────────────────────────────────┐
│           HYPERVISOR             │    │    CONTAINER ENGINE  (dockerd)   │
├──────────────────────────────────┤    ├──────────────────────────────────┤
│             HOST OS              │    │            HOST OS               │
├──────────────────────────────────┤    ├──────────────────────────────────┤
│          INFRASTRUCTURE          │    │         PHYSICAL SERVER          │
└──────────────────────────────────┘    └──────────────────────────────────┘
     Each VM carries its own OS 🐘           No Guest OS needed 🪶
```

### Feature Comparison

| Feature | 🖥️ Virtual Machines | 🐳 Docker Containers |
|---|---|---|
| **Architecture** | Full OS + Hypervisor per instance | Shares the host OS kernel |
| **Weight** | Heavyweight — Guest OS per VM | Lightweight — no Guest OS |
| **Startup time** | Minutes | Seconds / milliseconds |
| **Resource usage** | High — duplicates OS per VM | Low — shares host OS resources |
| **Isolation** | Full hardware-level isolation | Process-level isolation |
| **Deployment** | Slow and resource-intensive | Fast and efficient |
| **Best for** | Multiple OSes or legacy systems | Cloud-native apps and microservices |
| **Example** | Ubuntu on Windows via Hyper-V | Running a Node.js app in a container |

> 💡 Containers skip the Guest OS entirely — this is the single biggest reason they are faster, lighter, and more efficient than VMs.

---

## 🔑 Key Docker Concepts

### 🖼️ Docker Image

> A Docker image is a **lightweight, standalone, executable package** that includes everything needed to run a piece of software — code, runtime, libraries, environment variables, and configuration files.

**Three core properties:**

| Property | What it means |
|---|---|
| 📚 **Layered** | Built as stacked read-only layers — each instruction in a Dockerfile adds one layer |
| 🔒 **Immutable** | Cannot be changed after build — guarantees the same result every time |
| 🌍 **Portable** | Runs on any machine with Docker installed, always the same result |

**How layers stack — Node.js app example:**

```
┌──────────────────────────────────────────┐
│  Layer 4  │  Your app code               │  ← writable at runtime
├──────────────────────────────────────────┤
│  Layer 3  │  npm dependencies            │  ← read-only
├──────────────────────────────────────────┤
│  Layer 2  │  Node.js runtime             │  ← read-only
├──────────────────────────────────────────┤
│  Layer 1  │  Base OS (Alpine / Ubuntu)   │  ← read-only
└──────────────────────────────────────────┘
```

> Each layer is cached. If only your app code changes, Docker rebuilds only Layer 4 — not the entire image. This is what makes Docker builds fast.

---

### 📦 Docker Container

> A container is a **runtime instance of a Docker image** — a lightweight, standalone, executable package that includes everything needed to run a piece of software.

**Image → Container relationship:**

```
  🖼️ Image                                   📦 Container
  ──────────────────────────────────────────────────────
  Read-only blueprint             →           Live, running process
  Like a CLASS in OOP             →           Like an OBJECT in OOP
  Stored in a registry            →           Running on your host machine
  One image                       →           Can run as MANY containers simultaneously
```

**Three core properties:**

| Property | What it means |
|---|---|
| 🔒 **Isolation** | Own filesystem, network, and process space — fully separated from other containers |
| ⚡ **Ephemeral** | Temporary by nature — data is gone when the container is removed (use volumes to persist) |
| 🌍 **Portable** | Runs identically on any machine that has Docker installed |

**Container lifecycle:**

```
  [Created] ──▶ [Running] ──▶ [Stopped] ──▶ [Removed]
```

---

### 📄 Dockerfile

> A Dockerfile is a **plain text file with step-by-step instructions** that Docker reads to automatically build an image. Every instruction adds a new layer.

**Common instructions:**

| Instruction | Purpose |
|---|---|
| `FROM` | Set the base image to build on top of |
| `WORKDIR` | Set the working directory inside the container |
| `COPY` | Copy files from the host into the image |
| `RUN` | Execute a command at build time |
| `EXPOSE` | Declare the port the app listens on |
| `CMD` | Default command to run when the container starts |
| `ENV` | Set environment variables inside the image |

**Example — Node.js app:**

```dockerfile
# 1. Base image
FROM node:18-alpine

# 2. Set working directory
WORKDIR /app

# 3. Install dependencies (copy package files first for layer caching)
COPY package*.json ./
RUN npm install

# 4. Copy source code
COPY . .

# 5. Expose port and define start command
EXPOSE 3000
CMD ["node", "index.js"]
```

**Dockerfile → Image → Container:**

```
  📄 Dockerfile
       │
       │  docker build -t myapp .
       ▼
  🖼️ Docker Image ──── docker push ────▶ 📡 Registry (Docker Hub / ECR / ACR)
                                                      │
                                         docker pull  │
                                                      ▼
                                              📦 Container
```

---

## 🏗️ Docker Architecture

### The Three Components

```
  ┌──────────────────┐              ┌──────────────────────────────────────────┐
  │                  │              │              DOCKER HOST                 │
  │  CLIENT  (CLI)   │   REST API   │                                          │
  │                  │ ◀──────────▶ │  ┌────────────────────────────────────┐  │
  │  docker build    │              │  │      Docker Daemon  (dockerd)      │  │
  │  docker run      │              │  │                                    │  │
  │  docker push     │              │  │  Manages: Images · Containers      │  │
  │  docker pull     │              │  │           Networks · Volumes       │  │
  │  docker stop     │              │  └────────────────┬───────────────────┘  │
  │                  │              │                   │                      │
  └──────────────────┘              │      ┌────────────┴────────────┐         │
                                    │      │  Containers  │  Images   │         │
                                    │      └─────────────────────────┘         │
                                    └─────────────────────────┬────────────────┘
                                                              │  push / pull
                                                              ▼
                                                ┌─────────────────────────┐
                                                │        REGISTRY         │
                                                │  Docker Hub · ECR · ACR │
                                                │  JFrog · Nexus ...      │
                                                └─────────────────────────┘
```

### Docker Engine — Internal Components

| Component | Role | Analogy 🚗 |
|---|---|---|
| **Docker CLI** | The interface where commands are typed | Dashboard — buttons and controls |
| **REST API** | Translates CLI commands into HTTP requests for the Daemon | Wires — carry signals to the engine |
| **Docker Daemon** | Does all the actual work — pulls images, builds, manages containers | Engine — powers everything |

---

## 🔄 Docker Workflow

### High-Level

```
  Developer                       Registry                      Server
  ─────────                       ────────                      ──────
  Write Dockerfile
       │
       │  $ docker build
       ▼
  🖼️ Docker Image ─── $ docker push ───▶ 📡 Docker Hub / Private Registry
                                                     │
                                        $ docker pull│
                                                     ├──────────▶  🧪 Test Env
                                                     │
                                                     └──────────▶  🚀 Prod Env
```

### Low-Level — Step by Step

```
  1.  Developer writes Dockerfile → commits to version control (Git)

  2.  $ docker build
         └─▶ Docker reads Dockerfile line by line
         └─▶ Each instruction creates a new image layer
         └─▶ Result: a local Docker Image

  3.  $ docker push
         └─▶ Image uploaded to a Registry
         └─▶ (Docker Hub, AWS ECR, ACR, JFrog, Nexus...)

  4.  $ docker pull
         └─▶ DevOps / SRE team pulls image onto the target server

  5.  $ docker run
         └─▶ Container starts from the image
         └─▶ Runs identically in Test and Production
```

---

## ✅ Summary

| Concept | One-line summary |
|---|---|
| **Container** | Isolated, lightweight, portable runtime environment for an app |
| **Image** | Immutable, layered blueprint used to create containers |
| **Dockerfile** | Plain text recipe of instructions to build a Docker image |
| **Docker Daemon** | Background service that manages everything on the host |
| **Registry** | Remote storage for Docker images (Docker Hub, ECR, ACR…) |
| **Key advantage over VMs** | No Guest OS — containers share the host kernel, far lighter and faster |

---

<div align="center">

**← [Back to Index](../README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Day 2 →](../Day-02/README.md)**

---


</div>