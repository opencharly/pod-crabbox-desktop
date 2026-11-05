# pod-crabbox-desktop

The **desktop lease image** for the Crabbox `containerd` provider — the base
[`pod-crabbox`](https://github.com/opencharly/pod-crabbox) lease contract with
Xfce on TigerVNC and a noVNC web front end, plus the Wayland stack.

## What it provides

Every lease contract property of the base image is inherited unchanged:

| Property | Value |
|---|---|
| Base | `pod-crabbox` (CachyOS), pinned at `v2026.307.1828` |
| SSH | `sshd` on **2222**, public-key only, user `crabbox` |
| Sudo | `crabbox` has passwordless sudo |
| Inputs | read-only `/crabbox`: `authorized_keys`, `lease.env`, `bootstrap.sh` |
| Readiness | `/usr/local/libexec/crabbox/ready` — polled over SSH until exit 0 |
| PID 1 | `/usr/local/bin/crabbox-lease-init` → `exec sshd -D` |

This variant adds the desktop surface:

| Property | Value |
|---|---|
| Desktop | Xfce (`xfce4-session`, `xfwm4`, `xfce4-panel`, `xfdesktop`, `xfce4-settings`, `xfce4-terminal`, `xfce4-appfinder`, `thunar`) |
| Session bus | `dbus` (Arch's package; the plan row's `dbus-x11`) |
| VNC | `tigervnc` — `Xvnc` + `vncserver` |
| noVNC | pinned upstream releases: noVNC 1.6.0 + websockify 0.13.0 |
| Wayland alternative | `labwc` + `wayvnc` |
| VNC bind | `127.0.0.1:5901` (display `:1`) |
| noVNC bind | `127.0.0.1:6080` |
| Starter | `/usr/local/bin/crabbox-start-desktop` |

## How to use it

The desktop is a **per-lease session started over SSH**, never the image
entrypoint: sshd stays PID 1 and the readiness hook still runs first.

```sh
ssh crabbox@<lease>
crabbox-start-desktop
# then, from your workstation:
ssh -L 6080:127.0.0.1:6080 crabbox@<lease>
# open http://127.0.0.1:6080/vnc.html
```

The wrapper starts `vncserver :1` (Xfce, 1920x1080x24, loopback only) and
`websockify` serving noVNC's static web root on `127.0.0.1:6080`. Neither port
is published; the SSH tunnel is how a consumer reaches them.

VNC authentication is disabled by design: the server binds loopback only, so
the lease's SSH session is the boundary. `vncserver` refuses to start without a
password file otherwise.

## Package sources

`tigervnc`, the Xfce components, `dbus`, `labwc`, `wayvnc` and the X utilities
are Arch/CachyOS repository packages (`pacman -Si`, 2026-11-05). noVNC and
websockify have **no Arch/CachyOS package**, so they are installed from their
pinned upstream releases with charly's `download:` verb (noVNC 1.6.0,
websockify 0.13.0 — the same source URLs the AUR PKGBUILDs use). websockify's
core proxy needs only the Python standard library, which the base image ships.

## Layout

- `charly.yml` — the `pod-crabbox-desktop` image (pinned base, `pac` build; noVNC/websockify from their pinned upstream releases with `download:`).
- `candy/desktop-vnc/` — packages, the TigerVNC config, the starter, and the
  `check:` steps.
- `.github/workflows/tag-on-merge.yml` — CalVer tag + `CHANGELOG/` on merge.
- `.github/workflows/image.yml` — builds on `pull_request` and publishes at the
  merge tag, so the image tag equals the git tag.
- `CHANGELOG/` — per-CalVer history.

## Related

- [`opencharly/pod-crabbox`](https://github.com/opencharly/pod-crabbox) — the lease
  base image this one imports.
- [`CONTAINERD-IMAGES.md`](https://github.com/opencharly/opencharly/blob/main/CONTAINERD-IMAGES.md)
  §2 — the lease-image contract and this image's row.
