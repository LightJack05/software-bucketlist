# devenv — Design Specification v3

Status: draft for implementation
Supersedes: `secdev.md` (threat model & tier reasoning — still valid as rationale),
`secdev-2.md` (devenv concept draft — superseded by this document)

devenv is a local, capability-scoped execution system for disposable development
environments. It is a thin core that reads declarative config, resolves host-side
resources, executes pluggable provider binaries, and attaches via SSH. All execution
semantics live in providers.

> **North star (unchanged from secdev): friction *is* security.** The secure path must
> be the invisible, lowest-effort path. `devenv enter <box>` must be fewer keystrokes
> than doing it wrong on the metal. Adoption rate is the security property.

---

## 1. Core principles

1. **Environments are disposable cache.** Everything except user-authored config and
   explicitly persistent disks is rebuildable and may be destroyed at any time.
2. **The core interprets no semantics.** It does not know what an image, a runtime, or
   a credential *means*. It reads YAML, resolves host paths, executes providers,
   enforces the protocol, and attaches via SSH.
3. **Providers own all execution semantics** — image building, VM/container creation,
   networking, disk mechanics.
4. **The tier is the policy.** Security posture is not a global setting and not
   free-form per-box YAML; it is declared by the provider in a static posture manifest
   (§5.7). The core mechanically enforces the manifest. Overrides are explicit,
   per-box, and grep-able.
5. **Trust-routing rule:** which provider a task goes to is decided by *trust of the
   code*, not type of work. Untrusted third-party code only ever runs in the hardened
   ephemeral tier. The tool's job is to make that the path of least resistance.

### Direction-of-flow test (applied to every new channel)

> Which direction does data flow across the boundary, and does anything trusted parse
> or execute it on the far side?
> - untrusted → trusted, host executes/parses: **dangerous** — requires justification
> - trusted → untrusted: safe (host hands a capability in)
> - untrusted side merely displays: low risk

---

## 2. Directory layout (XDG-honest)

The sorting rule: **can it be regenerated without loss?** Yes → cache. No → state.
User-authored → config. This property is load-bearing: `rm -rf ~/.local/cache/devenv`
must always be safe, and everything that survives a reset must be enumerable from
config + state.

```text
~/.config/devenv/
  boxes.yaml                      # user-authored box definitions (THE config)
  providers.yaml                  # provider registry

~/.local/state/devenv/
  machines/<box>/
    data.qcow2                    # persistent disks (never touched by reset)
    <extra-disk>.qcow2            # additional disks declared in boxes.yaml
    ssh_host_ed25519_key{,.pub}   # per-box VM host identity (core-generated)
  creds/                          # host-side secret inventory, mode 0700
    <credential files / dirs>
  tokens.yaml                     # issued-token registry (for gc/revoke)

~/.local/cache/devenv/            # safe to delete entirely, always
  images/<provider>/<image>/<version>/   # versioned base images
    root.qcow2 ...
  machines/<box>/
    root-overlay.qcow2            # disposable runtime disks
    last-request.json             # most recent provider request (debugging)
    provider.log                  # captured stderr from last operation
```

Notes:
- Base images are **versioned** (`<version>/` level). Even if v1 uses an eager update
  strategy (§8), versioning keeps the deferred strategy open and makes gc decidable.
- `output/` mounts (build artifacts) live under `cache/machines/<box>/output/` —
  losing them must be acceptable by definition; if it isn't, it belongs on a
  persistent disk.

---

## 3. Box configuration — `~/.config/devenv/boxes.yaml`

```yaml
boxes:
  cluster-admin:
    provider: nixos
    image: "git+https://gitea.example/me/images?dir=k8s-tooling"   # opaque to core
    credentials:
      - kubeconfig
      - claude-code
    disks:                        # persistent disks beyond the implicit root
      - name: data
        size: 20G
      - name: scratch
        size: 100G
    output: false                 # bidirectional artifact mount (posture-gated)

  kernel-pet:
    provider: ubuntu
    image: "noble"
    credentials: []
    disks:
      - name: data
        size: 50G
    kernel_cmdline: "lockdown=integrity"   # only valid if posture allows (§5.7)
```

