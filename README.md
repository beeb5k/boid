# Noctalia XBPS repository

This repository automatically builds and publishes an **independent XBPS repository** for Noctalia from official upstream release archives.

## Scope

- Package provided: `noctalia`
- Target: **Void Linux x86_64 (glibc)**
- Public repository URL: **https://beeb5k.github.io/boid/repo/**

Only the current Noctalia binary and required XBPS repository metadata/signatures are published.

## Client trust setup (safe XBPS public-key flow)

Packages and repository metadata are signed with `xbps-rindex` using the private key stored in GitHub Actions secret `XBPS_PRIVATE_KEY_B64`.

- The private key must never be committed or shared.
- Clients trust only the matching public key.

### 1) Derive public key from your private key (maintainer side)

Use the same private key that you base64-encode into `XBPS_PRIVATE_KEY_B64`:

```sh
openssl pkey -in private.pem -pubout > noctalia-repo.plist
```

Distribute `noctalia-repo.plist` over a trusted channel you control (for example, attach it to a signed release in this repository).

### 2) Install trust key on Void client

```sh
sudo install -Dm0644 noctalia-repo.plist /var/db/xbps/keys/noctalia-repo.plist
```

Do **not** disable XBPS signature verification.

## Add this repository without replacing official Void repositories

Create a separate file under `/etc/xbps.d/`:

```sh
sudo tee /etc/xbps.d/20-noctalia-repo.conf >/dev/null <<'EOF'
repository=https://repo-default.voidlinux.org/current
repository=https://repo-default.voidlinux.org/current/nonfree
repository=https://repo-default.voidlinux.org/current/multilib
repository=https://repo-default.voidlinux.org/current/multilib/nonfree
repository=https://beeb5k.github.io/boid/repo/
EOF
```

If your system does not use multilib, remove the multilib lines.

## Synchronize and install/update Noctalia

```sh
sudo xbps-install -S
sudo xbps-install -u
sudo xbps-install noctalia
```

## Workflow behavior

Workflow file: `.github/workflows/publish.yml`

- Triggers:
  - every 12 hours (`schedule`)
  - manual (`workflow_dispatch`)
- Release detection:
  - reads `https://api.github.com/repos/noctalia-dev/noctalia/releases/latest`
  - downloads the exact tag archive
  - computes SHA-256 from that archive
  - updates a **temporary** template copy for build
- Build environment:
  - runs in `voidlinux/voidlinux:latest`
  - uses `xbps-src` with `XBPS_CHROOT_CMD=ethereal` for disposable container builds
  - targets `x86_64` glibc
- Publication state:
  - scheduled runs skip only when `.github/noctalia-last-success.env` matches latest upstream version/checksum
  - manual runs always rebuild current upstream release
  - state and template are updated **only after successful Pages deployment**
- Safety:
  - build/validation/signing happen before deployment
  - failed runs do not deploy and do not mark release as published
  - workflow concurrency prevents schedule/manual races

## Required repository configuration

1. GitHub Actions secret:
   - Name: `XBPS_PRIVATE_KEY_B64`
   - Value: base64-encoded PEM RSA private key used for `xbps-rindex` signing

2. GitHub Pages setting (manual, one-time):
   - **Settings -> Pages -> Build and deployment -> Source -> GitHub Actions**

`has_pages: false` repositories must enable this setting before deployment can succeed.

## Current support status

This repository currently supports **x86_64 glibc only**.
