# 🐳 Day 5 — Docker Multi-Stage Builds & Image Optimization

<div align="center">

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![CKA](https://img.shields.io/badge/CKA-Prep-F59E0B?style=for-the-badge&logo=kubernetes&logoColor=white)
![Day](https://img.shields.io/badge/Day-05-10B981?style=for-the-badge)

</div>

---

## 📌 Topics Covered

- [Compiled vs Interpreted Languages](#-compiled-vs-interpreted-languages)
- [Why Multi-Stage Builds?](#-why-multi-stage-builds)
- [Build Stage vs Runtime Stage](#-build-stage-vs-runtime-stage)
- [Multi-Stage Dockerfile Example](#-multi-stage-dockerfile-example)
- [How Multi-Stage Builds Work](#-how-multi-stage-builds-work)
- [Layer Reference — Creates vs Does Not Create](#-layer-reference--creates-vs-does-not-create)
- [Best Practices](#-best-practices)
- [Summary](#-summary)

---

## 💻 Compiled vs Interpreted Languages

Understanding this distinction is important because it directly affects how you structure your Dockerfile — specifically whether you need a separate build stage.

### Compiled Languages

- Source code is **translated into machine code (binary) before execution**
- Requires a **build stage** with compilers and build tools
- Faster execution — binary interacts directly with machine hardware
- Less portable — binary may not run on different architectures without recompilation
- **Examples:** Go, Java, C, C++

### Interpreted Languages

- Source code is **executed directly by an interpreter at runtime**
- No separate compilation step required
- Slower execution — interpreter processes code on the fly
- More portable — runs on any system that has the interpreter installed
- **Examples:** Python, JavaScript (Node.js), PHP, Ruby

### Impact on Docker

```
  Compiled Language (e.g. Go)              Interpreted Language (e.g. Python)
  ───────────────────────────              ──────────────────────────────────
  Stage 1: Build stage                     No separate build stage needed
    → Install compiler + tools             → Just install interpreter + deps
    → Compile source → binary

  Stage 2: Runtime stage                   Runtime stage
    → Copy only the binary                 → Copy source code
    → No compiler needed                   → Run with interpreter
    → Tiny final image                     → Still benefits from multi-stage
```

---

## 🤔 Why Multi-Stage Builds?

When building Docker images, every `FROM`, `RUN`, `COPY`, and `ADD` instruction creates a new layer — adding to the final image size. Build tools, compilers, and development dependencies that are needed to **build** the app are almost never needed to **run** it.

Without multi-stage builds:

```
  Single Stage Dockerfile
  ────────────────────────
  Base image
  + Compiler / build tools       ← needed at build time only
  + Build dependencies           ← needed at build time only
  + Source code
  + Runtime dependencies
  ─────────────────────────
  = BLOATED production image
    → slow downloads
    → larger attack surface
    → unnecessary tools in prod
```

With multi-stage builds:

```
  Stage 1: Build                 Stage 2: Runtime
  ──────────────────             ─────────────────
  Base image                     Slim base image
  + Compiler / build tools       + Runtime deps only
  + Build dependencies           + Compiled output (copied from Stage 1)
  + Source code                  ─────────────────────────────────────
  ↓ compile / build              = LEAN production image ✅
  Output artifacts
        │
        │ COPY --from=build
        ▼
  Only what is needed
```

---

## 🏗️ Build Stage vs Runtime Stage

| Aspect | Build Stage | Runtime Stage |
|---|---|---|
| **Purpose** | Prepare the application for deployment | Provide an environment to run the app |
| **Tools included** | Compilers, build tools, dev dependencies (e.g. `build-essential`) | Only essential runtime libraries (e.g. Flask) |
| **Image size** | Larger — includes everything needed to build | Smaller — optimised for deployment |
| **Goes to production** | ❌ Discarded after build | ✅ This is the final image |

### The Pizza Analogy

```
  Build Stage                              Runtime Stage
  ───────────                              ─────────────
  Making the pizza:                        Eating the pizza:
  → measuring cups                         → just the finished pizza
  → mixing bowls
  → oven                                   The tools and raw ingredients
  → raw dough, cheese, toppings            are no longer needed.

  Essential for preparation.               Only the result matters.
  Never served at the table.
```

### Benefits of separating stages

- **Smaller production images** — deploy faster, use fewer resources
- **Improved security** — build tools and compilers are not present in production, reducing attack surface
- **Better debugging** — build issues and runtime issues are clearly isolated

---

## 📄 Multi-Stage Dockerfile Example

A Python Flask application using multi-stage builds:

```dockerfile
# ── Stage 1: Build ────────────────────────────────────────────────
# Named "build" so the next stage can reference it
FROM python:3.9-slim AS build

# Set working directory
WORKDIR /app

# Copy application source
COPY app.py app.py

# Install Flask and build tools (build-essential not needed at runtime)
RUN pip install flask && apt update && apt install -y build-essential


# ── Stage 2: Runtime ──────────────────────────────────────────────
# Fresh slim image — no build tools carried over
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy only what we need from the build stage
COPY --from=build /app .

# Install only runtime dependency
RUN pip install flask

# Expose the app port
EXPOSE 5000

# Fixed command + default argument
ENTRYPOINT ["python"]
CMD ["app.py"]
```

### What `COPY --from=build` does

```
  Stage 1 (build)                        Stage 2 (runtime)
  ───────────────                        ─────────────────
  /app/
  ├── app.py              ──────────▶    /app/
  ├── build-essential                    ├── app.py   ✅ copied
  └── compiler tools                     └── (nothing else)

                                         build-essential ❌ not copied
                                         compiler tools  ❌ not copied
```

### Layer breakdown

```
Stage 1 (discarded after build):
  FROM python:3.9-slim AS build    ← base layer
  WORKDIR /app                     ← layer
  COPY app.py                      ← layer
  RUN pip install + apt install    ← layer (includes build-essential)

Stage 2 (this becomes the final image):
  FROM python:3.9-slim             ← fresh base layer
  WORKDIR /app                     ← layer
  COPY --from=build /app .         ← layer (only app files, no build tools)
  RUN pip install flask            ← layer (runtime only)
  EXPOSE 5000                      ← layer
  ENTRYPOINT ["python"]            ← metadata only
  CMD ["app.py"]                   ← metadata only
```

---

## ⚙️ How Multi-Stage Builds Work

- A Dockerfile can contain **multiple `FROM` instructions** — each one starts a new stage
- Each stage can be **named** using `AS <name>` for easy reference
- Files can be copied **between stages** using `COPY --from=<stage-name>`
- Only the **final stage** becomes the production image — all previous stages are discarded
- Discarded stages leave **no trace** in the final image — not even as hidden layers

```
  FROM image AS stage1      ← Stage 1 starts here
  ...instructions...

  FROM image AS stage2      ← Stage 2 starts here (Stage 1 discarded)
  COPY --from=stage1 ...    ← Pull specific files from Stage 1
  ...instructions...

  Final image = Stage 2 only
```

---

## 🧱 Layer Reference — Creates vs Does Not Create

| ✅ Creates a new layer | ❌ Metadata only — no layer |
|---|---|
| `FROM` | `CMD` |
| `RUN` | `ENTRYPOINT` |
| `COPY` | `ENV` |
| `ADD` | `LABEL` |
| | `USER` |
| | `VOLUME` |
| | `EXPOSE` |
| | `WORKDIR` |
| | `ARG` |
| | `STOPSIGNAL` |

> 📝 Note: In Day 3 and Day 4 notes, `WORKDIR`, `VOLUME`, and `EXPOSE` were listed as layer-creating. The reference material for Day 5 classifies them as non-layer-creating. This is a nuanced point — they do create filesystem entries but are often grouped with metadata instructions. Worth revisiting as your understanding deepens.

---

## ✅ Best Practices

### 1. Combine `RUN` commands to reduce layers

```dockerfile
# ❌ Bad — 3 separate layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ Good — 1 layer
RUN apt-get update && apt-get install -y curl && apt-get clean
```

### 2. Use `.dockerignore` to reduce build context size

```
# .dockerignore
node_modules
*.log
.git
.env
__pycache__
*.pyc
```

### 3. Always use multi-stage builds for production images

```dockerfile
# Development — single stage is fine
FROM python:3.9-slim
...

# Production — always use multi-stage
FROM python:3.9-slim AS build
...
FROM python:3.9-slim
COPY --from=build ...
```

### 4. Name your stages clearly

```dockerfile
FROM python:3.9-slim AS builder     ← descriptive names
FROM python:3.9-slim AS production
```

---

## 📋 Summary

| Concept | Key takeaway |
|---|---|
| **Compiled languages** | Need a build stage — compiler not needed at runtime |
| **Interpreted languages** | No compilation needed — still benefit from multi-stage to strip dev tools |
| **Build stage** | Contains all tools needed to build the app — discarded after |
| **Runtime stage** | Lean, production-ready image — only what the app needs to run |
| **Multi-stage builds** | Multiple `FROM` instructions — only the final stage becomes the image |
| **`COPY --from`** | Copy specific files from a previous stage into the current one |
| **Key benefit** | Smaller images, faster deployments, reduced attack surface |
| **Layer-creating** | `FROM` `RUN` `COPY` `ADD` |
| **Metadata only** | `CMD` `ENTRYPOINT` `ENV` `LABEL` `USER` `ARG` `VOLUME` `EXPOSE` `WORKDIR` |
| **Best practice** | Combine `RUN` commands, use `.dockerignore`, always multi-stage in production |

---

<div align="center">

**← [Day 4](../Day-04/README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Back to Index](../README.md)** &nbsp;&nbsp;|&nbsp;&nbsp; **[Day 6 →](../Day-06/README.md)**

</div>