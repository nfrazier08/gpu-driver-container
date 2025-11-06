# Cleanup Instructions

## Local Cleanup (Your Mac)

### 1. Review Your Changes
```bash
cd /Users/nfrazier/git/personal_gh/gpu-driver-container

# See what files were modified
git status

# Review the changes
git diff flatcar/nvidia-driver
```

### 2. Commit Your Changes
```bash
# Stage the changes
git add flatcar/nvidia-driver

# Optional: Add the summary document (or add to .gitignore if it's just for reference)
git add flatcar/CHANGES_SUMMARY.md

# Commit with a descriptive message
git commit -m "Fix Flatcar 6.6.95 kernel build - add non-interactive config

- Add 'make olddefconfig' to handle new kernel config options automatically
- Remove invalid 'nvidia-drm.o' build target
- Add nvidia-drm module loading for /dev/dri support
- Add proper nvidia-drm cleanup in unload function

This fixes the build hanging issue on newer Flatcar kernels where
modules_prepare would prompt for configuration options interactively."
```

### 3. Push to Your Remote
```bash
# Push to your fork
git push origin main
```

### 4. Optional: Clean Up Local Docker Images (if you built any locally)
```bash
# List local NVIDIA driver images
docker images | grep nvidia-driver

# Remove old/unused images if needed
docker rmi <image-id>
```

---

## Remote Cleanup (AWS GPU Node - Flatcar)

### 1. Stop and Remove Old Containers
```bash
# Stop the running driver container
docker stop nvidia-driver

# Remove the container
docker rm nvidia-driver

# Verify it's gone
docker ps -a | grep nvidia-driver
```

### 2. Clean Up Old/Failed Container Attempts
```bash
# List all stopped containers
docker ps -a

# Remove all stopped containers (optional, be careful!)
docker container prune

# Or remove specific old nvidia-driver containers
docker rm <container-id>
```

### 3. Clean Up Old Docker Images
```bash
# List all nvidia driver images
docker images | grep nvidia

# Remove old/untagged images
docker image prune

# Or remove specific old versions
docker rmi nvidia/nvidia-driver-flatcar:<old-version>
```

### 4. Verify Kernel Modules Are Unloaded
```bash
# Check if any NVIDIA modules are still loaded
lsmod | grep nvidia

# If modules are loaded, unload them manually
sudo rmmod nvidia-drm nvidia-modeset nvidia-uvm nvidia

# Verify they're gone
lsmod | grep nvidia
```

### 5. Clean Up Mount Points (if needed)
```bash
# Check if /run/nvidia is clean
ls -la /run/nvidia/

# If there are stale files, you can remove the directory
# (it will be recreated by the new container)
sudo rm -rf /run/nvidia/driver

# Clean up logs if needed
sudo rm -rf /tmp/nvidia/*
```

### 6. Verify GPU is Still Accessible
```bash
# Check if the host can see the GPU
lspci | grep -i nvidia

# Check device nodes
ls -la /dev/nvidia*
```

---

## Fresh Deployment Checklist

Once cleanup is complete, follow these steps:

### 1. Get Your Updated Code on the AWS Node

#### Option A: Pull Updates (if repo already exists)
```bash
# On your AWS GPU node
cd /path/to/gpu-driver-container
git pull origin main
cd flatcar
```

#### Option B: Reclone the Repository (fresh start)
```bash
# On your AWS GPU node

# Remove old repo directory
rm -rf /path/to/gpu-driver-container

# Clone your updated repo
git clone git@personal:nfrazier08/gpu-driver-container.git /path/to/gpu-driver-container

# Or if using HTTPS:
# git clone https://github.com/nfrazier08/gpu-driver-container.git /path/to/gpu-driver-container

# Navigate to flatcar directory
cd /path/to/gpu-driver-container/flatcar
```

### 2. Build the Updated Image
```bash
DRIVER_VERSION=535.183.01

docker build --pull \
    --tag nvidia/nvidia-driver-flatcar:${DRIVER_VERSION} \
    --file Dockerfile .
```

### 3. Run the New Container
```bash
docker run -d --privileged --pid=host \
    --restart=unless-stopped \
    -v /run/nvidia:/run/nvidia:shared \
    -v /tmp/nvidia:/var/log \
    -v /usr/lib64/modules:/usr/lib64/modules \
    --name nvidia-driver \
    nvidia/nvidia-driver-flatcar:${DRIVER_VERSION} init
```

### 4. Monitor the Logs
```bash
# Watch the startup process
docker logs -f nvidia-driver

# Wait for: "Done, now waiting for signal"
```

### 5. Verify Everything Works
```bash
# Check loaded modules (should include nvidia-drm now)
lsmod | grep nvidia

# Verify /dev/dri exists
ls -la /dev/dri/

# Test GPU access
docker run --rm --runtime=nvidia nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# Test a CUDA workload
docker run --rm --runtime=nvidia nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi -L
```

---

## Troubleshooting Cleanup Issues

### If Modules Won't Unload
```bash
# Check what's using the modules
lsof | grep nvidia

# Force kill any processes using the GPU
# (Be careful with this!)
sudo fuser -k /dev/nvidia*

# Try unloading again
sudo rmmod nvidia-drm nvidia-modeset nvidia-uvm nvidia
```

### If Docker Daemon Has Issues
```bash
# Restart Docker daemon
sudo systemctl restart docker

# Check Docker status
sudo systemctl status docker

# View Docker logs
journalctl -u docker -n 50
```

### If /run/nvidia is Busy
```bash
# Check what's mounted
mount | grep nvidia

# Unmount if needed
sudo umount /run/nvidia/driver
```

---

## What to Keep vs Delete

### Keep
- ✅ `/Users/nfrazier/git/personal_gh/gpu-driver-container/flatcar/nvidia-driver` (modified file)
- ✅ `flatcar/CHANGES_SUMMARY.md` (documentation)
- ✅ `flatcar/CLEANUP_INSTRUCTIONS.md` (this file)
- ✅ `flatcar/Dockerfile` (no changes, but needed)

### Can Delete (Optional)
- ❌ Old/failed Docker containers on AWS node
- ❌ Old Docker images on AWS node (if you want to save space)
- ❌ `/tmp/nvidia/*` logs on AWS node (if you want fresh logs)
- ❌ The summary files if you just want them for reference and don't want to commit them

### Don't Delete
- 🚫 `/flatcar/Dockerfile`
- 🚫 Any other files in the repo
- 🚫 `/dev/nvidia*` device nodes on the host
- 🚫 Kernel modules while they're loaded

---

## Quick Cleanup Script (AWS Node)

Save this as `cleanup.sh` for easy cleanup:

```bash
#!/bin/bash
set -e

echo "Stopping nvidia-driver container..."
docker stop nvidia-driver 2>/dev/null || true

echo "Removing nvidia-driver container..."
docker rm nvidia-driver 2>/dev/null || true

echo "Unloading NVIDIA kernel modules..."
sudo rmmod nvidia-drm 2>/dev/null || true
sudo rmmod nvidia-modeset 2>/dev/null || true
sudo rmmod nvidia-uvm 2>/dev/null || true
sudo rmmod nvidia 2>/dev/null || true

echo "Cleaning up old images..."
docker image prune -f

echo "Cleaning up logs..."
sudo rm -rf /tmp/nvidia/*

echo "Cleanup complete! Ready for fresh deployment."
lsmod | grep nvidia || echo "No NVIDIA modules loaded (good!)"
```

Run with:
```bash
chmod +x cleanup.sh
./cleanup.sh
```

