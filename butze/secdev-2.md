# devenv — Design Summary (concept draft) (Second MD file)

This document captures the architecture and intended behavior of a lightweight
local development environment orchestration system built around:

- YAML state
- filesystem-based capability delegation
- pluggable provider executables
- ephemeral runtime state
- SSH-based interaction

The system is designed to prioritize:
- usability (single command entrypoint)
- reproducibility where needed
- strict isolation of development workloads
- minimal core complexity

---

# 1. Core Philosophy

The system is built on four principles:

### 1.1 Environments are disposable cache
Everything except configuration and explicitly mounted state is ephemeral.

- VMs/containers are recreated at will
- overlays are rebuildable
- runtime state is never trusted

---

### 1.2 The core is not an orchestrator of semantics
The core does NOT interpret:

- images
- runtimes
- credentials
- provider logic

It only:
- reads YAML
- resolves filesystem mounts
- executes provider binaries
- attaches via SSH

---

### 1.3 Providers own all execution semantics
Providers decide:

- how images are built
- how VMs/containers are created
- how networking works
- what "image" means

The core does not interpret `image`.

---

### 1.4 The filesystem is the VFS graph
All capability delegation is expressed via:

- directories
- symlinks (preferred)
- optional hardlinks (discouraged for clarity)

---

# 2. Core State File

## ~/.local/state/devenv.yaml

Central declarative state file defining environments ("boxes").

```yaml
boxes:
  cluster-admin:
    provider: nixos
    image: k8s-tooling
    credentials:
      - kubeconfig
      - talosconfig
      - claude-code
    output: true

  random-npm:
    provider: docker
    image: my-amazing-npm-sinkhole
    credentials:
      - claude-code
```

### Semantics

- `provider`: maps to executable in providers registry
- `image`: opaque string passed to provider
- `credentials`: mounted read-only unless explicitly marked RW
- `output`: optional writable mount

No validation of meaning is performed by core.

---

# 3. Provider Registry

## ~/.local/state/devenv/providers.yaml

Maps provider names to executable paths.

```yaml
providers:
  nixos: /home/foo/.local/share/devenv/providers/nixos/bar
  docker: /home/foo/.local/share/devenv/providers/docker/run
```

### Versioning model

Versioning is not a core concept.

Instead:

```yaml
providers:
  nixos: /providers/nixos/v24/run
  old-nixos: /providers/nixos/v23/run
```

Provider identity = filesystem path.

---

# 4. Provider Interface

Providers are executables.

They receive a standardized request (e.g. JSON via stdin):

### Example request

```json
{
  "box": "cluster-admin",
  "image": "k8s-tooling",
  "mounts": {
    "creds": "/mnt/creds",
    "output": "/mnt/output"
  }
}
```

### Expected responses

#### SUCCESS
```json
{
  "status": "SUCCESS",
  "ip": "10.0.0.5",
  "port": 22
}
```

#### OUTDATED
```json
{
  "status": "OUTDATED"
}
```

Core may retry with `--outdated`.

#### FAIL
```json
{
  "status": "FAIL",
  "reason": "permission denied"
}
```

---

# 5. Core Lifecycle Model

## 5.1 Primary command: `devenv enter`

This is the only command intended for daily use.

Behavior:

1. Load `devenv.yaml`
2. Load `providers.yaml`
3. Resolve box definition
4. Ensure runtime exists:
   - base image exists
   - overlays exist
   - credentials mounted
   - cache present
5. Call provider
6. Receive endpoint (IP + port)
7. SSH attach into environment

---

## 5.2 Ensure model (Makefile-like)

Each box behaves like:

```text
ensure:
  - base image exists
  - overlay exists
  - credentials mounted
  - VM/container running
```

Missing components are created automatically.

---

## 5.3 Build model

Explicit rebuild:

```bash
devenv build <box>
```

Behavior:
- forces provider rebuild of image/runtime
- does NOT attach
- does NOT modify other boxes

---

## 5.4 Reset model

```bash
devenv reset <box>
```

Behavior:
- destroys runtime state
- recreates overlays
- reinitializes VM/container

---

## 5.5 Delete model

```bash
devenv delete <box>
```

Behavior:
- removes runtime state
- removes overlays
- does NOT delete provider definitions

---

## 5.6 Global reset

```bash
devenv reset --all
```

Behavior:
- stops all boxes
- wipes all overlays in:
  ~/.local/cache/devenv/
- recreates everything on next entry

---

# 6. Cache Model

All runtime state is ephemeral:

```text
~/.local/cache/devenv/<box>/
```

Contains:
- VM disks
- container state
- overlays
- runtime artifacts

This directory is safe to delete entirely.

---

# 7. Credential System

Credentials are mounted as VFS directories.

Example structure:

```text
~/.local/state/devenv/creds/
  kubeconfig
  claude-code.json
  github-copilot.json
  gemini.json
```

### Mount rules

- default: read-only mounts
- optional: `--creds-rw` enables writable mounts
- credentials are explicitly selected per box

---

# 8. Output Mount

Optional writable directory:

```text
~/.local/state/devenv/<box>/output
```

Used for:
- build artifacts
- LaTeX output
- file transfer to host

Not enabled by default.

---

# 9. SSH Interaction Model

After provider startup:

- provider returns IP + port
- core SSH attaches automatically
- no manual docker/VM interaction required

Primary UX:

```bash
devenv enter repo
```

---

# 10. Image Model

`image` is an opaque string passed directly to providers.

Examples:
- Docker image name
- NixOS flake reference
- directory name containing Dockerfile
- base64-encoded blob (provider-defined)

Core does NOT interpret image semantics.

---

# 11. Update Behavior

On `enter`:

- check image age
- if older than threshold (e.g. 7 days):
  - OFFER update
  - never force update

Example prompt:

```
Base image is 12 days old.
Update now? [y/N]
```

Default is NO.

---

# 12. Overlay Model

Two-layer system:

### System overlay
- OS-level changes
- packages
- runtime config

### Data overlay
- caches
- build artifacts
- tool state

Reset behavior can target either layer independently.

---

# 13. Providers as Execution Units

Providers are fully independent executables:

- bash scripts
- Go binaries
- Nix derivations
- Docker wrappers
- Kata/Firecracker adapters

Core only enforces:
- input format
- output format
- execution contract

---

# 14. Key Architectural Properties

### 14.1 Minimal core
Core only:
- reads YAML
- resolves filesystem
- executes provider
- attaches via SSH

---

### 14.2 Fully pluggable runtime
All runtime logic is delegated.

---

### 14.3 Reconstructible environments
Everything except explicitly persisted data is cache.

---

### 14.4 Capability-based security model
Security is expressed via:

- mounted credentials
- directory presence
- explicit opt-in flags

---

### 14.5 “enter is enough” UX goal
All complexity is hidden behind:

```bash
devenv enter <box>
```

---

# 15. Mental Model Summary

```text
YAML (state)
   ↓
Core (orchestrator)
   ↓
Provider (execution engine)
   ↓
VM/container runtime
   ↓
SSH session
```

and:

```text
Credentials = mounted capabilities
Images = provider-defined artifacts
Cache = disposable state
```

---

# 16. Non-goals

- no enterprise multi-tenancy
- no universal container standardization
- no central policy engine
- no enforced image semantics
- no heavy plugin framework

---

# 17. Core Design Intent

This system is not trying to be:

- Kubernetes
- Docker replacement
- VM manager
- cloud orchestrator

It is:

> a local, capability-scoped execution system for disposable development environments

optimized for:
- speed of entry
- isolation of workloads
- minimal cognitive overhead in daily use