Semantics:
- `provider` — key into `providers.yaml`.
- `image` — opaque string passed verbatim to the provider. The core never inspects it.
- `credentials` — names resolved under `state/creds/`, mounted **read-only by
  default**; read-write requires `mode: rw` per credential *and* posture permission.
- `disks` — persistent volumes. Core ensures the volume file exists (provider creates
  on first `ensure`), passes them in the request. The disk **name doubles as the
  filesystem label** — that label is the contract with the guest config (e.g. a NixOS
  flake mounting `/dev/disk/by-label/scratch` with `nofail` + `autoFormat`).
- `output` — convenience artifact mount; defaults come from the posture manifest.

This file is also the backup manifest: everything that must survive `reset --all` is
either this file or a disk it declares.

### Provider registry — `~/.config/devenv/providers.yaml`

```yaml
providers:
  nixos:  ~/.local/share/devenv/providers/nixos/run
  ubuntu: ~/.local/share/devenv/providers/ubuntu/run
```

Provider identity = filesystem path. Versioning is done by pointing at a different
path. Because old provider binaries outlive protocol revisions, every request carries
`protocolVersion` and providers MUST reject versions they do not understand.

---

## 4. CLI surface

```text
devenv enter <box>        # the only daily-use command; ensure + attach
devenv build <box>        # force provider rebuild of the base image; no attach
devenv reset <box>        # destroy runtime layers (default: all ephemeral layers)
devenv reset --all        # stop everything, wipe cache/machines/*
devenv destroy <box>      # reset + remove persistent disks + revoke credentials
devenv update             # build once, then per-box deep reset (§8)
devenv gc                 # reap unreferenced base images, revoke orphaned tokens
devenv list / status      # enumerate boxes, provider-reported state
devenv creds rotate [box] # rotate issued tokens (old token must die, not just expire)
```

`enter` behavior: load config → resolve box → run provider `ensure` → receive
endpoint → SSH attach with the pinned host key (§7). All complexity hides behind it.

---

## 5. Machine provider protocol

Providers are independent one-shot executables (bash, Go, anything). One verb per
execution. If a provider ever needs to be long-lived (e.g. holding a libvirt
connection), the upgrade path is a socket handshake à la Terraform plugins — not
stdin framing. Do not add framing to v1.

### 5.1 Invocation

```text
<provider-executable> <verb>     # verb as argv[1]; everything else on stdin
```

- **Request:** a single JSON document on stdin. Core writes it, closes its end; the
  provider reads to EOF (pipes block — no polling, no framing; EOF is the frame).
- **Response:** a single JSON document on **stdout**. stdout belongs to the protocol;
  anything human-readable goes to **stderr**. Core forwards stderr live, prefixed per
  box (`[cluster-admin] building image…`), and also captures it to `provider.log`.
- Core persists every request to `cache/machines/<box>/last-request.json` *before*
  invoking, so any invocation is reproducible by hand:
  `provider ensure < last-request.json`.
- Nothing sensitive on argv or env (world-/uid-readable via `/proc`; ARG_MAX). Env
  carries nothing the protocol depends on.

### 5.2 Verbs

| Verb      | Meaning | Notes |
|-----------|---------|-------|
| `ensure`  | Make the box exist and be reachable; respond only when SSH accepts connections | Makefile-like: creates missing base image / overlays / disks / domain as needed |
| `build`   | Force rebuild of the base image for this box's image ref | Does not attach; does not touch other boxes |
| `reset`   | Destroy the runtime layers named in `layers` | `layers` in request body, e.g. `["root"]`; absent = provider's full ephemeral set |
| `destroy` | reset + delete persistent disks + cleanup hooks (token revoke trigger) | First-class so gc/revoke has a home |
| `status`  | Report machine state, base version in use, backing-chain health | Should run `qemu-img info --backing-chain` equivalent checks |

