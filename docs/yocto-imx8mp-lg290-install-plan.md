# Yocto Fresh-Install Plan: Olimex i.MX8MP and LG290K Time Server

**Research date:** 2026-10-24  
**Scope:** Assess and plan a clean Yocto-based install for the Olimex i.MX8MP system hosting the RTKBase/LG290K time server. This is a plan only; no target or build configuration was changed.

## Recommendation

Proceed with a Yocto prototype on separate boot media, retaining the current system as the rollback image. A board-specific Yocto machine configuration exists for the Olimex i.MX8MP SOM EVB, including its Olimex device tree and U-Boot configurations. NXP's current public i.MX BSP source identifies release `LF6.18.20_2.0.0` and a Linux `6.18.20` kernel recipe.

This is feasible, but not yet a validated, turnkey image for the exact deployed board. The Olimex machine definition is in a community third-party BSP layer, and its kernel provider settings must be reconciled with the NXP release before building. The first milestone is a bootable image with confirmed DDR, storage, Ethernet, USB serial, and PPS behavior, rather than replacing the running system.

## Findings and board-specific configuration

The running target identifies itself as Olimex i.MX8MPlus and uses the device tree `imx8mp-olimex.dtb`. The current OS is an unofficial, vendor-built Armbian image using a 5.10.72 kernel. It is not a standard Armbian image, so the current installation should be preserved as a recovery baseline.

