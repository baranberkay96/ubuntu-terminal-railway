# Ubuntu 26.04 Web Terminal (Browser Shell): Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Ubuntu 26.04 Web Terminal (Browser Shell)** ("26.04" is the pinned LTS release) |
| Description | Persistent Ubuntu 26.04 LTS shell in your browser, password-protected. (70 chars) |
| Category | Other |
| Tags | ubuntu, terminal, web-terminal, linux, shell (the first five are sent), ttyd, tmux, ubuntu-26.04, devbox |
| Icon | `https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/ubuntu-linux.svg` (200, image/svg+xml; also used by 107) |

Why: Railway's Ubuntu niche had about 9.5k recent deploys spread across about 55 templates ("Ubuntu 26.04" about 635, "Ubuntu 26.04 (Web Terminal)" about 206). This package brings the design of our Debian 13 and Kali Linux web terminals (60, 109) to Ubuntu 26.04 LTS.

## Base

Derived from `60-debian-terminal` (toolset, Node.js from nodejs.org, `~/.npm-global`) and `109-kali-terminal` (distro ttyd package, build-time extra packages, banner without host figures). Same auth proxy, healthcheck and tmux design; see `60-debian-terminal/template/TEMPLATE.md` for the full rationale of the nginx + ttyd auth-header setup.

Ubuntu-specific changes:
- The `ubuntu` image already has the user `ubuntu` (uid 1000). The Dockerfile adds it to `sudo` and gives it bash instead of creating a user. Its home is created on the volume at first boot.
- ttyd 1.7.7 comes from Ubuntu universe (`1.7.7-4build1`), so it follows Ubuntu's security updates (libwebsockets) instead of a static binary.
- `UBUNTU_EXTRA_PACKAGES` (build argument, like Kali's `KALI_EXTRA_PACKAGES`) replaces Debian's runtime `APT_PACKAGES` restore: packages are baked into the image and cost no RAM or time at boot.
- No man pages: Ubuntu's container image is minimized.
- The fastfetch banner leaves out CPU, memory, disk and uptime (they would show the Railway host).
- File capabilities are stripped at build (as in 109): Ubuntu's `ping` carries `cap_net_raw`, which Railway's containers lack, so it failed with "Operation not permitted" even under sudo.

## Pinned versions (checked 2026-11-14)

| Component | Source | Version |
|---|---|---|
| Base image | `ubuntu:resolute-20261105.1@sha256:66b8084a0d45bd4877992bd38e04b51bfa99bd57293364cb6391fb674771846e` (newest dated `26.04` tag) | Ubuntu 26.04 LTS |
| ttyd | Ubuntu universe | 1.7.7-4build1 |
| Node.js | nodejs.org tarball, SHA-256 checked | 24.22.3 LTS |
| nginx | Ubuntu security | 1.28.3-2ubuntu1.12 |
| tmux | Ubuntu security | 3.6a-2ubuntu0.1 |
| Python | Ubuntu updates | 3.14.4 (seen in the first build) |

## Services

### Terminal

| Setting | Value |
|---|---|
| Source | repo `baranberkay96/ubuntu-terminal-railway`, root `/services/terminal` (Dockerfile) |
| Healthcheck | `/healthz` on `PORT=8080`, timeout 120 s |
| Public domain | yes, target port 8080 |
| Volume | `/home` (home of `ubuntu` is `/home/ubuntu`) |
| Restart policy | ON_FAILURE, 10 retries |
| Replicas | 1 |

### Variables

| Variable | Default | Notes |
|---|---|---|
| `PASSWORD` | `${{secret(32)}}` | 12-256 bytes, one line. Otherwise the login is locked with a random password; the deploy still succeeds and the log says how to fix it. |
| `USERNAME` | `ubuntu` | Invalid names fall back to `ubuntu` with a warning. |
| `PORT` | `8080` | Invalid values or 7681 fall back to 8080 with a warning. |
| `TERMINAL_TMUX` | `true` | `false`: plain login shell per tab. |
| `TERMINAL_FONT_SIZE` | `15` | 8-40; anything else falls back to 15 with a warning. |
| `UBUNTU_EXTRA_PACKAGES` | empty | Build argument. Invalid names and packages apt cannot install are skipped with a warning (build log and deploy log); never fails the build. |
| `TZ` | `UTC` | Shell time zone. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `15` | Grace period for the clean tmux shutdown. |

No required inputs. STANDARD rule 29: no value typed into the deploy form fails the deploy.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month.

| Service | Idle (measured) | Typical | Notes |
|---|---|---|---|
| Terminal | 0.04 GB RAM (measured), ~0.002 vCPU | 0.12 GB after `npm install` (measured), 0.3 GB avg | Peaks come from what users run; a `pip --no-binary :all:` source build hit the 0.5 GB cap and was OOM-killed while the terminal stayed up |
| Volume `/home` | 0.1 GB | 2-5 GB | Projects, venvs, node_modules |

Estimated monthly cost: idle about $0.50, typical about $4.50. Minimum plan: Free (0.5 GB RAM per service) runs the shell; heavy builds need more. Image: about 42 MB base plus about 150 MB of packages and about 30 MB of Node.js, compressed, far below the 4 GB Trial/Free image cap.

## Verification checklist

- [ ] Build completes; the log shows `ttyd version 1.7.7`, `v24.22.3` and `sha256sum: OK` for Node.
- [ ] `/healthz` 200; `/` 401 without a password, 401 with a wrong one, 200 with `PASSWORD`.
- [ ] The terminal WebSocket (`/ws`, subprotocol `tty`) opens with the password and a command's output comes back.
- [ ] `whoami` = `ubuntu`, `sudo -n true` works, `node -v`, `python3 -V`, `gcc --version`.
- [ ] A file in `/home/ubuntu` survives a redeploy.
- [x] RAM stays under 0.5 GB with a shell open (Free plan): 40 MB idle, 115 MB after npm install, with `serviceInstanceLimitsUpdate(memoryGB: 0.5)` on 2026-11-14.
- [ ] `PASSWORD=abc`: the deploy succeeds, the log shows the lock warning, every password gets 401.

## Open questions

- Railway's edge `Host`/`Origin` handling is the same as for 60 and 109, which work in production.