Variation goes in the request body, not in new verbs — every verb added is a case
every future provider must implement.

### 5.3 Request schema (illustrative)

```json
{
  "protocolVersion": 1,
  "verb_args": { "layers": ["root"] },
  "box": "cluster-admin",
  "image": "git+https://gitea.example/me/images?dir=k8s-tooling",
  "paths": {
    "state":  "/home/lj/.local/state/devenv/machines/cluster-admin",
    "cache":  "/home/lj/.local/cache/devenv/machines/cluster-admin",
    "images": "/home/lj/.local/cache/devenv/images/nixos"
  },
  "disks": [
    { "name": "data",    "size": "20G",  "path": ".../state/.../data.qcow2" },
    { "name": "scratch", "size": "100G", "path": ".../state/.../scratch.qcow2" }
  ],
  "mounts": [
    { "name": "creds",  "host": ".../creds-resolved/cluster-admin",
      "guest": "/creds",  "mode": "ro" },
    { "name": "output", "host": ".../cache/.../output",
      "guest": "/output", "mode": "rw" }
  ],
  "ssh_host_key": { "private": ".../state/.../ssh_host_ed25519_key" },
  "overrides": { "kernel_cmdline": null, "nested_virt": false }
}
```

Mount rules:
- `host` and `guest` are both explicit; the transport (virtiofs/9p/bind) is the
  provider's choice.
- **Read-only is enforced on the host side of the boundary** (e.g. virtiofsd
  exporting ro), never merely as a guest mount option. The guest is the adversary;
  its mount table is not a security control.

### 5.4 Response schema

```json
{ "status": "SUCCESS", "endpoint": { "ip": "10.0.0.5", "port": 22 },
  "base_version": "v42", "freshness": { "age_days": 12 } }
```

```json
{ "status": "FAIL", "code": "BASE_IMAGE_MISSING",
  "reason": "no cached image for ref ...; run build" }
```

- `reason` is free text for humans. `code` is from a small enumerated set for things
  the core must *act* on. The core displays reasons and dispatches on codes — it
  never parses prose.
- Initial code set: `OUTDATED`, `BASE_IMAGE_MISSING`, `AUTH_REQUIRED`,
  `UNSUPPORTED_PROTOCOL`, `INTERNAL`.
- Freshness is a **provider judgment** (the core cannot interpret an opaque image
  ref). A stale-but-working box returns `SUCCESS` with a `freshness` hint; the core
  renders the "image is N days old, update? [y/N]" prompt. Default is NO. The core
  never checks image age itself.

### 5.5 Exit-code semantics (authoritative layer)

- **The exit code is authoritative. JSON may only add detail. Any disagreement
  between them is itself a transport-level failure.**
- exit 0 + parseable `SUCCESS` → success.
- exit ≠ 0 + parseable `FAIL` → semantic failure; display `reason`, dispatch `code`.
- exit ≠ 0 + unparseable stdout → transport failure:
  `"provider exited <N>, no parseable output — see provider log"` (include the raw
  exit code; 137 vs 2 is OOM-or-timeout vs argument error).
- exit 0 + `FAIL`, or exit ≠ 0 + `SUCCESS` → fail closed, report the inconsistency.
- Providers are not required to emit JSON on failure — they can't keep that promise
  (segfaults, `set -e`). The protocol is designed for the broken case.

### 5.6 Credential providers

Separate, smaller contract modeled on docker/git credential helpers:

```text
devenv-cred-<name> get  < {"box": "...", "credential": "kubeconfig"}  > {"path": "...", "mode": "ro"}
devenv-cred-<name> rotate / revoke / list
```

