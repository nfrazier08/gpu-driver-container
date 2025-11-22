# Ubuntu 22.04 GPU Driver Container - DRI Support Changes Summary

## Date
November 21, 2025

## Problem
The NVIDIA GPU driver container needed to support Direct Rendering Infrastructure (DRI) on Ubuntu 22.04. The `/dev/dri` directory and device nodes were not being created, preventing GPU rendering capabilities for containerized applications.

## Objective
Enable Direct Rendering Infrastructure (DRI) support by loading the `nvidia-drm` kernel module, which creates the `/dev/dri` directory and device nodes (`/dev/dri/card0`, `/dev/dri/renderD128`) required for GPU rendering and compute workloads.

## Solution Overview
Nine main changes were made to `/ubuntu22.04/nvidia-driver`, following the pattern established for flatcar and completing the nvidia-drm module parameter support:

### 1. Initialize nvidia_drm_sign_args Variable (Line 129)
**Change**: Added initialization for nvidia-drm signing support

**Added**:
```bash
local nvidia_drm_sign_args=""
```

**Why**: The `nvidia_drm_sign_args` variable must be initialized to avoid "unbound variable" errors. When module signing is enabled with `PRIVATE_KEY`, the nvidia-drm module needs to be signed like the other kernel modules.

### 2. Add nvidia-drm Module Signing (Lines 158-167)
**Change**: Added signing support for nvidia-drm module

**Before**:
```bash
sign-file sha512 \$DONKEY_FILE pubkey.x509 nvidia-uvm.ko"
nvidia_sign_args="--linked-module nvidia.ko --signed-module nvidia.ko.sign"
nvidia_modeset_sign_args="--linked-module nvidia-modeset.ko --signed-module nvidia-modeset.ko.sign"
nvidia_uvm_sign_args="--signed"
```

**After**:
```bash
sign-file sha512 \$DONKEY_FILE pubkey.x509 nvidia-uvm.ko &&                                     \
sign-file sha512 \$DONKEY_FILE pubkey.x509 nvidia-drm.ko"
nvidia_sign_args="--linked-module nvidia.ko --signed-module nvidia.ko.sign"
nvidia_modeset_sign_args="--linked-module nvidia-modeset.ko --signed-module nvidia-modeset.ko.sign"
nvidia_uvm_sign_args="--signed"
nvidia_drm_sign_args="--signed"
```

**Why**: When module signing is enabled with `PRIVATE_KEY`, the nvidia-drm module needs to be signed like the other kernel modules.

### 3. Add nvidia-drm.ko to Precompiled Package (Lines 186-188)
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

### 4. Load Kernel DRM Modules Before nvidia-drm (Lines 305-315)
**Change**: Added loading of kernel DRM prerequisites and nvidia-drm module

**Before**:
```bash
echo "Loading NVIDIA driver kernel modules..."
set -o xtrace +o nounset
modprobe nvidia "${NVIDIA_MODULE_PARAMS[@]}"
modprobe nvidia-uvm "${NVIDIA_UVM_MODULE_PARAMS[@]}"
modprobe nvidia-modeset "${NVIDIA_MODESET_MODULE_PARAMS[@]}"
set +o xtrace -o nounset
```

**After**:
```bash
echo "Loading kernel DRM modules..."
modprobe drm
modprobe drm_kms_helper

echo "Loading NVIDIA driver kernel modules..."
set -o xtrace +o nounset
modprobe nvidia "${NVIDIA_MODULE_PARAMS[@]}"
modprobe nvidia-uvm "${NVIDIA_UVM_MODULE_PARAMS[@]}"
modprobe nvidia-modeset "${NVIDIA_MODESET_MODULE_PARAMS[@]}"
modprobe nvidia-drm "${NVIDIA_DRM_MODULE_PARAMS[@]}"
set +o xtrace -o nounset
```

**Why**: The `nvidia-drm` module depends on kernel DRM subsystem symbols from `drm` and `drm_kms_helper`. These must be loaded first to resolve symbol dependencies (drm_kms_helper_poll_fini, drm_atomic_helper_*, etc.). The `nvidia-drm` module is required to create the `/dev/dri` directory and device nodes.

### 5. Add nvidia-drm Module Unloading (Lines 406-409)
**Change**: Added proper unloading of `nvidia-drm` module

**Before**: Only unloaded `nvidia-modeset`, `nvidia-uvm`, `nvidia-peermem`, and `nvidia`

**After**:
```bash
if [ -f /sys/module/nvidia_drm/refcnt ]; then
    rmmod_args+=("nvidia-drm")
    ((++nvidia_deps))
fi
# ... followed by nvidia-modeset, nvidia-uvm, nvidia-peermem, nvidia
```

**Why**: Ensures proper cleanup when stopping the driver container. The `nvidia-drm` module must be unloaded before `nvidia-modeset` since it depends on it.

