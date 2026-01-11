# Docker Build Debugging & Optimization Lab

## Purpose of This Lab
This lab is designed to teach students **how Docker really behaves on a developer workstation**, why builds sometimes appear “stuck,” and how system resources, Docker configuration, and Dockerfile design directly affect developer experience.

This README intentionally walks through a *real failure sequence* encountered on a modest local machine and contrasts it with a properly tuned setup and a cloud-based virtual environment (CloudShare).

By the end of this lab, students will be able to:
- Diagnose Docker build failures and "hangs"
- Understand Docker + WSL2 architecture on Windows
- Optimize Dockerfiles for build speed
- Correctly use `.dockerignore`
- Explain why first builds are slow and subsequent builds are fast
- Apply debugging techniques using build logs
- Understand why builds run faster in a virtual lab environment

---

## System Context (Instructor Machine)

**Host System**
- CPU: Intel i7 @ 2.6 GHz
- RAM: 12 GB
- OS: Windows 10/11
- Docker Runtime: Docker Desktop (WSL2 backend)
- Development Shell: WSL2 Ubuntu

This is a *perfectly reasonable* development machine, but it is **resource-constrained relative to Docker’s default assumptions**.

---

## What Actually Happened (Root Cause Analysis)

### 1. Docker Desktop + WSL2 Architecture
On Windows, Docker does **not** run natively. Instead:

1. Docker Desktop runs a lightweight Linux VM
2. WSL2 runs its own Linux VM
3. Your shell runs *inside WSL2*
4. Docker CLI talks across VM boundaries to the Docker daemon

This means:
- CPU scheduling is virtualized
- Disk I/O crosses filesystem boundaries
- Memory pressure is amplified

On machines with 16–32 GB RAM this is less visible. On a 12 GB system, it matters.

---

### 2. Why Docker Felt Like a “Processor Hog”

By default, Docker Desktop:
- Assumes it can use **most available CPU cores**
- Assumes it can consume **most available RAM**

During a `docker build`, especially with Node.js:
- `npm install` performs CPU-heavy dependency resolution
- Many small files are written (I/O heavy)
- Compilation steps are CPU-bound

Without limits, Docker competes directly with:
- Windows
- WSL2
- Your browser
- Your IDE

This causes the system to feel frozen or unresponsive.

---

## Why Throttling Docker Was Necessary

### Correct Docker Desktop Resource Limits

On this system, Docker must be *intentionally constrained*.

Recommended settings:
- **CPU**: 50–60% of available cores
- **Memory**: 5–6 GB maximum
- **Swap**: Minimal or disabled
- **WSL Integration**: Enable only the active distro

This ensures:
- The OS remains responsive
- Docker builds progress steadily
- Students can observe behavior instead of fighting the machine

This is not a workaround — it is **proper capacity planning**.

---

## Why the Build Looked “Stuck”

The original build used:

```
RUN npm install && npm install -g serve && npm run build
```

Problems:
- No progress output
- Long-running step
- CPU-heavy
- Disk-heavy

Docker was working correctly, but **provided no visible feedback**.

This is a *developer experience problem*, not a failure.

---

## Why Build Logs Matter

Docker build logs are the **only visibility** you have into what is happening inside a container.

Using:
```
docker build --progress=plain .
```

Forces Docker to:
- Emit every step
- Show cache usage
- Reveal slow layers

This single flag turns Docker from a black box into a teachable system.

---

## Why First Builds Are Always Slow

First builds are slow because Docker must:
1. Pull the base image (`node:22-alpine`)
2. Resolve and download dependencies
3. Compile assets
4. Populate build cache

Subsequent builds reuse cached layers:

```
#7 WORKDIR /app        → cached
#8 COPY package*.json  → cached
#11 RUN npm install    → cached
```

Docker is deterministic. Speed comes from **layer reuse**.

---

## Optimizing the Dockerfile (Correct Pattern)

### Key Principles
- Copy dependency manifests first
- Install dependencies before application code
- Remove unnecessary artifacts
- Keep layers stable

### Revised Optimized Dockerfile

```
FROM node:22-alpine

WORKDIR /app

# Copy only dependency manifests first
COPY package*.json ./

# Install dependencies (cached unless package.json changes)
RUN npm ci --progress=false

# Copy application source
COPY src ./src
COPY public ./public

# Build the application
RUN npm run build

# Install lightweight static server
RUN npm install -g serve@latest \
 && rm -rf node_modules

EXPOSE 3000
CMD ["serve", "-s", "build"]
```

This structure maximizes cache reuse and minimizes rebuild time.

---

## Why `.dockerignore` Is Critical

Without `.dockerignore`, Docker:
- Sends the *entire directory* as build context
- Includes:
  - `node_modules`
  - `.git`
  - Logs
  - Build artifacts

This dramatically slows builds and increases memory usage.

### Required `.dockerignore`

```
node_modules
.git
.gitignore
Dockerfile
README.md
npm-debug.log
build
.env
```

This reduces build context size and improves performance.

---

## Debug Lab: “Diagnose a Stuck Docker Build”

### Lab Objective
Students will intentionally trigger and diagnose a slow Docker build.

### Steps
1. Remove `.dockerignore`
2. Run `docker build .`
3. Observe slow or silent behavior
4. Interrupt build (`Ctrl+C`)
5. Re-run with:
   ```
   docker build --progress=plain .
   ```
6. Identify slow layers
7. Restore `.dockerignore`
8. Rebuild and compare timing

### Learning Outcomes
- Understand Docker build context
- Learn to read build logs
- See cache behavior
- Experience real-world debugging

---

## Why This Works Faster in CloudShare

CloudShare environments:
- Run Docker natively on Linux
- Have dedicated CPU cores
- Have predictable I/O
- Do not traverse Windows/WSL boundaries

Result:
- Faster builds
- Stable performance
- Consistent student experience

This is why **local builds are for learning**, and **cloud builds are for scale**.

---

## Revised Fix for This Instructor Setup

### Required Changes
1. Keep Docker resource limits conservative
2. Always use `--progress=plain`
3. Optimize Dockerfile layer ordering
4. Use `.dockerignore`
5. Expect first build slowness

### Correct Build Command

```
docker build --progress=plain -t welcome-to-docker .
```

This setup is now stable, performant, and classroom-ready.

---

## Next Module Preview: MCP Integration

In the next module, this container will be:
- Converted into an MCP-compatible service
- Exposed to Claude Desktop
- Used as a controlled AI tool

Students will learn **how containers become AI tools**, not just applications.

---



