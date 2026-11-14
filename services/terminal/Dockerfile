# Ubuntu 26.04 LTS Web Terminal for Railway.
#
# Ubuntu 26.04 LTS (Resolute Raccoon) with ttyd serving tmux sessions, behind
# an nginx basic-auth proxy. The shell runs as the non-root user "ubuntu"
# (passwordless sudo), and /home is a Railway volume. The base image is pinned
# by dated tag and digest, the other versions below: change them and redeploy
# to upgrade.
FROM ubuntu:resolute-20261105.1@sha256:66b8084a0d45bd4877992bd38e04b51bfa99bd57293364cb6391fb674771846e

# Node.js 24 LTS "Krypton" official tarballs. The checksums come from
# https://nodejs.org/dist/v24.22.3/SHASUMS256.txt.
ARG NODE_VERSION=24.22.3
ARG NODE_SHA256_AMD64=4f2b175a6d53a5c617a80dba2676238a0688cef28c6da4eba35c7d844e66fcfb
ARG NODE_SHA256_ARM64=6e51a2ada00f0011cc266f65d98922b403d7ad88f809ca2aa615a5fe1ce6e86c

# Extra Ubuntu packages to bake into the image, space separated, for example
# "golang-go postgresql-client". Railway passes the service variable of the
# same name as a build argument, so changing it rebuilds the image with those
# packages. They survive redeploys, unlike `apt install` in the shell.
ARG UBUNTU_EXTRA_PACKAGES=""

ARG DEBIAN_FRONTEND=noninteractive

