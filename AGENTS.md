# AGENTS.md — pod-crabbox-desktop

The **desktop lease image** for the Crabbox `containerd` provider: the
`pod-crabbox-desktop` image (`charly.yml`) and the `desktop-vnc` candy
(`candy/desktop-vnc/`) it composes on top of `opencharly/pod-crabbox`. This repo
owns exactly the desktop delta — Xfce, TigerVNC, noVNC/websockify, the Wayland
stack and the starter. The lease contract belongs to the base repo; a change to
it lands there, not here.

Canonical files:

- `charly.yml` — the image: `base: crabbox.pod-crabbox` at a pinned tag,
  `distro: [cachyos, arch]`, `build: [pac]`, `user: root`.
- `candy/desktop-vnc/charly.yml` — the packages, the TigerVNC config, the
  starter and the `check:` steps.
- `candy/desktop-vnc/crabbox-start-desktop` — the wrapper the consumer runs over
  SSH.
- `.github/workflows/image.yml` — the image build; it triggers on
  `tag-on-merge` **completion** (`workflow_run`), never on `push: tags`
  (`opencharly/.github#277`: tag-on-merge mints tags through the REST API, which
  emits no push event).
- `.github/workflows/tag-on-merge.yml` — the org's CalVer tag + `CHANGELOG/`
  dispatcher, installed by the org bootstrap.
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-image:image` — the `candy:` image entry (base, user, build, builder).
- `/charly-image:layer` — the candy body: `distro:` packages, plan step verbs,
  and the `check:` surface.
- `/charly-distros:cachyos` — the CachyOS base, the `arch-builder`, and Arch
  package semantics.
- `/charly-build:build` — the image build and the pinned-release `download:` path.
- `/charly-check:check` — the check verbs, the ADE requirement, and the R10 gate.
- `/charly-internals:git-workflow` — before any git/PR action.

## What is unique here

- **The image tag IS the git tag.** `.github/workflows/image.yml` passes the
  newest `v*` tag (minus its `v`) to `charly box build --tag`, which uses a
  supplied tag verbatim. `defaults.tag: auto` is only the LOCAL-build fallback;
  the CI caller always supplies the tag.
- **The base is a pinned import, never copied.** `import: crabbox:
  @github.com/opencharly/pod-crabbox:v2026.307.1828` with `base:
  crabbox.pod-crabbox` builds the base when needed; `--push` publishes only the
  requested box (opencharly/plugin-build#49).
- **noVNC and websockify are pinned upstream releases, not AUR packages.**
  `pacman -Ss novnc` and `pacman -Ss websockify` are empty (2026-11-05). They are
  fetched with `download:` from the exact source URLs the AUR PKGBUILDs use
  (`noVNC v1.6.0`, `websockify v0.13.0`) and asserted by version-bearing checks.
  The AUR builder route was measured and does not work from a variant repo: the
  emitted `FROM ghcr.io/opencharly/arch-builder:<this-build-tag>` is not
  published and the imported builder is deliberately not pushed
  (opencharly/charly#982). Do not add `build: [aur]` back without fixing
  that builder-resolution path; never substitute a guessed package name.
- **TigerVNC's `vncserver` takes ONLY a display number.** `vncserver :1 -geometry
  …` exits with `usage: vncserver <display>`. Everything else lives in
  `/etc/tigervnc/vncserver-config-defaults`; the candy writes it.
- **VNC auth is deliberately off.** The server binds loopback only; the lease's
  SSH session is the boundary. `SecurityTypes=None` is also what lets
  `vncserver` start without a password file.
- **The starter never replaces the entrypoint.** The desktop is a per-lease
  session started over SSH; sshd must stay PID 1 and the readiness hook must run
  first. Do not add an `entrypoint:` that runs Xfce.

## Build / validate / test

- `charly box validate` at the repo root — the structural check on the image and
  its candy.
- `charly box generate pod-crabbox-desktop` — inspect the emitted Containerfile
  and the baked `ai.opencharly.description` plan.
- `charly box build pod-crabbox-desktop` — the local image build (`pac` packages
  plus the pinned-release noVNC/websockify `download:` steps).
- `charly check box pod-crabbox-desktop` — the build-scope `check:` steps.
- The PR image build (`.github/workflows/image.yml`, `pull_request` path) builds
  the image and publishes nothing; the merge-time `workflow_run` path publishes
  `ghcr.io/opencharly/pod-crabbox-desktop:<git-tag>`.

## Modify this repo

- A package may be added only from a named source (the base candy, an existing
  `layer-*`/`pod-*` repo's Arch section, the provider's docs, the Arch package
  index, or the AUR RPC). Record the source in a comment.
- Every claim in the plan needs a deterministic `check:`. A package that merely
  installs without a probe is not proof.
- `charly check box` runs only build-context `check:` steps. The VNC/noVNC
  **ports** are proven at build time as the loopback configuration the wrapper
  hands to `vncserver`/`websockify`; a claim that they are listening needs a
  deploy-context bed, not this gate.

## Landing

- PR-only. Every change lands through a pull request; the org-required
  `charly/pr-validator` validates the diff and body and arms native auto-merge on
  PASS. Direct pushes to `main` are blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR
  body IS the changelog.
- The authoritative rulebook is the umbrella `AGENTS.md` in
  `opencharly/opencharly`. Do not restate its rules here.
