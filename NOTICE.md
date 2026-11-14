# NOTICE: Ubuntu 26.04 Web Terminal template

## Software installed in the image

| Component | Source | License | How it is used |
|---|---|---|---|
| Ubuntu 26.04 LTS (Resolute Raccoon) | `ubuntu` official image, https://hub.docker.com/_/ubuntu, tag `resolute-20261105.1`, index digest `sha256:66b8084a…` (2026-11-12) | Various free licenses (see `/usr/share/doc/*/copyright` in the image) | Base image (`FROM`) and apt packages |
| ttyd 1.7.7 | Ubuntu package `ttyd` (1.7.7-4build1, universe) | MIT, Copyright (c) 2016-2022 Shuanglei Tao | Web terminal on loopback, installed unmodified |
| nginx 1.28 | Ubuntu package `nginx` | BSD-2-Clause | Basic-auth reverse proxy. The config was written for this template |
| tmux 3.6a | Ubuntu package `tmux` | ISC | Persistent sessions |
| tini 0.19.0 | Ubuntu package `tini` | MIT | Init and child subreaper |
| Node.js 24.22.3 LTS | Official tarballs from https://nodejs.org/dist/v24.22.3/, SHA-256 checked | MIT (bundled components under their own licenses, see `/usr/local/share/doc/nodejs/LICENSE`) | Installed unmodified into `/usr/local` |
| git, Python 3.14, build-essential, ripgrep, fd-find, fastfetch and the other CLI tools | Ubuntu packages | Each package's own license (GPL, MIT, BSD, Apache and similar) | Installed unmodified from the Ubuntu archive |

Changes to installed Ubuntu files: the image's existing `ubuntu` account (uid 1000) is added to the `sudo` group and gets bash as its shell, `/etc/sudoers.d/90-ubuntu-terminal` grants it passwordless sudo, nginx's default site is removed, and `/etc/skel` gets one line in `.bashrc` that loads this template's shell defaults plus an empty `.sudo_as_admin_successful` that silences Ubuntu's sudo hint. No package files are edited.

Optional packages named in `UBUNTU_EXTRA_PACKAGES` are installed from the Ubuntu archive at build time under their own licenses.

Product names belong to their owners. This is a community template, not affiliated with Canonical, the ttyd author, the OpenJS Foundation or Railway. Ubuntu is a registered trademark of Canonical Ltd.

## Upstream files and data checked (2026-11-14)

- Docker Hub tags of `library/ubuntu`: `resolute-20261105.1` (index `sha256:66b8084a0d45bd4877992bd38e04b51bfa99bd57293364cb6391fb674771846e`, amd64 manifest `sha256:0d5255be…`, about 42 MB compressed), the newest dated tag of `26.04`/`resolute` on 2026-11-14.
- Ubuntu `resolute`, `resolute-updates` and `resolute-security` `Packages` indexes (main and universe, amd64): every package name and version above, and the installed size of the package set (about 575 MiB, about 151 MiB of `.deb` files).
- https://nodejs.org/dist/index.json (24.22.3 is the newest 24.x "Krypton" LTS, 2026-11-10) and its `SHASUMS256.txt`.
- The ttyd behaviour this template relies on (auth-header mode `-H`, loopback `-i lo`, the `-c` startup log line that prints the credential) was checked for the Debian Web Terminal template of the same author against ttyd tag `1.7.7`; Ubuntu ships the same release.

## Market templates reviewed

Railway's marketplace lists about 55 Ubuntu templates with about 9.5k recent deploys between them, among them "Ubuntu 26.04" and "Ubuntu 26.04 (Web Terminal)" (figures from the marketplace API, 2026-11). Only these listing names and deploy counts were used, to size the niche. No files or text from other templates were used.

All files in this package were written for this template, adapted from the same author's Debian 13 and Kali Linux Web Terminal templates: the Dockerfile, `entrypoint.sh`, `terminal-session.sh`, `bashrc.sh`, `tmux.conf`, `nginx.conf.template`, `motd.txt` and the docs.
