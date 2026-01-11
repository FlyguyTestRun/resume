# Docker Foundations: Canonical Project Layout, Build Debugging, and Data Persistence Lab

## Audience
This module is designed for students learning Docker Desktop in a VM-based classroom environment and for instructors preparing reproducible, low-friction development labs.

---

## Learning Objectives
By the end of this lab, students will be able to:

- Explain the difference between host systems and containerized environments
- Use a canonical Docker project layout
- Diagnose slow or “stuck” Docker builds using build logs
- Explain why `.dockerignore` is critical to performance
- Build and run a Python application using Docker Init
- Understand how bind mounts and volumes persist data
- Explain why Docker behaves differently on local laptops vs. CloudShare VMs

---

## Canonical Project Layout (Required)

All Docker-based projects in this course will follow **this exact structure**:

```
python-docker-app/
├── .dockerignore
├── Dockerfile
├── compose.yaml
├── requirements.txt
├── main.py
├── README.Docker.md
└── README.md   ← (this document)
```

### Why this layout matters

- Keeps Docker build contexts small
- Prevents accidental inclusion of large host directories
- Ensures reproducibility across student machines
- Mirrors CloudShare VM layouts

**Rule:** Never run `docker build` or `docker compose` from a parent directory containing unrelated files.

---

## Environment Context

### Student Hardware (Local)
- i7-class CPU (~2.6GHz)
- 12 GB RAM
- Docker Desktop running via WSL2

### Classroom Environment (CloudShare)
- Dedicated CPU and disk IO
- Minimal file sprawl
- Pre-sized project directories

**Result:** Docker builds appear significantly faster in CloudShare due to reduced build context and optimized IO.

---

## Step 1: Containerizing a Python App with Docker Init

We use Docker Init to scaffold a containerized Python application.

### Selected options
- **Platform:** Python
- **Python version:** 3.12.3 (container-only)
- **Port:** 8000
- **Run command:** `python main.py`

### Important Clarifications

- Selecting a Python version does **not** install Python locally
- The port defines where the app listens **inside the container**
- The run command defines the container lifecycle

---

## Step 2: Understanding Generated Files

### `.dockerignore`

Controls what files are sent to Docker during a build.

Minimum required contents:

```
.git
__pycache__/
*.pyc
.env
node_modules/
dist/
build/
.obsidian/
cloudshare/
*.log
```

**Impact:** Reduces build time, CPU usage, and disk IO.

---

### `Dockerfile`

Defines:
- Base image (Python runtime)
- Working directory
- File copy order
- Dependency installation
- Startup command

This is the immutable blueprint for the container image.

---

### `compose.yaml`

Defines runtime configuration:
- Port mapping (`8000:8000`)
- Service name
- Future multi-container expansion

This enables a single-command startup:

```
docker compose up --build
```

---

## Step 3: Why the First Build Is Slow

### Observed behavior

```
transferring context: 7.25GB
```

### Explanation

Docker sends the **entire build context** to the build engine. Without a proper `.dockerignore`, this includes:

- Git repositories
- Node modules
- VM artifacts
- Personal files

This overwhelms:
- CPU
- Disk IO
- Memory

### Fix

- Always build from a clean project directory
- Always define `.dockerignore` before building

---

## Step 4: Debugging a Failed Build (Lab Exercise)

### Failure Scenario

```
failed to calculate checksum: "/requirements.txt": not found
```

### Root Cause

The Dockerfile uses a BuildKit bind mount:

```
--mount=type=bind,source=requirements.txt,target=requirements.txt
```

This requires `requirements.txt` to exist **at build time** in the build context root.

### Resolution Steps

1. Ensure `requirements.txt` exists
2. Ensure Docker is run from the correct directory
3. Re-run:

```
docker compose up --build
```

---

## Step 5: Bind Mounts and Data Persistence

### What is a bind mount?

A bind mount maps a host directory directly into a container.

Example:

```
volumes:
  - ./app:/usr/src/app
```

### Common mistake (Observed)

```
- /usr/src/app/node_modules
```

This is interpreted as a service dependency, not a volume.

### Correct pattern

```
volumes:
  - ./app:/usr/src/app
  - node_modules:/usr/src/app/node_modules
```

---

## Step 6: Understanding Container Logs and Exit States

### Example

```
INFO Accepting connections at http://localhost:3000
INFO Gracefully shutting down
ExitCode: 0
```

### Explanation

- The application started successfully
- The primary process exited
- Docker shut down the container cleanly

**Containers stop when their main process stops.**

---

## Language Comparison for Containers (Conceptual)

### Python
- Fast to learn
- Ideal for teaching and prototyping
- Large ecosystem

### Node.js
- Event-driven, non-blocking IO
- Excellent for real-time and frontend-backed services
- Strong Docker ecosystem

### Rust
- Memory safety without garbage collection
- Extremely fast startup and low resource usage
- Ideal for MCP servers and infrastructure tooling

**Teaching choice:** Python first, Node and Rust later.

---

## Why Docker Feels Faster in CloudShare

- Smaller build contexts
- Dedicated CPU resources
- No background applications
- Predictable IO performance

This reinforces the importance of environment hygiene.

---

## Lab Completion Checklist

- [ ] Canonical project layout created
- [ ] `.dockerignore` defined
- [ ] Image builds without errors
- [ ] Application accessible at `http://localhost:8000`
- [ ] Student can explain why the first build was slow

---

## Next Module

**Persisting data between containers using named volumes and bind mounts**, followed by **MCP-enabled tool integration**.

---

_End of Module 1_

