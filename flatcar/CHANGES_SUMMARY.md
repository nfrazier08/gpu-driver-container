# Flatcar GPU Driver Container - Changes Summary

## Date
November 5, 2025

## Problem
The NVIDIA GPU driver container failed to build on Flatcar Linux with kernel 6.6.95, showing the error:
```
Failed to read kernel interface 'nv-linux.o'
```

## Root Causes
1. **Interactive Kernel Configuration**: The kernel `modules_prepare` step was running interactively and prompting for configuration options (specifically for compressed debug information). This caused the build script to hang, as subsequent commands were being fed into the interactive prompt instead of executing.

2. **Missing nvidia-drm Module**: The `nvidia-drm.ko` kernel module was not included in the precompiled driver package, causing module loading failures even after successful compilation.

## Solution Overview
Five main changes were made to `/flatcar/nvidia-driver`:

### 1. Added Non-Interactive Kernel Configuration (Line 127)
**Change**: Added `make olddefconfig` before `make modules_prepare`

**Before**:
```bash
cp /lib/modules/${KERNEL_VERSION}/build/.config /usr/src/linux
make -C /usr/src/linux modules_prepare
```

**After**:
```bash
cp /lib/modules/${KERNEL_VERSION}/build/.config /usr/src/linux
make -C /usr/src/linux olddefconfig
make -C /usr/src/linux modules_prepare
```

**Why**: `olddefconfig` automatically sets default values for new kernel config options without prompting interactively. This prevents the build from hanging on configuration prompts.

### 2. Fixed Driver Compilation Targets (Line 139)
**Change**: Removed invalid build target `nvidia-drm.o`

**Before**:
```bash
make -j ${MAX_THREADS} SYSSRC=/lib/modules/${KERNEL_VERSION}/source nv-linux.o nv-modeset-linux.o nvidia-drm.o
```

**After**:
```bash
make -j ${MAX_THREADS} SYSSRC=/lib/modules/${KERNEL_VERSION}/source nv-linux.o nv-modeset-linux.o
```

**Why**: Only kernel interface files (`nv-linux.o` and `nv-modeset-linux.o`) need to be built at compile time. The `nvidia-drm` module is built as a complete kernel module (`nvidia-drm.ko`) during the full driver installation, not as an interface object file.

### 3. Added nvidia-drm Module Loading (Line 229)
**Change**: Added `nvidia-drm` to the list of modules to load

**Before**:
```bash
modprobe -d ${NVIDIA_KMODS_DIR} -a nvidia nvidia-uvm nvidia-modeset
```

**After**:
```bash
modprobe -d ${NVIDIA_KMODS_DIR} -a nvidia nvidia-uvm nvidia-modeset nvidia-drm
```

**Why**: The `nvidia-drm` module is required to create the `/dev/dri` directory and device nodes, which are needed for Direct Rendering Infrastructure (DRI) support.

### 4. Added nvidia-drm.ko to Precompiled Package (Lines 210-212)
**Change**: Added `nvidia-drm.ko` to the mkprecompiled packaging command

**Before**: Only packaged `nvidia.ko`, `nvidia-modeset.ko`, and `nvidia-uvm.ko`

**After**:
```bash
--kernel-module nvidia-uvm.ko                                \
${nvidia_uvm_sign_args}                                      \
--target-directory .                                         \
--kernel-module nvidia-drm.ko                                \
${nvidia_drm_sign_args}                                      \
--target-directory .
```

**Why**: The `nvidia-drm` kernel module must be included in the precompiled package so it can be installed during `init` mode. Without this, the module would not be available for loading.

### 5. Added nvidia-drm Module Unloading (Lines 260-263)
**Change**: Added proper unloading of `nvidia-drm` module

**Before**: Only unloaded `nvidia-modeset`, `nvidia-uvm`, and `nvidia`

**After**:
```bash
if [ -f /sys/module/nvidia_drm/refcnt ]; then
    rmmod_args+=("nvidia-drm")
    ((++nvidia_deps))
fi
# ... followed by nvidia-modeset, nvidia-uvm, nvidia
```

**Why**: Ensures proper cleanup when stopping the driver container. The `nvidia-drm` module must be unloaded before `nvidia-modeset` since it depends on it.

## Files Modified
- `/flatcar/nvidia-driver` - Main driver installation script

## Testing Results
After applying these changes:
- ✅ Driver compiles successfully without hanging
- ✅ All NVIDIA kernel modules load correctly
- ✅ GPU is accessible via `nvidia-smi`
- ✅ `/dev/dri` directory is created with proper device nodes
- ✅ Driver version 535.183.01 works on Flatcar kernel 6.6.95

## Deployment Instructions
```bash
# On AWS GPU node running Flatcar
cd /path/to/gpu-driver-container/flatcar

DRIVER_VERSION=535.183.01

# Build the updated image
docker build --pull \
    --tag nvidia/nvidia-driver-flatcar:${DRIVER_VERSION} \
    --file Dockerfile .

# Run the driver container
docker run -d --privileged --pid=host \
    --restart=unless-stopped \
    -v /run/nvidia:/run/nvidia:shared \
    -v /tmp/nvidia:/var/log \
    -v /usr/lib64/modules:/usr/lib64/modules \
    --name nvidia-driver \
    nvidia/nvidia-driver-flatcar:${DRIVER_VERSION} init

# Verify
docker logs -f nvidia-driver
docker run --rm --runtime=nvidia nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
ls -la /dev/dri/
```

## Additional Notes
- The objtool warnings about "naked return found in RETHUNK build" are harmless and can be ignored
- The BTF (BPF Type Format) warnings about unavailable vmlinux are expected on Flatcar and do not affect functionality
- The driver container must remain running to keep the GPU drivers loaded
- Use `--restart=unless-stopped` to ensure the driver loads automatically after system reboot

## Git Commit Message Suggestion
```
Fix Flatcar 6.6.95 kernel build and nvidia-drm support

- Add 'make olddefconfig' to handle new kernel config options automatically
- Remove invalid 'nvidia-drm.o' build target
- Add nvidia-drm.ko to precompiled package
- Add nvidia-drm module loading for /dev/dri support
- Add proper nvidia-drm cleanup in unload function

This fixes the build hanging issue on newer Flatcar kernels where
modules_prepare would prompt for configuration options interactively,
and ensures nvidia-drm module is properly packaged and loaded for
Direct Rendering Infrastructure (DRI) support.
```