Same I/O and exit-code rules as §5.1/§5.5. The default provider is trivial: resolve a
name to a path under `state/creds/`. Token-minting providers (e.g. per-repo Gitea
deploy tokens) implement `rotate`/`revoke`; `devenv gc` reconciles `tokens.yaml`
against the upstream and revokes orphans. **Read-only / minimum-scope tokens by
default; write capability is explicit, per-use opt-in.** A stolen read token leaks
one repo the VM already has; a stolen write token is the supply-chain attack escaping
through your own credentials.

### 5.7 Posture manifest

A static file shipped next to the provider executable (`posture.yaml`), declaring
defaults and ceilings:

```yaml
posture:
  output_mount:    deny        # deny | off | on
  creds_mode:      ro-only     # ro-only | rw-allowed
  kernel_cmdline:  deny        # deny | allow
  nested_virt:     deny
  persistence:     ephemeral-root
```

- The core reads the manifest and **mechanically refuses** box YAML that exceeds it,
  unless the box sets `force: true` — making every posture override explicit,
  visible, and grep-able.
- The core still interprets no *semantics*; it compares requested fields against
  declared ceilings. The provider owns its security envelope the same way it owns
  execution.
- This is how "the tier is the policy" survives a semantics-free core: the untrusted
  tier's manifest denies output mounts and rw creds; the kernel-pet tier's allows
  cmdline parameters and convenience channels.

---

## 6. Disk & image model

### 6.1 Base images

- Built by providers (`build` verb), stored under
  `cache/images/<provider>/<image>/<version>/`, treated **read-only** thereafter.
- The qcow2 backing reference is written with an **absolute path** and explicit
  backing format (`qemu-img create -f qcow2 -b <abs-path> -F qcow2 ...`).
- Libvirt domain XML declares the chain explicitly with `<backingStore>` (terminated
  by an empty `<backingStore/>`), so labeling/locking applies to every chain element,
  the base is enforced read-only while only the leaf is writable, and the dependency
  information is recoverable from `virsh dumpxml` even if tool state is lost.
- Session libvirt (`qemu:///session`) is the default for a single-user tool; system
  libvirt requires ownership/ACL care for files under `$HOME`.

### 6.2 Overlays and persistent disks

- Per-box overlays live in `cache/machines/<box>/` and are disposable by definition.
- Persistent disks live in `state/machines/<box>/`, are created on first `ensure`,
  attached by name, and **never touched by `reset`** — only by `destroy`.
- Overlay size may exceed the base (guest grows into it via
  `boot.growPartition` + `autoResize` or equivalent).

### 6.3 Deletion and gc — there is no `--cascade`

The backing reference lives only in the overlay's header (child → parent); the parent
does not know its children. "Backing file not found" is a *use-time* error, never a
deletion-time one. The single invariant:

> **Never remove a base while any overlay's header still references it.**

- Box deletion is leaf-first by construction:
  `virsh destroy` → `virsh undefine --nvram` → `rm -rf cache/machines/<box>/`
  (plus persistent disks and token revoke on `destroy`).
  Do **not** use `undefine --remove-all-storage` — the tool knows exactly which paths
  it created; explicit removal is safer near shared base files.
- Base deletion is a gc decision:
  1. refcount from tool state (which boxes record this base version),
  2. **verify against reality** by scanning overlay headers
     (`qemu-img info --backing-chain`) before deleting,
  3. delete only unreferenced, non-current versions.
- Escape hatch (persistent-pet tier only): flatten instead of delete —
  `virsh blockpull` live, or `qemu-img rebase -b ""` / `convert` offline — for
  "delete the old cloud image but keep this messy VM."
- `qemu-img rebase -u` across a *rebuilt* backing file is silent corruption
  (block-level, different blocks). Overlays never survive a base version change.

---

## 7. SSH model

- Core generates a host keypair per box at creation, stores it in
  `state/machines/<box>/`, passes the private key path in the request; the provider
  injects it as the VM's host key (NixOS: mount + `services.openssh.hostKeys`;
  Ubuntu: cloud-init — acceptable there because it is the VM's own identity, not a
  host credential; all other secrets stay out of user-data).
