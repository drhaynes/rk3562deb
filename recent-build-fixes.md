**Orientation for New Chat Session:**
We are currently working on an RK3562-based Debian tablet project, specifically bringing up the open-source Panfrost GPU driver and Wayland (Phosh) environment. The build process (using `build.sh`) has been fixed to work correctly within dedicated root containers. We have also resolved initial DRM initialization failures by correctly wiring the MIPI DSI panel in the device tree (`rk3562-rk817-tablet-v10-panfrost.dts`). Currently, we have applied a few workarounds (like forcing USB to host mode for keyboard debugging and setting the GPU regulator to `always-on` to prevent early boot panics) to stabilize the system enough to reach the graphical environment. The immediate next step is to test the latest built image on the device and analyze whether the device successfully boots to the GUI, or if further kernel/userspace debugging is required using the active USB console.

# Recent Build and Boot Fixes Summary

This document summarizes the recent changes made to resolve build failures inside dedicated containers and fix boot/graphical environment initialization issues on the RK3562 tablet.

## 1. Upstream Kernel Patch Metadata Fix
- **Files Modified:** `overlay/kernel-patches/rk817-boot-ocv-calibration.patch`, `src/kernel/kernel-patches/rk817-boot-ocv-calibration.patch`
- **Change:** Corrected malformed hunk headers (`@@ -1764,18 +1773,49 @@` instead of `47`, and adjusted subsequent hunk offsets).
- **Reason:** The `patch` utility failed with a syntax error, halting the kernel build. Adjusting the line counts allowed it to apply cleanly.
- **Cleanup:** Removed leftover ad-hoc debug scripts (`patch_rk808.py`, `patch_script.py`, `fix_build.sh`, etc.) from the workspace.

## 2. Container Build Fixes
- **Files Modified:** `build_rootfs.sh`, `build.sh`
- **Change:** 
  - Changed `mount --bind` to `mount --rbind` for `/proc`, `/sys`, and `/dev` during the `debootstrap` chroot setup in `build_rootfs.sh`.
  - Introduced conditional `SUDO_CMD` logic in `build.sh`'s `create_image` function so it can run cleanly as `root` in a container without `sudo` installed.
  - Added `mkdosfs` to the initial dependency check in `build.sh`.
- **Reason:** 
  - Non-recursive bind mounts of `/dev` in containers resulted in missing device nodes (like `/dev/null`), causing the `dictionaries-common` post-installation scripts to crash.
  - Hardcoded `sudo` calls in the image generation step caused failures on containers running natively as root.
  - Missing `dosfstools` caused `genimage` to fail during the final `.img` creation step.

## 3. Display Panel Probing Fix
- **Files Modified:** `overlay/arch/arm64/boot/dts/rockchip/rk3562-rk817-tablet-v10-panfrost.dts`
- **Change:** Moved the `panel@0` node from the root of the device tree into the `&dsi` (MIPI DSI) controller subsystem node.
- **Reason:** The standard `simple-panel-dsi` driver requires the panel to be a child of the DSI bus to probe successfully. Without this, the driver deferred probing (`-EPROBE_DEFER` / `-517`), stalling the entire DRM and Panfrost graphics pipeline and preventing the GUI from starting.

## 4. Boot Diagnostics and Stability (Workarounds)
- **Files Modified:** `overlay/arch/arm64/boot/dts/rockchip/rk3562-rk817-tablet-v10-panfrost.dts`, `build_rootfs.sh`
- **Change:**
  - Set `dr_mode = "host";` for the USB OTG port in the device tree to keep USB keyboards active during Linux boot for live console debugging.
  - Added `regulator-always-on;` to the `vdd_gpu` regulator to prevent hard bus faults / kernel panics during early Panfrost probing caused by aggressive power domain collapsing. *(Note: This impacts battery life and should be refined in the future).*
  - Fixed a `systemd` ordering cycle by removing `power-profiles-daemon.service` from the `After=` and `Wants=` rules of `rk-power-profile-sync.service` in `build_rootfs.sh`.