ENV LANG=C.UTF-8 \
    TZ=UTC

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# System packages: the auth proxy (nginx), the web terminal (ttyd 1.7.7 from
# Ubuntu universe), the init (tini), tmux and the tools people expect in a dev
# shell. `apt-get upgrade` picks up security fixes released after the base
# image was built. Ubuntu's container image is minimized (no man pages).
RUN apt-get update \
 && apt-get upgrade -y \
 && apt-get install -y --no-install-recommends \
      bash-completion bind9-dnsutils build-essential bzip2 ca-certificates curl \
      fastfetch fd-find file git git-lfs gnupg htop iproute2 iputils-ping jq less \
      libcap2-bin lsof nano ncdu ncurses-term netcat-openbsd nginx openssh-client openssl \
      pipx procps python3 python3-dev python3-pip python3-venv ripgrep rsync \
      sqlite3 sudo tini tmux tree ttyd tzdata unzip vim wget xz-utils zip \
 && rm -f /etc/nginx/sites-enabled/default \
 && rm -rf /var/lib/apt/lists/* \
 && ttyd --version

# Optional extras from UBUNTU_EXTRA_PACKAGES, checked as package names first
# so the variable cannot inject shell or apt options. A value typed into the
# deploy form must not fail the build: invalid names and packages that apt
# cannot install are skipped with a warning (here and in the deploy log, from
# /etc/ubuntu-terminal/skipped-packages), and the rest is installed.
RUN set -eux; \
    install -d -m 0755 /etc/ubuntu-terminal; \
    : > /etc/ubuntu-terminal/skipped-packages; \
    read -r -a requested <<< "${UBUNTU_EXTRA_PACKAGES//,/ }"; \
    extra=(); \
    for pkg in "${requested[@]}"; do \
      if [[ "$pkg" =~ ^[a-z0-9][a-z0-9+.-]+$ ]]; then \
        extra+=("$pkg"); \
      else \
        echo "WARNING: UBUNTU_EXTRA_PACKAGES: skipping invalid package name '${pkg:0:64}'" >&2; \
        printf '%s\n' "${pkg:0:64}" >> /etc/ubuntu-terminal/skipped-packages; \
      fi; \
    done; \
    if (( ${#extra[@]} > 0 )); then \
      apt-get update; \
      if ! apt-get install -y --no-install-recommends "${extra[@]}"; then \
        echo "WARNING: UBUNTU_EXTRA_PACKAGES: installing '${extra[*]}' together failed. Trying one package at a time." >&2; \
        for pkg in "${extra[@]}"; do \
          if ! apt-get install -y --no-install-recommends "$pkg"; then \
            echo "WARNING: UBUNTU_EXTRA_PACKAGES: skipping '${pkg}', apt could not install it." >&2; \
            printf '%s\n' "$pkg" >> /etc/ubuntu-terminal/skipped-packages; \
          fi; \
        done; \
      fi; \
      rm -rf /var/lib/apt/lists/*; \
    fi; \
    chmod 0644 /etc/ubuntu-terminal/skipped-packages

# Railway's container capability set lacks NET_RAW, and the kernel refuses to
# exec a binary whose file capabilities exceed that set: Ubuntu's ping
# (cap_net_raw) fails with "Operation not permitted" for every user, root
# included. Without file capabilities ping falls back to ICMP datagram
# sockets, which work under sudo.
RUN set -eux; \
    caps="$(getcap -r /usr 2> /dev/null || true)"; \
    for bin in $(printf '%s\n' "$caps" | awk 'NF {print $1}'); do setcap -r "$bin"; done; \
    test -z "$(getcap -r /usr 2> /dev/null || true)"

# Node.js LTS from nodejs.org. Ubuntu 26.04 ships Node.js 22.
RUN set -eux; \
    case "$(dpkg --print-architecture)" in \
      amd64) narch="x64";   sum="${NODE_SHA256_AMD64}" ;; \
      arm64) narch="arm64"; sum="${NODE_SHA256_ARM64}" ;; \
      *) echo "unsupported architecture: $(dpkg --print-architecture)" >&2; exit 1 ;; \
    esac; \
    tarball="node-v${NODE_VERSION}-linux-${narch}.tar.xz"; \
    curl -fsSL --retry 5 --retry-all-errors --retry-delay 2 -o "/tmp/${tarball}" \
      "https://nodejs.org/dist/v${NODE_VERSION}/${tarball}"; \
    echo "${sum}  /tmp/${tarball}" | sha256sum -c -; \
    tar -xJf "/tmp/${tarball}" -C /usr/local --strip-components=1 --no-same-owner; \
    rm -f "/tmp/${tarball}" /usr/local/CHANGELOG.md /usr/local/README.md; \
    install -d /usr/local/share/doc/nodejs; \
    mv /usr/local/LICENSE /usr/local/share/doc/nodejs/LICENSE; \
    node --version; \
    npm --version

# The login user. Ubuntu's image already has "ubuntu" with uid 1000, which
# keeps file ownership on the volume stable across image updates; it only
# gets sudo and bash here. Its home is created on the volume at first boot
# from /etc/skel: Ubuntu's default dotfiles plus one line that loads this
# template's shell defaults. The sudo hint in Ubuntu's bash.bashrc is
# switched off, since the banner explains sudo.
RUN set -eux; \
    printf '\n# Ubuntu Web Terminal defaults (prompt, history, PATH). Delete these lines to opt out.\nif [ -r /etc/ubuntu-terminal/bashrc ]; then . /etc/ubuntu-terminal/bashrc; fi\n' \
      >> /etc/skel/.bashrc; \
    touch /etc/skel/.sudo_as_admin_successful; \
    if id -u ubuntu > /dev/null 2>&1; then \
      test "$(id -u ubuntu)" = 1000; \
      usermod --append --groups sudo --shell /bin/bash ubuntu; \
    else \
      useradd --uid 1000 --user-group --groups sudo --shell /bin/bash --no-create-home ubuntu; \
    fi; \
    rm -rf /home/ubuntu; \
    printf 'ubuntu ALL=(ALL:ALL) NOPASSWD: ALL\n' > /etc/sudoers.d/90-ubuntu-terminal; \
    chmod 0440 /etc/sudoers.d/90-ubuntu-terminal; \
    visudo -cf /etc/sudoers.d/90-ubuntu-terminal

COPY entrypoint.sh /usr/local/bin/ubuntu-terminal-entrypoint
COPY terminal-session.sh /usr/local/bin/terminal-session
COPY bashrc.sh /etc/ubuntu-terminal/bashrc
COPY motd.txt /etc/ubuntu-terminal/motd
COPY nginx.conf.template /etc/ubuntu-terminal/nginx.conf.template
COPY tmux.conf /etc/tmux.conf
RUN chmod 0755 /usr/local/bin/ubuntu-terminal-entrypoint /usr/local/bin/terminal-session \
 && chmod 0644 /etc/ubuntu-terminal/bashrc /etc/ubuntu-terminal/motd \
      /etc/ubuntu-terminal/nginx.conf.template /etc/tmux.conf

EXPOSE 8080

# On Railway, podman-init is PID 1 and this runs under it. tini -s registers as
# a child subreaper, so orphaned processes (the tmux server, background jobs)
# are reaped whether or not tini is PID 1.
ENTRYPOINT ["/usr/bin/tini", "-s", "--", "/usr/local/bin/ubuntu-terminal-entrypoint"]
