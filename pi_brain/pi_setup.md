# Raspberry Pi 5 — Void Linux Setup

## Install Void Linux

1. Download Void ARM64 rootfs: https://voidlinux.org/download/
2. Write to SD card via `bmaptool` or `dd`
3. Boot Pi, login as `root`/`voidlinux`

## Post-Install Setup

```bash
# Set root password
passwd

# Create user
useradd -m -G wheel,docker username
passwd username
```

## Install Dependencies

```bash
# Update
xbps-install -Suv

# Core tools
xbps-install curl git nano

# Docker (for Proton Bridge)
xbps-install docker
usermod -aG docker username
systemctl enable docker

# Node.js 20+
xbps-install nodejs

# Build tools (for native npm packages)
xbps-install base-devel
```

## Install Ollama

```bash
curl -fsSL https://ollama.ai/install.sh | sh
systemctl enable ollama
ollama pull gemma3:1b-instruct-q4_0
```

## Timezone

```bash
ln -sf /usr/share/zoneinfo/Europe/Amsterdam /etc/localtime
```

## Services

```bash
## Services

```bash
# Docker, Ollama
systemctl enable docker ollama

# Email Cleaner (after deployment)
cp /srv/email-cleaner/ops/systemd/*.service /etc/systemd/system/
systemctl enable email-clean.timer
```
```
