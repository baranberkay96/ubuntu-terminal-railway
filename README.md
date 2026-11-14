# Deploy and Host Ubuntu 26.04 Web Terminal with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/ubuntu-2604-web-terminal-browser-shell?referralCode=20H68q)

An Ubuntu 26.04 LTS (Resolute Raccoon) shell that opens in any browser. You work as the user `ubuntu` with sudo, inside tmux, so sessions and long-running jobs survive closed tabs. A generated password protects the login, and your home directory sits on a persistent volume. Git, Python, Node.js 24, build tools and everyday CLI utilities come preinstalled.

## About Hosting Ubuntu 26.04 Web Terminal

A web terminal needs a long-running Linux container, a terminal server, real authentication in front of it, TLS, and storage that outlives rebuilds. This community template builds on the official `ubuntu` image from Docker Hub, pinned by dated tag and digest, and serves the shell with ttyd. nginx stands in front of ttyd and checks a password that Railway generates at deploy, so the login never shows up in logs. `/home` is a Railway volume, Railway's edge provides HTTPS, and the container restarts automatically if it stops.

## Common Use Cases

- A cloud Ubuntu shell for Chromebooks, tablets, locked-down work laptops or any machine without SSH
- Running long jobs, scripts and builds in tmux and checking back on them later
- Learning Linux and the command line on a disposable Ubuntu 26.04 LTS machine
- Testing software and packages against Ubuntu 26.04 LTS before rolling it out elsewhere
- A small always-on workspace for Git, Python and Node.js projects

## Dependencies for Ubuntu 26.04 Web Terminal Hosting

- A Railway volume mounted at `/home` (included in the template)
- Nothing else: no database, no API keys, no external accounts

### Deployment Dependencies

- Ubuntu 26.04 LTS release notes: https://documentation.ubuntu.com/release-notes/26.04/
- Official Ubuntu image on Docker Hub: https://hub.docker.com/_/ubuntu
- ttyd (web terminal, MIT): https://github.com/tsl0922/ttyd
- tmux: https://github.com/tmux/tmux/wiki
- Node.js releases: https://nodejs.org/en/about/previous-releases
- Railway build variables: https://docs.railway.com/builds/dockerfiles#using-variables-at-build-time
- Railway Fair Use Policy: https://railway.com/legal/fair-use

### Implementation Details

| Service | Source | Purpose |
|---|---|---|
| Terminal | Dockerfile based on `ubuntu:resolute-20261105.1` | nginx basic-auth proxy on port 8080 (public HTTPS domain, `/healthz`), ttyd on loopback, tmux sessions, user `ubuntu` with sudo, volume at `/home` |

**Plan:** works on every plan, including Free (0.5 GB RAM per service). Measured on Railway with a 0.5 GB memory limit: about 0.04 GB of RAM idle with an open tmux session, and about 0.12 GB after an `npm install` of Express and TypeScript. Large source builds (for example compiling Python packages with `pip install --no-binary`) can need more than 0.5 GB. On Free the kernel then stops that build, not the terminal; on Hobby and Pro the service can use more memory.

**First login**

1. Open the service's public URL.
2. Your browser asks for a user name and password. The user name is `ubuntu` (the `USERNAME` variable). The password is the generated `PASSWORD` value on the service's Variables tab.
3. You land in a tmux session named `main`. Close the tab at any time and reopen the URL to resume. For a second, independent session, open `https://{your-domain}/?arg=work`.
4. `sudo -i` gives you a root shell.

**Tools included**: git, git-lfs, curl, wget, vim, nano, tmux, htop, ncdu, tree, jq, ripgrep, fd (`fdfind`), rsync, sqlite3, zip/unzip, dig, ping, netcat, openssh-client, build-essential (gcc, g++, make), Python 3.14 with pip, venv and pipx, and Node.js 24 LTS with npm. `npm install -g` installs into `~/.npm-global`, so it works without sudo and persists.

**More packages**: set the `UBUNTU_EXTRA_PACKAGES` variable to a space-separated list of Ubuntu packages, for example `golang-go postgresql-client`. Railway passes the variable to the Docker build, so the next deploy builds an image with those packages and they survive every redeploy. Names apt cannot install are skipped with a warning in the build and deploy logs, so a typo never fails the deploy. Packages you install with `sudo apt install` in the shell work right away but reset on the next redeploy. The container image is minimized, so man pages are not installed.

**What persists**: everything under `/home/ubuntu`: files, Git repositories, shell history, dotfiles, Python virtualenvs and pipx apps in `~/.local`, global npm packages in `~/.npm-global`. Running processes and packages installed with apt reset on every redeploy or restart. tmux keeps sessions alive across browser reconnects, not across container restarts.

**Security**: the shell is never served without a password. With an empty, short (under 12 characters) or overlong `PASSWORD` the service still starts, but the login is locked with a random password nobody knows and the deploy log says how to fix it. Invalid `PORT`, `USERNAME` or `TERMINAL_FONT_SIZE` values fall back to the defaults with a warning in the log. nginx checks the password, which the container keeps only as a SHA-512 crypt hash, and rate-limits attempts. It also rejects WebSocket connections from other sites. ttyd listens on 127.0.0.1 only. The password does not appear in logs, process arguments or the shell environment. To change it, edit `PASSWORD` and redeploy. Browsers cache basic-auth logins until you close them, so use a private window on shared computers.

**Limits**: this is a container, not a VM. There is no systemd, no Docker inside the shell, no raw sockets (`ping` works with `sudo`), and only HTTP(S) on the public domain. Railway's [Fair Use Policy](https://railway.com/legal/fair-use) applies to everything that runs in this shell.

**Scaling**: this is a single-user machine on one volume, so keep one replica. Railway scales RAM and CPU vertically up to your plan limit. For a team, deploy one terminal per person.

**Pinning and upgrades**: the Ubuntu base image is pinned by dated tag and digest on the `FROM` line of `services/terminal/Dockerfile`, and Node.js by version and SHA-256 checksum. Every fresh build runs `apt-get upgrade`, so security fixes released after the base image are included. To move to a newer base, change the tag and digest and redeploy.

### Why Deploy Ubuntu 26.04 Web Terminal on Railway?

Railway gives the terminal an HTTPS URL, a persistent volume and automatic restarts, with no server to administer. You get a protected Ubuntu 26.04 LTS shell from one click and pay only for the CPU and memory it uses. This is a community template, not affiliated with Canonical, the ttyd author or Railway. Ubuntu is a registered trademark of Canonical Ltd.
