# Flatcar GPU Driver Container - Changes Summary

## Date
November 5, 2025

## Problem
The NVIDIA GPU driver container needed to support Direct Rendering Infrastructure (DRI) on Flatcar Linux with kernel 6.6.95. The `/dev/dri` directory and device nodes were not being created, preventing GPU rendering capabilities for containerized applications. During the implementation, several build and runtime issues were encountered that needed to be resolved.

## Objective
Enable Direct Rendering Infrastructure (DRI) support by loading the `nvidia-drm` kernel module, which creates the `/dev/dri` directory and device nodes (`/dev/dri/card0`, `/dev/dri/renderD128`) required for GPU rendering and compute workloads.

## Issues Encountered
1. **Interactive Kernel Configuration**: The kernel `modules_prepare` step was running interactively and prompting for configuration options (specifically for compressed debug information). This caused the build script to hang, as subsequent commands were being fed into the interactive prompt instead of executing. *Note: This might need to be reversed if/when using image in production nodes.*

2. **Missing nvidia-drm Module**: The `nvidia-drm.ko` kernel module was not being packaged in the precompiled driver package, so it wasn't available to load during runtime.

3. **Unbound Variable Error**: The `nvidia_drm_sign_args` variable was not initialized, causing script failures when trying to package the nvidia-drm module.

4. **Missing DRM Kernel Module Dependencies**: The `nvidia-drm` module depends on kernel DRM subsystem modules (`drm` and `drm_kms_helper`) which weren't loaded before attempting to load nvidia-drm, causing "Unknown symbol" errors.

## Solution Overview
Seven main changes were made to `/flatcar/nvidia-driver`:

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

### 5. Initialize nvidia_drm_sign_args Variable (Lines 177, 186-187, 191)
**Change**: Added initialization and signing support for nvidia-drm module

**Before**: No signing variable for nvidia-drm

**After**:
```bash
local nvidia_drm_sign_args=""
# ...
if [ -n "${PRIVATE_KEY}" ]; then
    # ... other signing commands ...
    sign-file sha512 \$DONKEY_FILE pubkey.x509 nvidia-drm.ko
    nvidia_drm_sign_args="--signed"
fi
```

**Why**: The `nvidia_drm_sign_args` variable must be initialized to avoid "unbound variable" errors. When module signing is enabled with `PRIVATE_KEY`, the nvidia-drm module needs to be signed like the other kernel modules.

### 6. Load Kernel DRM Modules Before nvidia-drm (Lines 236-237)
**Change**: Added loading of kernel DRM prerequisites

**Before**:
```bash
modprobe -d ${NVIDIA_KMODS_DIR} -a nvidia nvidia-uvm nvidia-modeset nvidia-drm
```

**After**:
```bash
# Load kernel DRM modules first (required by nvidia-drm)
modprobe drm
modprobe drm_kms_helper

# Now load NVIDIA modules
modprobe -d ${NVIDIA_KMODS_DIR} -a nvidia nvidia-uvm nvidia-modeset nvidia-drm
```

**Why**: The `nvidia-drm` module depends on kernel DRM subsystem symbols from `drm` and `drm_kms_helper`. These must be loaded first to resolve symbol dependencies (drm_kms_helper_poll_fini, drm_atomic_helper_*, etc.).

For example from logs:

```
[ 8474.208869] nvidia_drm: Unknown symbol drm_kms_helper_poll_fini (err -2)
[ 8474.209932] nvidia_drm: Unknown symbol drm_kms_helper_poll_disable (err -2)
[ 8474.211076] nvidia_drm: Unknown symbol drm_kms_helper_poll_init (err -2)
[ 8474.212163] nvidia_drm: Unknown symbol drm_atomic_helper_disable_plane (err -2)
[ 8474.213295] nvidia_drm: Unknown symbol drm_helper_hpd_irq_event (err -2)
[ 8474.214349] nvidia_drm: Unknown symbol drm_atomic_helper_check (err -2)
[ 8474.215360] nvidia_drm: Unknown symbol __drm_atomic_helper_plane_destroy_state (err -2)
[ 8474.216566] nvidia_drm: Unknown symbol drm_atomic_helper_connector_destroy_state (err -2)
[ 8474.217820] nvidia_drm: Unknown symbol drm_atomic_helper_plane_reset (err -2)
[ 8474.218879] nvidia_drm: Unknown symbol drm_helper_mode_fill_fb_struct (err -2)
[ 8474.219996] nvidia_drm: Unknown symbol drm_atomic_helper_set_config (err -2)
```

### 7. Added nvidia-drm Module Unloading (Lines 271-274)
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
After applying these changes, Direct Rendering Infrastructure (DRI) support is fully functional:
- ✅ **Primary Goal Achieved**: `/dev/dri` directory created with device nodes (`card0`, `renderD128`)
- ✅ All NVIDIA kernel modules load correctly (including `nvidia-drm`)
- ✅ Kernel DRM modules load properly (`drm`, `drm_kms_helper`)
- ✅ GPU is accessible via `nvidia-smi` from containers using `--runtime=nvidia`
- ✅ Driver version 535.183.01 works on Flatcar kernel 6.6.95
- ✅ Tested successfully on AWS Tesla T4 GPU

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
Add Direct Rendering Infrastructure (DRI) support for Flatcar 6.6.95

Enable /dev/dri device creation by adding nvidia-drm module support:
- Add nvidia-drm.ko to precompiled package
- Initialize nvidia_drm_sign_args variable and add signing support
- Load kernel DRM modules (drm, drm_kms_helper) before nvidia-drm
- Add nvidia-drm module loading to create /dev/dri devices
- Add proper nvidia-drm cleanup in unload function

Build improvements:
- Add 'make olddefconfig' to handle kernel config prompts non-interactively
- Remove invalid 'nvidia-drm.o' build target

This enables GPU rendering and compute capabilities for containerized
applications by creating /dev/dri/card0 and /dev/dri/renderD128 device
nodes through the nvidia-drm kernel module.

Tested successfully on AWS Tesla T4 with Flatcar Linux 6.6.95 and
NVIDIA driver 535.183.01.
```

