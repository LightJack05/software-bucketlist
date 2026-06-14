# `secdev` — Design Notes

A script collection (not a daemon) for running dev work inside disposable VMs, so that
untrusted code (supply-chain deps, build scripts, LSPs) never executes as my host uid
alongside my browser, Bitwarden, SSH keys, and Wayland clipboard.

> **North star: friction *is* security.** A tool I route around protects nothing. The
> secure path must be the *invisible, lowest-effort* path — faster to reach for than
> running it on the metal. An "invisible" system at 90% coverage beats a "bulletproof"
> one I bypass half the time. Adoption rate is the security property.

---

## 1. Why this exists (threat model)

The host is hardened at the **boot/kernel** axis (Secure Boot, signed UKI, `lockdown=confidentiality`,
IOMMU, ptrace_scope, etc.). None of that defends the axis I actually care about here:
**a process running as my uid (euid 1000) scraping secrets from another process as the same uid.**
In the Unix model the uid *is* the boundary; same-uid processes are mutually trusting.

So the real defense against hostile third-party code is **isolation, not more kernel flags**:
put the untrusted code behind a **hypervisor boundary** (a VM), not just a shared-kernel
container. Containers (even sysbox) share the host kernel — a kernel LPE escapes. A KVM VM
moves the boundary to CPU virt extensions; the attacker needs a hypervisor escape (much rarer).

**The guiding test for every feature/channel:**
> *Which direction does data flow across the boundary, and does anything trusted parse or
> execute it on the far side?*
> - untrusted → trusted, host executes/parses = **dangerous** (this is what we avoid)
> - trusted → untrusted = safe (host hands capability in)
> - untrusted side merely *displays* = low risk

---

## 2. Trust-routing rule (the most important rule)

**Which VM a task goes in is decided by *trust of the code*, not type of work.**

| Code trust | Tier | Posture |
|---|---|---|
| Untrusted (random deps, npm/cargo/pip, unknown repos) | **NixOS** | Ephemeral, full discipline, risky channels OFF by default |
| Trusted-but-kernel-dangerous (my modules, my BPF, cmdline tests) | **Ubuntu pet** | Persistent root OK (I wrote what runs), convenience channels ON |
| Reproduce-the-metal | **Arch trashcan** | My existing reproducible install, disposable |

**The danger to guard against:** laziness routing untrusted code into a weaker-isolation
tier ("it's already booted, root's mutable, I'll just clone it here"). The persistent tiers
have a *weaker* security story — persistence lets an implant persist, mutability makes
tampering non-evident. Keep the line bright: **untrusted code only ever runs in the
hardened ephemeral NixOS tier.** `secdev` must make that the path of least resistance.

---

## 3. The three tiers

### NixOS — reproducible dev (the workhorse)
- For: Go, k8s/operators, kind, normal userspace, anything pulling untrusted deps.
- Built from **my** NixOS config via `nixos-generators` → qcow2, used **read-only** as a
  baseline for all VMs of this tier.
- Ephemeral root (disposable; denies persistence to implants).
- `nix-ld` enabled so foreign dynamically-linked binaries run (the FHS-loader problem).
- LSPs/tooling from **nixpkgs declaratively**, *not* Mason — converts the slowest,
  most-attacker-attractive cache from "mutable trusted executables" into a
  content-addressed, hash-verified derivation. (Mason on NixOS also fights read-only root
  + no FHS loader anyway.)

### Ubuntu — mutable kernel/experiment pet
- For: loading/debugging kernel modules, BPF, **testing weird cmdlines**, getting dirty.
- Persistent root is a *feature* here — get messy, wipe when wrecked.
- **No base image to maintain:** download latest cloud-init server image on creation,
  cache ~1 week. Verify checksum/signature (Canonical `SHA256SUMS` + GPG) — a cached image
  is the base of kernel-test VMs and a swap target if host write access is gained.
  Treat "stale image" as a soft prompt, not silent use (stale = old kernel).

### Arch — the trashcan
- For: "does this happen on my real config or just a weird VM" + last-resort.
- **Zero maintenance:** rides my existing metal install repo. Reinstall = boot Arch ISO,
  `git clone`, `./install.sh`, wait ~2h. Slim package branch, no GUI.
- Disposable on purpose; staleness doesn't matter (not relied on for isolation freshness,
  not running hostile third-party code).

**Maintenance is inverted from intuition:** the *immutable* tier is the high-maintenance
one (I must rebuild the image to get updates). Mitigation: auto-rebuild NixOS image when
older than ~1 week, **async / pre-emptive** (never block `secdev new` on a Nix build),
keep last-known-good and only promote a successful build.

---

## 4. Disk / persistence model (NixOS tier)

Three tiers of persistence on **different disks** — root and toolchain are *different layers*:

1. **Ephemeral root** — image-derived, disposable, trusted. Rebuilt only when image flake changes.
2. **`/data`** — the repo. Writable, untrusted, persisted per-env. (It's the payload, but contained.)
3. **`/cache` — per-env persistent Nix store** — where `nix develop` output lands so the
   expensive (minutes–hours) toolchain resolution happens **once per project**, not per boot.

### Why a persistent Nix store is safe where a Mason dir wasn't
Store paths are **content-addressed** (`/nix/store/<hash>-name`, valid only if contents hash
to `<hash>`). An attacker can't silently *replace* a dep at the same path — they get a new
unreferenced path or a verify failure. Converts "persistent mutable trusted executables"
into "persistent immutable hash-verified cache." Caveats:
- Content-addressing guarantees **integrity, not safety** — a legit-but-hostile dep still
  runs. The VM contains that; the store just doesn't *add* a tampering vector.
- **Store disk is per-env, never shared** — a poisoned input in env A must not land paths
  env B has in scope (same blast-radius logic as per-repo tokens).
- **Store disk is VM-only** — host never mounts it, never `nix-store --import`s from it.
  It contains executables; host Nix must never trust VM-realized paths.

### The mechanism (the genuinely hard 15 lines)
`/nix/store` is *already populated* by the image (the OS lives there). Can't just mount an
empty disk over it. Use **overlayfs**: lower = image store (ro), upper = `/cache` (persistent),
plus workdir on `/cache`.
- **CRUX BUG:** the Nix **database** (`/nix/var/nix/db`) must persist with the upper layer,
  or Nix won't *recognize* the cached paths and rebuilds anyway — with **no visible symptom**
  (VM boots fine, just silently re-downloads).
- Boot-ordering is finicky (store mounts very early); getting it wrong = no boot, or boots
  but silently ignores the cache.
- Don't `nix-store --optimise` an overlaid store (hardlink interactions).
- **Don't** try "point `nix develop` at `/cache/nix/store`" — the store prefix is baked into
  every path's hash and into binaries' RPATHs/interpreter; relocation breaks them. Overlay
  keeps the `/nix/store` prefix correct.

**Prototype this standalone before wiring into `secdev`** (slow, quiet failure modes):
build image → boot with `/cache` overlay → `nix develop` something slow → shut down →
reboot → confirm (a) paths present, (b) `nix store verify` passes, (c) `nix develop` instant.
If it re-downloads, the db didn't persist; if no boot, overlay shadowed the base store.

### NixOS config difficulty (lopsided)
- **Trivial (~40 lines):** sshd + my pubkey, package set, `nix-ld`, user, flakes,
  git-hardening defaults (`core.fsmonitor false`, empty hooksPath). Same mental model as a
  flake `devShell`.
- **Moderate:** `fileSystems.*` declarations for `/data`, `/cache` (declarative, not fstab).
- **Hard, disproportionately:** the overlay store block above. Build trivial+moderate first,
  get a booting SSH-able VM, add overlay **last** as an isolated change against known-good.

---

## 5. Credentials

**Accepted risk:** any in-VM process running as me can read a mounted credential ("knock
knock / who's there / malware / here's the token"). Not worth the complexity of
forward-only injection for my daily case. **Mechanism:** mirror structure
`~/.secdev/<env>/creds`, mounted **read-only** into the VM, populated by the wrapper.

**The mitigation that actually bounds damage is SCOPE, not timing:**
- **Read-only token by default.** A stolen *read* token leaks this one repo's code (the VM
  already has it — ~zero marginal loss). A stolen *write* token lets malware **push a
  poisoned commit upstream** → CI, collaborators, my other machines = the supply-chain
  attack escaping the VM via my own creds, the exact thing the VM exists to prevent.
- **Write capability = explicit, deliberate opt-in**, ideally minted only for the duration
  of an intended push then dropped back to read-only. Never the default.
- Scope each token to the **single repo**, nothing else (or a deploy key per repo).

**Lifecycle:**
- `secdev creds rotate` (per-env or all) — but rotation only helps if the old token **dies**.
- Revoke on teardown. `secdev gc` reconciles the registry against Gitea and revokes orphans
  (deleted envs, uncleanly-died VMs). Otherwise: a growing pile of live tokens +
  `~/.secdev/*/creds` as a host-side secret inventory.

**Persistent-tier caveat:** on Ubuntu/Arch the ro-mount is *not* effectively ephemeral — a
persistent-root VM can **copy** the token to its own disk on first sight. Doesn't change the
risk acceptance, but makes **read-only-by-default + rotation matter most exactly there.**

**cloud-init caveat (Ubuntu):** user-data is a classic place secrets leak into VM
metadata/logs — keep creds out of cloud-init, use the ro-mount.

---

## 6. Handler interface

Modular per-tier handlers behind a standardized contract (keeps it a script collection).
Lean **separate executables** with a documented contract (small JSON-over-stdout for
IP/port/status) over a single Go package — more Unix-y, debuggable, swappable. Collapse to a
package only if duplication/overhead actually bites.

**"Return an IP" is too thin.** First-class verbs:
- `provision/create`, `boot`, `wait-ready` (SSH reachable), `connect-info`,
- `destroy` **with cleanup** — reclaim `/cache`+`/data` disks (NixOS), drop cached-image disk
  (Ubuntu), and **trigger token revoke** (all). If destroy isn't first-class, GC/revoke has
  nowhere clean to live.

**Tiers differ in ways the interface must *expose*, not hide** (else `if image=="ubuntu"`
leaks everywhere): **kernel cmdline as a parameter** (Ubuntu kernel tier needs it; set via
libvirt XML / QEMU `-append` at create time — *not* an image property), **nested-virt flag**,
**persistence model** (does teardown wipe or preserve `/data`?).

---

## 7. Lifecycle (self-managing, daemon-in-VM)

- Lifecycle daemon lives **in the guest** (host stays a thin script collection). VM issues
  its own `poweroff` when idle; boots on demand via `secdev connect <env>` (+ fzf over
  `secdev list`). tmux: second window just SSHes in again → second shell.
- **Define "idle" as MY presence** (SSH sessions from my key / tmux-attach heartbeat),
  **not** "is anything running." Otherwise:
  - long builds / `nix develop` mid-download / working kind clusters look idle → killed mid-task;
  - **malware that wants to survive can fake "busy" to veto the protective auto-poweroff.**
  Idle-eviction bounds malware runtime — don't let malware control it.
- ~30 min idle timeout, boot-on-demand.

---

## 8. GUI / artifact viewing

**The friction trap:** making me `scp` a PDF every LaTeX rebuild is exactly the papercut
that ends in "screw it, build latexmk locally" — and then untrusted TeX packages run on the
metal with my Bitwarden. The convenience must win the common case.

- **PDF / build artifacts → bidirectional `out/` mount, auto-open.** Rebuild → PDF appears
  in host viewer; don't hand me a path, watch the dir and refresh. The PDF is built from
  *my* source by *my* pinned latexmk — marginal risk is low (different, smaller threat than
  random npm). **Defaults to ON for trusted tiers.**
  - Keep the mount **narrow** (a dedicated `out/` drop-box, not a broad rw share that grows
    into "share the whole project rw").
  - For the *genuinely untrusted* case: toggle it OFF, or rasterize on the VM side
    (`pdftoppm` → PNG, near-inert) / open in a sandboxed viewer (not main Firefox profile —
    PDF parser CVEs). **Not** in the daily LaTeX path (that's the over-engineering that gets
    tools abandoned).
- **Interactive GUI (rare) → `waypipe` over SSH**, not VNC/XRDP, not a custom HTML
  compositor. Wayland-native, rides existing SSH. Pocket it for when rasterize-and-drop
  won't do (browser rendering tests, GUI debuggers).

---

## 9. Port forwarding

- SSH `-L` from the same `secdev connect`. `foo.secdev.local:8080` via managed `/etc/hosts`
  entries (least moving parts) or tiny dnsmasq / mDNS.
- **Bind forwarded ports to host localhost**, not `0.0.0.0` (don't expose the untrusted VM's
  service to the LAN).
- The forwarded service runs in the **untrusted VM** — host browser pointing at it is
  host-touches-untrusted. Fine for my own web app; don't feed it host cookies / main
  logged-in profile.

---

## 10. Per-tier defaults (so I almost never pass flags)

The secure choice and the convenient choice only conflict if "secure" is one global posture
I fight. It's **per-env** — declare the env's nature once (or infer from image/handler), and
each tier gets the posture that fits. **The tier *is* the policy.**

| | NixOS (untrusted) | Ubuntu pet (trusted/kernel) | Arch (trashcan) |
|---|---|---|---|
| Root | ephemeral | persistent | persistent |
| `/cache` overlay store | yes | n/a | n/a |
| bidi `out/` mount + auto-open | **off** by default | **on** | on |
| host browser → fwd ports | discouraged | ok | ok |
| token default | read-only | read-only | read-only |
| nested virt | off | opt-in | opt-in |
| cmdline-as-param | rarely | **yes** | maybe |

> Make the secure default the path of least resistance: `secdev new` + fuzzy-clone should be
> **fewer keystrokes** than `git clone && cd && npm install` on the host. Laziness pointed at
> the secure option is the most durable control there is. Win the 10 seconds, win the war.

---

## Build order (suggested)

1. (Already queued behind it: the eBPF LSM tripwire — separate project, garnish not peer.)
2. NixOS image tiers 1+2 (booting, SSH-able, `nix-ld`, `/data`) — an afternoon.
3. `secdev` lifecycle skeleton (create/connect/list/destroy, fzf, idle daemon-in-VM) — my wheelhouse.
4. Overlay `/cache` store — **prototype standalone first**, then wire in. The hard part.
5. Credential mount + scope + `rotate`/`gc`.
6. Ubuntu + Arch handlers behind the same interface.
7. PDF auto-open mount; `waypipe` + port-forward conveniences.

---

### Standing reminders
- Boot-axis hardening (host) and isolation (VMs) are **complementary, orthogonal** — neither
  substitutes for the other.
- The eBPF LSM is a **tripwire**, not a wall — log denials loudly; the alarm is the product,
  prevention is a bonus. Verify it even loads under `lockdown=confidentiality` first.
- Every new convenience = a channel across the boundary. Re-run the direction-of-flow test
  each time. Conveniences aren't concessions — they're load-bearing, because a tool I route
  around protects nothing.