### 6. Initialize NVIDIA_DRM_MODULE_PARAMS (Line 16)
**Change**: Added module parameters array for nvidia-drm

**Added**:
```bash
NVIDIA_MODULE_PARAMS=()
NVIDIA_UVM_MODULE_PARAMS=()
NVIDIA_MODESET_MODULE_PARAMS=()
NVIDIA_DRM_MODULE_PARAMS=()
NVIDIA_PEERMEM_MODULE_PARAMS=()
```

**Why**: To maintain consistency with other NVIDIA modules and allow users to pass custom parameters to the nvidia-drm module (e.g., `modeset=1` for display configuration). Without this initialization, the script would fail with "unbound variable" errors when trying to use the parameter array.

### 7. Add nvidia-drm Configuration File Support (Lines 262-268)
**Change**: Added support for reading nvidia-drm module parameters from config file

**Added**:
```bash
    # nvidia-drm
    if [ -f "${base_path}/nvidia-drm.conf" ]; then
       while IFS="" read -r param || [ -n "$param" ]; do
           NVIDIA_DRM_MODULE_PARAMS+=("$param")
       done <"${base_path}/nvidia-drm.conf"
       echo "Module parameters provided for nvidia-drm: ${NVIDIA_DRM_MODULE_PARAMS[@]}"
    fi
```

**Why**: Allows users to configure nvidia-drm module parameters via a `/drivers/nvidia-drm.conf` file, following the same pattern as nvidia.conf, nvidia-uvm.conf, nvidia-modeset.conf, and nvidia-peermem.conf. This is essential for settings like enabling display modesetting or other DRM-specific configurations.

### 8. Use Module Parameters When Loading nvidia-drm (Line 314)
**Change**: Updated nvidia-drm module loading to use configured parameters

**Before**:
```bash
modprobe nvidia-drm
```

**After**:
```bash
modprobe nvidia-drm "${NVIDIA_DRM_MODULE_PARAMS[@]}"
```

**Why**: Applies any user-configured parameters from the `NVIDIA_DRM_MODULE_PARAMS` array when loading the module. This completes the parameter handling chain and ensures nvidia-drm can be configured just like the other NVIDIA kernel modules. Common use cases include setting `modeset=1` for display output support.

### 9. Remove --no-drm Flag from nvidia-installer (Line 457)
**Change**: Removed the `--no-drm` flag to allow nvidia-drm module installation

**Before**:
```bash
nvidia-installer --kernel-module-only --no-drm --ui=none --no-nouveau-check -m=${KERNEL_TYPE}
```

**After**:
```bash
nvidia-installer --kernel-module-only --ui=none --no-nouveau-check -m=${KERNEL_TYPE}
```

**Why**: The `--no-drm` flag explicitly prevents the nvidia-drm kernel module from being installed, even when it's successfully built. This flag was blocking DRI/DRM functionality. Removing it allows nvidia-drm.ko to be installed and loaded, enabling the creation of `/dev/dri` devices. Note: On AWS kernels without vmlinux, BTF generation will be skipped with a warning, but the module will still install and function correctly.

## Files Modified
- `/ubuntu22.04/nvidia-driver` - Main driver installation script

## Testing Instructions
After applying these changes, you can build and test the driver container:

```bash
# On Ubuntu 22.04 node with GPU
cd /path/to/gpu-driver-container

DRIVER_VERSION=535.183.01

# Build the updated image
docker build --pull \
    --build-arg DRIVER_VERSION=${DRIVER_VERSION} \
    --tag nvidia/driver:${DRIVER_VERSION}-ubuntu22.04 \
    --file ubuntu22.04/Dockerfile .

# Run the driver container
docker run -d --privileged --pid=host \
    -v /run/nvidia:/run/nvidia:shared \
    -v /var/log:/var/log \
    --name nvidia-driver \
    nvidia/driver:${DRIVER_VERSION}-ubuntu22.04

# Verify
docker logs -f nvidia-driver
ls -la /dev/dri/
docker run --rm --runtime=nvidia nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

## Expected Results
After applying these changes, Direct Rendering Infrastructure (DRI) support should be fully functional:
- ✅ **Primary Goal Achieved**: `/dev/dri` directory created with device nodes (`card0`, `renderD128`)
- ✅ All NVIDIA kernel modules load correctly (including `nvidia-drm`)
- ✅ Kernel DRM modules load properly (`drm`, `drm_kms_helper`)
- ✅ GPU is accessible via `nvidia-smi` from containers
- ✅ DRI support enables GPU rendering and compute capabilities

## Additional Notes
- These changes follow the same pattern as the flatcar implementation
- The driver container must remain running to keep the GPU drivers loaded
- Use appropriate restart policies to ensure the driver loads automatically after system reboot
- Module signing support is included for secure boot environments

## Related Changes
This implementation mirrors the DRI support added to flatcar. See `/flatcar/CHANGES_SUMMARY.md` for the original implementation details and rationale.

