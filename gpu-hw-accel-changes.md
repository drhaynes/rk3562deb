# GPU Hardware Acceleration Changes

This document summarizes the modifications made to the `rkdebian` project to transition from the proprietary Rockchip Mali GPU stack to the open-source Mesa Panfrost driver. This change aims to resolve severe UI performance issues (e.g., 4fps in Qt Wayland-EGL apps) and lack of proper OpenGL support.

## 1. Default GPU Stack Switch

- **Files Modified:** `build.sh`, `build_rootfs.sh`
- **Change:** Changed the default value of the `RKDEBIAN_GPU_STACK` environment variable from `mali` to `panfrost`. This instructs the build system to use the Mesa Panfrost drivers and the corresponding device tree (`rk3562-rk817-tablet-v10-panfrost.dtb`) instead of packaging the proprietary `libmali` blobs.

## 2. OpenCL Support

- **Files Modified:** `build_rootfs.sh`
- **Change:** Added `mesa-opencl-icd` to the `apt-get install` package list for the root filesystem. Since switching away from the proprietary Mali blob removes `libMaliOpenCL.so`, this package provides OpenCL compute capabilities via Mesa's Rusticl/Clover implementations.

## 3. Removal of Software Rendering Workarounds

- **Files Modified:** `build_rootfs.sh`
- **Change:** Removed the `plasma-discover-safe` wrapper script and the `sed` commands that modified the `org.kde.discover.desktop` file. The wrapper previously forced `QSG_RHI_BACKEND`, `QT_QUICK_BACKEND`, and `QT_OPENGL` to `software` because the Mali driver could not initialize EGL properly. With Panfrost, Qt hardware acceleration works natively.

## 4. Re-enabling Chromium GPU Compositing

- **Files Modified:** `build_rootfs.sh`
- **Change:** Removed the `--disable-gpu` fallback flag from the Chromium environment configuration (`/etc/chromium.d/rk3562-hw-accel`). This allows Chromium to utilize hardware-accelerated GPU compositing via Panfrost (Glamor/Wayland) instead of falling back to slow software compositing.

## 5. Boot Splash Removal (Verbose Boot)

- **Files Modified:** `build_rootfs.sh`, `extlinux.conf`
- **Change:**
  - Stripped out the installation of Plymouth, Plymouth themes, and the custom static framebuffer logo script (`boot-fb-logo.sh`) from the rootfs builder.
  - Edited the bootloader configuration (`extlinux.conf`) to remove the `quiet`, `nosplash`, and `loglevel=0` parameters. Added `console=tty1`.
  - **Reason:** Plymouth often has compatibility issues with different graphics stacks on Rockchip devices. Disabling it and enabling verbose logging to the screen makes debugging display bring-up issues much easier.

## 6. Upstream Kernel Patch Fixes

During the build process, two local kernel patches failed to apply against the latest upstream Rockchip `develop-6.1` kernel branch. These were resolved:

- **`overlay/kernel-patches/rk817-dev-off-poweroff.patch`**: Recreated the patch with perfectly aligned unified diff headers and context lines to account for a new upstream `regmap_update_bits` call inside `rk817_shutdown_prepare`.
- **`overlay/kernel-patches/rk817-boot-ocv-calibration.patch`**: The upstream kernel removed a helper function (`rk817_bat_get_ocv_voltage`). Injected a localized reimplementation of this function directly into the patch to safely read the `RK817_GAS_GAUGE_OCV_VOL_H/L` registers via `regmap_read`, allowing the boot OCV calibration logic to function as originally intended.
