# Docker-OSX on Proxmox VE

Run macOS in Docker on Proxmox for BlueBubbles iMessage bridge and other hackintosh needs.

## Prerequisites

- Proxmox VE 8.x with Docker installed
- CPU with virtualization support (VT-x/AMD-V)
- KVM enabled (`/dev/kvm` available)
- Minimum 8GB RAM available
- 64GB+ storage

## Quick Start

### 1. Create Working Directory

```bash
ssh root@your-proxmox-ip
mkdir -p /opt/docker-osx
cd /opt/docker-osx
```

### 2. Create Empty Disk Image

```bash
qemu-img create -f qcow2 maindisk.qcow2 64G
```

### 3. Create Startup Script

```bash
cat > start-macos.sh << 'SCRIPT'
#!/bin/bash
docker run -d \
  --name macos \
  --dns=1.1.1.1 \
  --device /dev/kvm \
  -p 5999:5999 \
  -p 50922:10022 \
  -p 1234:1234 \
  -v /opt/docker-osx/maindisk.qcow2:/image \
  -e IMAGE_PATH=/image \
  -e 'EXTRA=-display none -vnc 0.0.0.0:99,password=off' \
  -e WIDTH=1920 \
  -e HEIGHT=1080 \
  -e GENERATE_UNIQUE=true \
  sickcodes/docker-osx:latest
SCRIPT
chmod +x start-macos.sh
```

### 4. Start the Container

```bash
./start-macos.sh
```

First run downloads macOS recovery (~850MB) and converts it. Wait for QEMU to start:

```bash
# Watch logs until you see "qemu-system-x86_64"
docker logs -f macos
```

### 5. Connect via VNC

**Option A: noVNC (Web-based - Recommended)**

```bash
docker run -d \
  --name novnc \
  --network host \
  -e REMOTE_HOST=127.0.0.1 \
  -e REMOTE_PORT=5999 \
  dougw/novnc
```

Access: `http://your-proxmox-ip:8081`

**Option B: Direct VNC**

Connect to `your-proxmox-ip:5999` (no password)

> Note: macOS Screen Sharing may not work with QEMU VNC. Use a proper VNC client like RealVNC or TigerVNC.

### 6. Install macOS

1. Select **macOS Base System** in OpenCore boot picker
2. Open **Disk Utility**
3. Erase the ~64GB disk as **APFS** (name: `Macintosh HD`)
4. Close Disk Utility
5. Select **Reinstall macOS**
6. Follow prompts (30-60 minutes, mostly unattended)

### 7. Post-Installation

After macOS boots to desktop:

```bash
# Remove install media reference (optional - speeds up boot)
docker exec macos sed -i '/InstallMedia/d' /home/arch/OSX-KVM/Launch.sh
```

#### Enable SSH (for remote management)

In macOS: System Settings → General → Sharing → Remote Login → ON

```bash
# SSH from Proxmox host
ssh user@localhost -p 50922
```

#### Disable Sleep

```bash
sudo pmset -a sleep 0 disksleep 0 displaysleep 0
```

## Port Mapping

| Host Port | Container Port | Purpose |
|-----------|----------------|---------|
| 5999 | 5999 | VNC (display :99) |
| 50922 | 10022 | SSH |
| 1234 | 1234 | BlueBubbles API |

## Persistence

The `maindisk.qcow2` file contains your macOS installation. Back it up!

```bash
# Stop container first for consistent backup
docker stop macos
cp /opt/docker-osx/maindisk.qcow2 /backup/macos-backup-$(date +%Y%m%d).qcow2
docker start macos
```

## Troubleshooting

### Container exits immediately

Check KVM is available:
```bash
ls -la /dev/kvm
# Should show: crw-rw---- 1 root kvm ...
```

### VNC won't connect

1. Verify QEMU is running: `docker exec macos pgrep qemu`
2. Check port is listening: `ss -tlnp | grep 5999`
3. Use noVNC instead of native VNC clients

### macOS won't boot after install

Select **Macintosh HD** (not macOS Base System) in OpenCore picker.

### iMessage activation fails

1. Generate unique serial numbers (use `GENERATE_UNIQUE=true`)
2. Reset NVRAM from OpenCore picker
3. Wait 24 hours and retry
4. May need to contact Apple Support

## BlueBubbles Setup

After macOS is running:

1. Download BlueBubbles Server from https://bluebubbles.app/downloads/
2. Grant permissions: Full Disk Access, Accessibility, Automation
3. Sign into iCloud/iMessage in Messages.app
4. Configure BlueBubbles server (port 1234)
5. Connect clients to `http://proxmox-ip:1234`

## Resources

- [Docker-OSX](https://github.com/sickcodes/Docker-OSX)
- [OSX-KVM](https://github.com/kholia/OSX-KVM)
- [BlueBubbles](https://bluebubbles.app/)

## License

MIT