- Core attaches with the pinned public key (per-box `known_hosts`,
  `StrictHostKeyChecking=yes`). No TOFU prompts, no `StrictHostKeyChecking=no`, ever.
  Fresh-VM-fresh-key churn disappears because the key is stable per box.
- Forwarded ports bind to host **localhost** only. The forwarded service runs in the
  untrusted VM: don't point a logged-in browser profile at it.

---

## 8. Lifecycle

### 8.1 Idle eviction — bound on the host, politeness in the guest

- **Idle is defined as MY absence** (no live SSH session / recent attach), not "is
  anything running" — long builds must not look idle, and malware must not be able to
  fake "busy" to veto the protective shutdown.
- The authoritative bound is **host-side**: core (or a host timer) tracks attach
  activity — which is host-observable — and `virsh destroy`s after ~30 min idle.
- A guest daemon may additionally do *polite* shutdown (flush, clean poweroff) but is
  killable by guest root and therefore never the security bound.
- Boot-on-demand via `devenv enter` (fzf over `devenv list`).

### 8.2 Update flow

```text
devenv update:
  provider build (once per image ref)        # new base version appears in cache
  per box: reset with the deep layer set     # eager: immediately for all boxes
                                             # deferred: recorded; applied at next reset
```

- v1 strategy: **eager** — simplest, one current base version per image. Versioned
  image directories are kept anyway (cheap; keeps deferred open; makes gc decidable).
- An update-reset discards **all** ephemeral layers (the overlay's backing changed);
  a plain reset discards only what `layers` names. In v1 these coincide (single root
  overlay) — documented behavior: **reset = cold toolchain cache; expect a pull on
  next enter.**
- Staleness prompting is provider-driven via `freshness` (§5.4); never silent
  forced updates, never silent stale use.

---

## 9. v1 scope cuts and roadmap

| Item | Decision | Rationale / trigger to revisit |
|---|---|---|
| Per-box `/nix` overlay disk (warm toolchain across resets) | **Cut from v1** | Protocol already speaks `layers`; v2 adds a well-known `nix` disk name with blessed reset semantics. Revisit when re-pulls actually hurt. |
| Host-side binary cache (harmonia/attic serving the host store, listed in image substituters) | **v1.5 candidate, likely before nix overlay** | Turns every re-pull into a LAN copy and dedupes across all boxes — helps more than the per-box overlay ever did. |
| `waypipe` GUI channel, PDF auto-open mount | Later convenience verbs/mounts | Each is a new channel: re-run the direction-of-flow test when adding. |
| Long-lived provider processes | Only if a provider demonstrably needs it | Upgrade path: socket handshake (Terraform-style), not stdin framing. |

## 10. Build order

1. Protocol fixtures: request/response JSON schemas, exit-code matrix, a `fake`
   provider in bash that exercises every verb and failure mode (the core is testable
   against it before any real provider exists).
2. Core skeleton: config load, posture enforcement, provider exec + stderr tagging +
   `last-request.json`, SSH attach with pinned keys.
3. NixOS provider v1: flake ref → `nixos-generators` qcow2, versioned bases, root
   overlay, labeled persistent disks, host-key injection, `ensure/build/reset/
   destroy/status`.
4. Credential default provider + `tokens.yaml` + `gc` (image reaping + token revoke).
5. Host-side idle eviction.
6. Ubuntu provider (cloud image + checksum/GPG verify + cloud-init identity) behind
   the same contract.
7. Conveniences (output mount auto-open, port forwarding, waypipe) — gated by
   posture, one direction-of-flow review each.

---

### Standing reminders

- Boot-axis hardening (host) and isolation (VMs) are complementary and orthogonal.
- Every new convenience is a channel across the boundary — re-run the
  direction-of-flow test each time. Conveniences are load-bearing: a tool routed
  around protects nothing.
- Keep the directories honest: if `rm -rf ~/.local/cache/devenv` ever becomes unsafe,
  something is filed in the wrong place — fix the filing, not the deletion.