The Yocto layer [`Freescale/meta-freescale-3rdparty`](https://github.com/Freescale/meta-freescale-3rdparty) contains [`olimex-imx8mp-evb.conf`](https://github.com/Freescale/meta-freescale-3rdparty/blob/master/conf/machine/olimex-imx8mp-evb.conf). Its relevant settings include:

- Yocto machine name: `olimex-imx8mp-evb`
- Kernel device-tree basename: `imx8mp-olimex`
- U-Boot configurations: `imx8mp_olimex_fspi_defconfig` and `imx8mp_olimex_ndm_defconfig`
- U-Boot DTB name: `imx8mp-evk.dtb`
- Board-specific LPDDR4 firmware and an NXP BSP selection

The separate U-Boot recipe [`u-boot-olimex_2024.04.bb`](https://github.com/Freescale/meta-freescale-3rdparty/blob/master/recipes-bsp/u-boot/u-boot-olimex_2024.04.bb) applies an Olimex board patch. The machine configuration's `IMX_DEFAULT_KERNEL` setting references `linux-fslc`; therefore, do **not** assume that selecting this machine automatically selects NXP's latest `linux-imx` recipe. Confirm and explicitly set the kernel provider/version against the chosen NXP manifest. Keep the Olimex DDR initialization and bootloader configuration intact.

NXP's [meta-imx repository](https://github.com/nxp-imx/meta-imx) currently documents BSP release `LF6.18.20_2.0.0`, manifest branch `imx-linux-wrynose`, and the `linux-imx` 6.18.20 recipe. Its tested-board list includes i.MX8MP EVK variants, but does not list the Olimex EVK specifically. The Olimex third-party machine layer is consequently important and must be pinned to a revision compatible with the NXP release.

## Fresh-image scope

Build a minimal headless image. Include only the operating-system and product components needed for operation:

- OpenSSH server for remote administration (the only requested administrative addition)
- RTKBase application and its required runtime
- `str2str`/RTKLIB functionality used by the configured data streams
- GPSD and Chrony for receiver data and time service
- systemd, the target's network support, USB serial support, and PPS support

Do not add a desktop, general-purpose developer toolchain, or unrelated utilities to the deployed image. Build tools belong on the separate Yocto build host. Package application/runtime dependencies into the image rather than relying on manual post-install package changes.

The initial target should use a dedicated unprivileged service account, explicit systemd units, and only the device and filesystem permissions required by the service. Keep SSH enabled for the first prototype; use key-based access and disable remote root login before production deployment.

## Implementation plan

### 1. Preserve the current system and capture a baseline

1. Record the exact SOM/EVK and carrier-board revision, boot source, storage map, U-Boot environment, and recovery/serial-console method. Confirm that the board matches the Olimex configuration and LPDDR4 assumptions.
2. Save the current bootable media or a verified restorable image, RTKBase settings, receiver configuration, and network configuration off the target.
3. Record known-good application behavior: LG290 firmware, serial settings, receiver mode/survey configuration, RTCM message counts, GPSD reports, Chrony sources, PPS visibility, and service startup order.
4. Do not overwrite the current root or boot media during the prototype.

### 2. Pin a compatible BSP source set

1. Start from NXP's manifest and release tag `imx-linux-wrynose` / `imx-6.18.20-2.0.0.xml` as documented in [NXP meta-imx](https://github.com/nxp-imx/meta-imx).
2. Add `meta-freescale-3rdparty` at a specific reviewed commit containing the Olimex machine definition. Record every layer revision in the product source manifest; do not build from floating branch heads.
3. Check all layer compatibility declarations and resolve any incompatibilities before building. Read and accept NXP EULA terms only if appropriate for the project.
4. Inspect the effective BitBake environment for `olimex-imx8mp-evb`:
   - `bitbake -e virtual/kernel`: provider, source revision, and `LINUX_VERSION` must resolve to NXP `linux-imx` 6.18.20.
   - `bitbake -e virtual/bootloader`: Olimex U-Boot provider, version, configuration, and required patch must be selected.
   - Confirm the generated deployment includes `imx8mp-olimex.dtb`, the correct boot container, DDR training firmware, and boot files expected by this board.
5. If the layer's current kernel default prevents selecting `linux-imx`, resolve it with a small, tracked product-layer machine or distro override. Do not silently substitute `linux-fslc` or remove the Olimex board-specific U-Boot/LPDDR4 support.

### 3. Create the minimal product image

1. Add a product layer with a minimal headless image recipe and explicit package dependencies. Add OpenSSH server and its service configuration, but no desktop or unrelated administration packages.
2. Package RTKBase, its Python/runtime dependencies, and the required `str2str` executable reproducibly. Avoid downloading or compiling application dependencies on the target at first boot.
3. Add systemd units for the raw receiver/RTCM data stream, GPSD, Chrony, and the RTKBase services actually required by this configuration. Express dependencies and restart behavior explicitly.
4. Configure GPSD for the LG290's actual NMEA-bearing stream, and configure Chrony for the intended GNSS and PPS inputs. Use the confirmed `/dev/ttyACM0` at 460800 baud for the initial prototype; prefer a stable by-id device path if the hardware exposes one.
5. Include kernel options and device permissions for USB ACM serial, PPS, RTC, and required network interfaces. Do not assume `/dev/pps0` exists on the new kernel until verified on the board.
6. Keep receiver configuration (base/rover mode, survey-in, message rates) explicit and reproducible. Avoid a receiver reset/reboot during first boot unless the acceptance procedure accounts for survey restart.
7. Use a dedicated, least-privilege service account and restrict each service's writable paths and device access. Store credentials outside the image; do not bake SSH private keys, NTRIP secrets, or other credentials into the image.

### 4. Build and review on a separate build host

1. Build on a supported Linux workstation/CI host with adequate memory and disk, not on the 3.6 GiB target board.
2. Build the smallest suitable NXP image plus the product image, then inspect the final package manifest and root filesystem.
3. Run Yocto's vulnerability checks where available, review and resolve known CVEs, and retain the generated SPDX SBOM. The Yocto [SBOM documentation](https://docs.yoctoproject.org/current/dev-manual/sbom.html) describes the generated image metadata and its use for supply-chain and vulnerability assessment.
4. Save the build manifest, layer commits, configuration, build logs, image checksums, SBOM, and recovery instructions as release artifacts.
5. Produce a tested update/recovery strategy before production: signed artifacts, authenticated delivery, rollback to a known-good image, and a documented kernel/BSP CVE response cadence.

### 5. Boot and test without replacing the current installation

1. Write the image only to known, separate test media after verifying the target device by size, model, and serial. Do not infer a safe target from Linux device names.
2. Keep the current bootable media unchanged and establish serial-console access before booting the prototype.
3. First boot acceptance:
   - Board model and expected `olimex-imx8mp-evb` machine are identifiable.
   - Kernel reports version 6.18.20 and the expected NXP source/build.
   - The Olimex device tree, DDR, storage, Ethernet, and USB devices initialize without fatal errors.
   - Reboot and power-cycle return to the same image; the original system remains recoverable.
4. Hardware and service acceptance:
   - LG290 enumerates reliably, is accessible at 460800 baud, and retains the intended configuration.
   - GPSD receives valid NMEA/TPV and satellite reports from the configured feed.
   - RTCM output contains expected observation messages with valid CRCs.
   - PPS is present and timestamped correctly if wired; verify kernel PPS events and Chrony PPS association rather than assuming node presence is enough.
   - Chrony receives GNSS samples and selects the intended source under the documented source/fallback policy. Validate UTC and offset against a trusted reference.
   - All required units start in the right order and recover after receiver disconnect/reconnect and system reboot.
   - OpenSSH accepts authorized keys, rejects password/root access as configured, and remains reachable after reboot.
5. Run a sustained soak test (at least 24 hours recommended) and compare timing stability, GNSS fix behavior, RTCM continuity, CPU/memory, and restart behavior with the baseline.

### 6. Migrate only after acceptance

1. Review all test results and resolve regressions before deployment.
2. Create a final verified backup and recovery plan for the current installation.
3. Install the tested image to the production target using the board-appropriate method. Treat bootloader writes as a separate high-risk action and preserve a recovery path.
4. Verify boot, remote access, time service, and RTCM output before declaring migration complete.
5. Track kernel/BSP updates and rebuild/retest the image on a regular security maintenance cadence.

## Open decisions before implementation

- Confirm the exact SOM/carrier-board revision and the board's recovery/boot-media procedure.
- Confirm whether the source layer revision is compatible with the selected NXP `LF6.18.20_2.0.0` manifest without unreviewed patches.
- Resolve and verify the effective kernel provider; the Olimex machine's current kernel default is not by itself proof that `linux-imx` 6.18.20 will be built.
- Decide whether Chrony should prefer GNSS/PPS with network NTP as fallback, or whether NTP remains preferred until a defined GNSS accuracy/fix condition is met.
- Define the update signing, delivery, rollback, and vulnerability-response mechanism.

## References

- [NXP meta-imx: LF6.18.20_2.0.0 release, manifest, and tested boards](https://github.com/nxp-imx/meta-imx)
- [NXP imx-6.18.20-2.0.0 manifest](https://github.com/nxp-imx/imx-manifest/blob/imx-linux-wrynose/imx-6.18.20-2.0.0.xml)
- [NXP linux-imx 6.18 recipe](https://github.com/nxp-imx/meta-imx/blob/master/meta-imx-bsp/recipes-kernel/linux/linux-imx_6.18.bb)
- [Olimex i.MX8MP Yocto machine configuration](https://github.com/Freescale/meta-freescale-3rdparty/blob/master/conf/machine/olimex-imx8mp-evb.conf)
- [Olimex U-Boot recipe and board patch](https://github.com/Freescale/meta-freescale-3rdparty/blob/master/recipes-bsp/u-boot/u-boot-olimex_2024.04.bb)
- [Yocto Project SBOM documentation](https://docs.yoctoproject.org/current/dev-manual/sbom.html)
- [Current system's Armbian upgrade guidance](https://docs.armbian.com/getting-started/updating/) (major distribution upgrades are not the recommended path; prefer a fresh image)
