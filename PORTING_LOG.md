# MSM8916 Linux 3.18 Porting Log

Target Device: Oppo A37 (A37f / 15399)
Target SoC: Qualcomm Snapdragon 410 (MSM8916)
Donor Kernel: `kernel.lnx.3.18.r34-rel` (tag `LA.UM.7.6.r1-05500-89xx.0`, Linux 3.18.140)
Base Reference: LineageOS Oppo MSM8939 3.10 tree (`kernel_oppo_msm8939`, Linux 3.10.108)

---

## Phase 0: Discovery Findings & Strategy

- **Strategy Selected**: Option A (forward-porting platform DTS and target glue onto donor 3.18).
- **SoC Presence in 3.18 Donor**:
  - `drivers/clk/msm/clock-gcc-8916.c` and `clock-rpm-8916.c` are physically present in the donor tree (`clock-rpm-8916.c` is identical to 3.10).
  - `drivers/clk/msm/clock-cpu-8939.c` is already built under `CONFIG_ARCH_MSM8916`.
  - `drivers/soc/qcom/socinfo.c` already registers MSM8916 IDs (206, 247, 248, 249, 250).
  - Thermal TSENS (`qcom,msm8916-tsens`) is already implemented in `drivers/thermal/msm-tsens.c`.
  - USB: ChipIdea HSUSB (`ci13xxx_msm` + `msm_otg`) is fully supported in `drivers/usb/gadget/` and `drivers/usb/phy/`.
  - GPU: Adreno 306 (`a3xx`) support is present in `drivers/gpu/msm/`.
  - Display: MDSS 28nm LPM DSI PLL (`mdss-dsi-pll-28lpm.c`) is present.
  - Power: QPNP VM-BMS and linear charger drivers are present in `drivers/power/`.
- **Target Device Facts**:
  - Kernel Architecture: 64-bit ARM64 (`aarch64`)
  - Userspace: 32-bit ARM (`armv8-a` / `armeabi-v7a`) with 64-bit binder
  - Bootloader (LK) DT table format: QCDT v3 with custom `oppoId = 15399` (`dtbToolOppo`)
  - Offsets: Base `0x80000000`, Kernel offset `0x00008000`, Ramdisk offset `0x01000000`, Tags offset `0x00000100`, Page size `2048`
  - Ramoops / pstore: Base `0x9ff00000`, Size `0x400000` (4MB), Record size `0x40000` (256KB)
  - Console UART: `earlycon console=ttyHSL0,115200,n8 earlyprintk`
  - Toolchain: Android GCC 4.9 (`aarch64-linux-android-4.9` pie-release)

---

## Phase 1: Toolchain & Build System Setup

- Toolchain: `/home/megh/projects/upstream/tc-aarch64-4.9/bin/aarch64-linux-android-` verified working.
- Host `dtc`: Fixed multiple definition of `yylloc` for GCC 10+ in `scripts/dtc/dtc-lexer.l` and `dtc-lexer.lex.c_shipped`.
- Python 3 compatibility: Removed obsolete `gcc-wrapper.py` invocations in top-level `Makefile` to avoid breaking builds on modern host Python.
- Built `dtbToolOppo` binary from `device_oppo_A37/dtbtool/dtbtool.c` to produce Oppo-specific QCDT v3 master DTBs.

---

## Phase 2: Device Tree & Platform Glue

- Imported MSM8916 SoC DTSI files and Oppo 15399 board DTSI files into `arch/arm/boot/dts/qcom/` (symlinked from `arch/arm64/boot/dts/qcom`).
- Fixed duplicate label conflict in `dsi-panel-jdi-1080p-video.dtsi`.
- Added `msm8916-mtp-15399.dtb` target to `arch/arm/boot/dts/qcom/Makefile`.
- Verified DTC compilation of `msm8916-mtp-15399.dtb` and generated 206KB `dt.img` with chipsets 206, 248, 249, 250 with `oppoId: 15399`.

---

## Phase 3: Driver Porting & Subsystem Integration

1. **Clocks**:
   - Enabled `clock-gcc-8916.o` and `clock-rpm-8916.o` under `CONFIG_ARCH_MSM8916` in `drivers/clk/msm/Makefile`.
   - Wired `MDSS_PLL_TARGET_8916` into `drivers/clk/msm/mdss/` (`mdss-pll.h`, `mdss-pll.c`, `mdss-dsi-pll-28lpm.c`) for `qcom,mdss_dsi_pll_8916`.
2. **Pinctrl**:
   - Ported `pinctrl-msm-tlmm.c` and `pinctrl-msm-tlmm.h` into `drivers/pinctrl/qcom/` to preserve 3.10 DT pin configuration syntax.
   - Selected `PINCTRL_MSM` in `PINCTRL_MSM_TLMM` to link `msm_pinctrl_probe`.
3. **Power / Charging / BMS**:
   - Ported `drivers/power/oppo/` (`oppo_battery.c`, `oppo_bms.c`, `oppo_adc.c`, `oppo_bq27541.c`).
   - Added Oppo power supply properties (`POWER_SUPPLY_PROP_CHARGE_TECHNOLOGY`, `FAST_CHARGE`, `AUTHENTIC`, `BATTERY_CHARGING_ENABLED`) to `include/linux/power_supply.h` and `drivers/power/power_supply_sysfs.c`.
   - Updated `oppo_bq27541.c` to use Linux 3.18 `idr_alloc()` API instead of obsolete `__idr_pre_get()`.
   - Added `the_chip`, `opchg_set_pmic_soc_memory`, `opchg_get_pmic_soc_memory`, and `opchg_get_charger_inout` to `drivers/power/qpnp-linear-charger.c`.
   - Added `opchg_backup_ocv_soc` to `drivers/power/qpnp-vm-bms.c`.
4. **Oppo SoC Platform**:
   - Ported `drivers/soc/oppo/` (`boot_mode.c`, `device_info.c`, `oppo_project.c`) and `include/soc/oppo/`.
5. **Touchscreen**:
   - Ported `drivers/input/touchscreen/synaptics_oppo/` (`synaptics_oppo_driver_3508.c`, `synaptics_oppo_driver_3203.c`, `synaptics_redremote.c`).
   - Added Oppo gesture key definitions (`KEY_GESTURE_CIRCLE`, `KEY_GESTURE_SWIPE_UP`, etc.) to `include/uapi/linux/input-event-codes.h`.
6. **Crypto & SPMI**:
   - Enabled `CONFIG_MSM_SPMI=y` and `CONFIG_MSM_SPMI_PMIC_ARB=y`.
   - Enabled `CONFIG_CRYPTO_DEV_QCOM_MSM_QCE=y` and `CONFIG_CRYPTO_DEV_QCEDEV=y` for hardware crypto engine acceleration in `dm-req-crypt`.
   - Disabled obsolete enterprise `CONFIG_PFT` (Per-File Tagger).
7. **USB**:
   - Added gadget include path `-I$(srctree)/drivers/usb/gadget/` to `drivers/usb/gadget/Makefile` for `u_f.h` inclusion.
8. **Audio**:
   - Added `qcom,msm8x16-audio-codec` compatibility match in `sound/soc/msm/msm8952.c`.

---

## Phase 4: Build & Packaging

- **Kernel Image**: Successfully compiled `arch/arm64/boot/Image` (18MB, 64-bit ARM64 kernel).
- **Master DTB**: Successfully built `dt.img` (206KB, QCDT v3 with oppoId 15399).
- **Packaging**: Created AnyKernel3 flashable zip `kernel-3.18-a37f-test1.zip` in `/home/megh/projects/upstream/` which preserves userspace ramdisk and injects permissive SELinux cmdline.

---

## Phase 5: Debugging Splash Hang & Test 2 Fixes

- **Symptom**: Phone booted to bootloader splash screen and froze without ADB.
- **Analysis**:
  - Extracted stock `boot.img` from ROM `lineage-rev-17.1-20260826-UNOFFICIAL-A37.zip`.
  - Discovered userspace ramdisk uses a 32-bit ARM (`armeabi-v7a`) `init` binary.
  - In 64-bit kernels, 32-bit ARMv7 binaries execute CP15 barrier and SWP instructions. Without kernel-level emulation, ARMv8 Cortex-A53 raises `SIGILL` on the first instruction, immediately killing `init`.
- **Fixes Applied**:
  - Enabled `CONFIG_ARMV8_DEPRECATED=y`, `CONFIG_SWP_EMULATION=y`, `CONFIG_CP15_BARRIER_EMULATION=y`, and `CONFIG_SETEND_EMULATION=y`.
  - Backported OF device tree support into `fs/pstore/ram.c` from 3.10 and added the `ramoops` DT node to `arch/arm/boot/dts/qcom/15399/msm8916-common-15399.dtsi` at `0x9ff00000` (4MB).
  - Rebuilt kernel `Image` and master `dt.img`.
  - Built standalone `boot-3.18-test2.img` (QCDT v0 format with original ramdisk) and flashable zip `kernel-3.18-a37f-test2.zip`.
  - Pushed `kernel-3.18-a37f-test2.zip` directly to device `/sdcard/kernel-3.18-a37f-test2.zip` via ADB.
  - Committed and pushed to GitHub: commit `42f51fdacb9`.

---

## Phase 6: Diagnostic Breakthroughs & Test 3 - Test 7 Fixes

### 1. Camera Panic (`cam_ahb_clk_init`) Fixed (Test 5):
- `of_property_count_strings()` returned `-22` (`-EINVAL`) when `bus-vectors` was absent in device tree.
- The driver treated `-22` as an unsigned count, allocating invalid memory and causing a NULL pointer dereference in `media_device_register_entity`.
- **Fix**: Updated `cam_hw_ops.c` to check `if (data.cnt <= 0)` and added NULL safety guards in `msm.c`.

### 2. TLMM Pinctrl Core Restored (Test 5):
- In 3.18, a symbol collision caused `pinctrl-msm-tlmm.c` to call the wrong `msm_pinctrl_probe()` from the generic Linaro driver, corrupting pin descriptor pointers.
- As a result, `1000000.pinctrl` failed at `t = 0.252s`, causing every hardware peripheral (Display, USB, I2C, SPI, UART, WiFi) to defer probe indefinitely.
- **Fix**: Resolved symbol linkage so the Qualcomm MSM TLMM driver initializes cleanly.

### 3. TWRP Recovery Partition Auto-Mount Fixed (`TWRP_RAMOOPS_FIX.md`):
- Recovered full ramoops and TWRP capability on device (`recovery_ramoops_v3.img`).
- Fixed `soc.0` simple-bus enumeration trap so `/data`, `/system_root`, and `/cache` mount with zero errors across reboots.

### 4. Root Cause of Test 6 Reboot at Splash Screen:
- **Crash Log Diagnostic**: `console-ramoops-0` captured Test 6 executing up to `t = 0.434s`:
  - `Unable to handle kernel paging request at virtual address fffffffffffffdfb`
  - `PC is at devinfo_probe+0xec/0x664 (drivers/soc/oppo/device_info.c)`
- **Mechanism**:
  - `0xfffffffffffffdfb` is `-517` (`-EPROBE_DEFER`) sign-extended in 64-bit.
  - When SMEM probe has not completed, `smem_alloc()` returns `ERR_PTR(-EPROBE_DEFER)`.
  - `drivers/soc/oppo/device_info.c` and `oppo_project.c` checked `if (p)` instead of `IS_ERR_OR_NULL(p)`. Dereferencing `*p` caused an instant CPU panic.
  - SMEM itself failed during probe at `t = 0.220s`:
    `ioremap_nocache()` failed for `0x863ff0e0` and `0x86300000` because on 64-bit ARM, `ioremap_nocache()` refuses to map memory considered valid RAM (`pfn_valid()` check).
  - MSM8916 device tree had the legacy 3.10 `memory { ... }` node which Linux 3.18 ignores, leaving the SMEM physical range inside standard allocatable system RAM.

### 5. Fixes Applied in Test 7 (`2ca552a241a`):
- **Device Tree Memory Carveout**:
  - Converted `msm8916-memory.dtsi` and `15399/msm8916-common-15399.dtsi` to standard `/reserved-memory` with `compatible = "removed-dma-pool"` and `no-map;`.
  - Regions removed from system RAM:
    - `external_image_mem`: `0x86000000` (8MB, contains SMEM `0x86300000` and target info `0x863ff0e0`)
    - `nvbackup_regions`: `0x85f00000` (1MB)
    - `modem_adsp_mem`: `0x86800000` (85MB)
    - `peripheral_mem`: `0x8bd00000` (6MB)
    - `pstore_reserve_mem`: `0x9ff00000` (4MB)
  - With `no-map;`, `memblock_remove` is called, ensuring `pfn_valid()` returns false and `ioremap_nocache()` succeeds without warnings.
- **ION Memory Region Phandles**:
  - Updated `msm8916-ion.dtsi` to supply `memory-region = <...>` phandles matching Linux 3.18 `msm_ion.c` requirements.
- **SMEM Pointer Safety**:
  - `drivers/soc/oppo/device_info.c`: Added `IS_ERR(p)` check in `dram_type_add()`; `devinfo_probe` now returns `-EPROBE_DEFER` cleanly when SMEM is deferred.
  - `drivers/soc/oppo/oppo_project.c`: Wrapped `smem_alloc()` with `IS_ERR_OR_NULL()` and guarded `get_project()`, `get_PCB_Version()`, `get_Modem_Version()`, `get_Operator_Version()`.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` and DTB.
  - Built master `dt_test7.img` (QCDT v2 with oppoId 15399).
  - Packaged `boot-3.18-test7.img` (19MB) and `kernel-3.18-a37f-test7.zip`.
  - Pushed to `/sdcard/` and flashed to `boot` partition via `dd`.

---

## Phase 7: ION DMA_CMA Crash Fix (Test 8 - `dd76db4d9ed`)

- **Diagnosis**:
  - `console-ramoops-0` confirmed Test 7 reached `t = 4.862s` before crashing.
  - SMEM initialization succeeded, `devinfo_probe` succeeded, clocks, thermal, cpufreq, and battery all probed.
  - Fatal crash at `t = 4.862s`:
    `Unable to handle kernel NULL pointer dereference at virtual address 00000000`
    `PC is at cma_get_base+0x0/0xc` (called from `msm_ion_probe+0x4f0`)
  - **Mechanism**:
    - `lineageos_a37f_defconfig` lacked `CONFIG_DMA_CMA=y`.
    - When `CONFIG_DMA_CMA` is disabled, `include/linux/dma-contiguous.h` defines a stub `dev_get_cma_area(dev)` returning `NULL`.
    - In `drivers/staging/android/ion/msm/msm_ion.c`, `msm_ion_get_heap_dt_data()` parsed the dynamic CMA pools (`venus_qseecom_mem` and `audio_mem`), found no static `reg` property, and called `cma_get_base(dev_get_cma_area(heap->priv))`.
    - Passing `NULL` into `cma_get_base()` dereferenced `cma->base_pfn` at `0x00000000`, crashing CPU 1.
- **Fixes Applied**:
  - Enabled `CONFIG_DMA_CMA=y` and `CONFIG_CMA_SIZE_MBYTES=16` in `lineageos_a37f_defconfig`.
  - Added NULL safety checks in `drivers/staging/android/ion/msm/msm_ion.c` to prevent crashes if `dev_get_cma_area()` ever returns NULL.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and `dt_test8.img`.
  - Built `boot-3.18-test8.img` and `kernel-3.18-a37f-test8.zip`.
  - Flashed `boot-3.18-test8.img` to `boot` partition (md5sum `6b59b2221f4a3c73aad1da21208c9d88` verified).

---

## Phase 8: Modem PIL Voltage Property & NULL Safety Fix (Test 9 - `374b5e80312`)

- **Diagnosis**:
  - `console-ramoops-0` confirmed Test 8 booted past ION to `t = 4.940s`.
  - Crash occurred during modem Hexagon PIL subsystem initialization:
    `pil-q6v5-mss 4080000.qcom,mss: Missing vdd_cx-voltage property`
    `Unable to handle kernel NULL pointer dereference at virtual address 00000139`
    `PC is at pil_mss_driver_probe+0x13c/0x524`
  - **Mechanism**:
    - `arch/arm/boot/dts/qcom/msm8916.dtsi` node `qcom,mss@4080000` was missing `vdd_cx-voltage = <7>;`.
    - In `drivers/soc/qcom/pil-q6v5.c`, `return ERR_CAST(prop)` returned `NULL` instead of `ERR_PTR(-EINVAL)`.
    - `pil_mss_driver_probe` checked `if (IS_ERR_OR_NULL(q6)) return PTR_ERR(q6);`. Since `PTR_ERR(NULL)` is 0, probe continued and dereferenced `drv->q6` at `+0x139`.
- **Fixes Applied**:
  - Added `vdd_cx-voltage = <7>;` (`RPM_SMD_REGULATOR_LEVEL_TURBO`) to `qcom,mss@4080000` in `arch/arm/boot/dts/qcom/msm8916.dtsi`.
  - Fixed `pil-q6v5.c` to return `ERR_PTR(-EINVAL)`.
  - Added defensive NULL checking in `pil-q6v5-mss.c`.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and `dt_test9.img`.
  - Packaged `boot-3.18-test9.img` and `kernel-3.18-a37f-test9.zip`.
  - Flashed `boot-3.18-test9.img` to `boot` partition (md5sum `fccc874379920ee7e0f1f3b265132202` verified).


---

## Phase 9: Android 10 First Stage Mount Path, PS_HOLD Reboot, and MDSS DSI Hierarchy (Test 10 - `ce0dc0712ee`)

- **Diagnosis (from Test 9 ramoops)**:
  - Linux 3.18 kernel booted completely with **zero kernel panics or crashes** and reached userspace at `t = 10.916s` (`init: init first stage started!`).
  - Android 10 first stage init failed early mounting `/system`:
    ```
    init: [libfs_mgr]ReadDefaultFstab(): failed to find device default fstab
    init: [libfs_mgr]Failed to open '/dev/block/platform/soc.0/7824900.sdhci/by-name/system': No such file or directory
    init: [libfs_mgr]__mount(source=/dev/block/platform/soc.0/7824900.sdhci/by-name/system(missing),target=/system,type=ext4)=-1: No such file or directory
    init: Failed to mount /system: No such file or directory
    init: Failed to mount required partitions early ...
    reboot: Restarting system with command 'bootloader'
    Reboot failed -- System halted
    ```
  - **Root Causes**:
    1. **Fstab Path**: In Linux 3.10, root buses used `bus_no_reg_magic` atomic counter, creating `/dev/block/platform/soc.0/`. In Linux 3.18, upstream removed `bus_no_reg_magic`, making the bus name `/dev/block/platform/soc/`. Because `msm8916.dtsi` specified `soc.0`, `first_stage_mount` failed to find the required block device.
    2. **PS_HOLD Reboot Failure**: `restart@4ab000` node in `msm8916.dtsi` lacked `reg-names = "pshold-base", "tcsr-boot-misc-detect";`, preventing `drivers/power/reset/msm-poweroff.c` from mapping the `ps_hold` register and registering `arm_pm_restart`. As a result, when userspace invoked `reboot()`, restart failed and printed `Reboot failed -- System halted`.
    3. **MDSS DSI Clocks**: `clock_gcc_mdss` referenced `clk_pixel_clk_src` and `clk_byte_clk_src` instead of `clk_dsi_pll0_pixel_clk_src` and `clk_dsi_pll0_byte_clk_src`, failing probe with `Failed to get pixel source (-2)`.
    4. **MDSS DSI Two-Tier Driver Hierarchy**: In CAF 3.18, the MDSS DSI driver was refactored into a parent resource driver (`qcom,mdss-dsi`) and child controller (`qcom,mdss-dsi-ctrl`). Without the parent `mdss_dsi` node, `mdss_dsi_res` was NULL and child probe failed with `mdss_dsi_ctrl_probe: Unable to get the ctrl_pdata (-22)`.
- **Fixes Applied**:
  - In `arch/arm/boot/dts/qcom/msm8916.dtsi`:
    - Changed `firmware/android/fstab/system` device path to `/dev/block/platform/soc/7824900.sdhci/by-name/system`.
    - Added `reg-names = "pshold-base", "tcsr-boot-misc-detect";` to `restart@4ab000`.
    - Updated `clock_gcc_mdss` to use `clk_dsi_pll0_pixel_clk_src` and `clk_dsi_pll0_byte_clk_src`.
  - In `include/dt-bindings/clock/msm-clocks-8916.h`:
    - Added definitions for `clk_dsi_pll0_byte_clk_src` (`0x44539836`) and `clk_dsi_pll0_pixel_clk_src` (`0x5767c287`).
  - In `arch/arm/boot/dts/qcom/msm8916-mdss.dtsi`:
    - Structured `mdss_dsi: qcom,mdss_dsi@0` (compatible `qcom,mdss-dsi`) as the parent device with GDSC, VDDA, VDDIO supplies, bus scaling, and clocks.
    - Positioned `mdss_dsi0: qcom,mdss_dsi@1a98000` as the child controller with panel supplies and DSI clocks.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and `dt_test10.img`.
  - Built `boot-3.18-test10.img` and `kernel-3.18-a37f-test10.zip`.
  - Pushed to device `/sdcard/` and flashed `boot-3.18-test10.img` to `boot` partition (md5sum `7ff22938a16f5fa557a3486edd1e4dab` verified).


---

## Phase 10: Restore `soc.0` Bus Naming for Legacy Userspace & Fix DSI Escape Clock (Test 11 - `ff9cedf7359`)

- **Diagnosis (from Test 10 ramoops & pmsg)**:
  - **Milestone Reached**:
    - Android 10 first stage init mounted `/system` at `t = 11.309s`:
      `init: [libfs_mgr]__mount(source=/dev/block/platform/soc/7824900.sdhci/by-name/system,target=/system,type=ext4)=0: Success`
    - Init switched root to `/system`, loaded SELinux policy, and launched second stage init at `t = 12.049s` (`init second stage started!`).
    - Launched core services: `logd`, `servicemanager`, `hwservicemanager`, `vndservicemanager`, `vold`.
    - Android kernel ran continuously for over 138 seconds without panics or unexpected reboots.
  - **Two Remaining Issues Discovered**:
    1. **Second-Stage `init.qcom.rc` Mount Timeout**:
       At `t = 25.331s`, second stage init ran `/vendor/etc/init/hw/init.qcom.rc:42`:
       `wait /dev/block/platform/soc.0/${ro.boot.bootdevice}`
       `symlink /dev/block/platform/soc.0/${ro.boot.bootdevice} /dev/block/bootdevice`
       Because the bus was named `soc` (without `.0`), this wait timed out, `/dev/block/bootdevice` was never created, and `mount_all /vendor/etc/fstab.qcom` skipped mounting `/data`, `/cache`, and `/persist`.
    2. **MDSS DSI Escape Clock Provider**:
       In Test 10, DSI controller probe failed at `t = 4.800s`:
       `mdss_dsi_link_clk_init: can't find dsi_esc_clk. rc=-2`
       Because `clk_gcc_mdss_esc0_clk` was incorrectly requested from `&clock_gcc_mdss` rather than `&clock_gcc`.
- **Fixes Applied**:
  - In `arch/arm/boot/dts/qcom/msm8916.dtsi`:
    - Defined root bus node as `soc: soc.0 { };` so the platform bus natively produces `/sys/devices/soc.0/` and `/dev/block/platform/soc.0/`.
    - Updated `firmware/android/fstab/system` device path to `/dev/block/platform/soc.0/7824900.sdhci/by-name/system`.
  - In `arch/arm/boot/dts/qcom/msm8916-mdss.dtsi`:
    - Fixed `clk_gcc_mdss_esc0_clk` provider from `&clock_gcc_mdss` to `&clock_gcc`.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and `dt_test11.img`.
  - Built `boot-3.18-test11.img` and `kernel-3.18-a37f-test11.zip`.
  - Flashed `boot-3.18-test11.img` to `boot` partition (md5sum `1811c56a92d38f96b735f285946ed4df` verified).

---

## Phase 11: Android Firmware Search Paths & KGSL Crash Prevention (Test 12 - `6d0862a8401`)

- **Diagnosis (from Test 11 ramoops & pmsg)**:
  - **Milestone Reached**:
    - Second stage init executed `mount_all /vendor/etc/fstab.qcom` successfully in 1659ms (`/data`, `/cache`, `/persist` mounted cleanly).
    - DSI panel attached and registered `FrameBuffer[0] 720x1280` (`oppo15399boe ili9881c 720p video mode dsi panel`) with continuous splash.
    - Android reached `class_start main` and `class_start late_start`. All major Android services started: `drm`, `keystore`, `storaged`, `qcamerasvr`, `surfaceflinger`.
  - **Panic Mechanism at `t = 28.564s`**:
    - `surfaceflinger` (PID 344) opened `/dev/kgsl-3d0` to initialize Adreno GPU acceleration.
    - `a3xx_microcode_read` called `_load_firmware(device, "a300_pm4.fw", &ptr, &len)`.
    - `request_firmware("a300_pm4.fw")` failed with `-2` (`ENOENT`) because the kernel's built-in `fw_path[]` in `drivers/base/firmware_class.c` only searched desktop Linux paths (`/lib/firmware/...`), omitting Android paths (`/vendor/firmware`, `/system/vendor/firmware`, etc.).
    - `a3xx_microcode_read` called `KGSL_DRV_FATAL(...)`, which expanded to `BUG()` in `kgsl_log.h:48`, crashing the kernel:
      `Kernel BUG at a3xx_microcode_read+0xf0`
      `Kernel panic - not syncing: Fatal exception`
    - PS_HOLD restart executed 5 seconds later, rebooting the phone back to splash.
- **Fixes Applied**:
  - In `drivers/base/firmware_class.c`:
    - Added Android firmware search paths (`/etc/firmware`, `/vendor/firmware`, `/system/vendor/firmware`, `/system/etc/firmware`, `/firmware/image`) directly to the static `fw_path[]` table.
  - In `drivers/gpu/msm/adreno_a3xx.c`:
    - Replaced `KGSL_DRV_FATAL(...)` with `KGSL_DRV_ERR(...)` on microcode read failure so GPU firmware issues log errors but never panic the kernel.
  - In `arch/arm64/configs/lineageos_a37f_defconfig`:
    - Enabled `CONFIG_FW_LOADER_USER_HELPER=y` and `CONFIG_FW_LOADER_USER_HELPER_FALLBACK=y`.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and generated `dt_test12.img`.
  - Built `boot-3.18-test12.img` and `kernel-3.18-a37f-test12.zip`.
  - Flashed `boot-3.18-test12.img` to `boot` partition (md5sum `7d7f7a6783284e8a4e52c275d78c3b53` verified).

---

## Phase 12: KGSL SMMU Device Tree Integration & MMU Null Pagetable Guard (Test 13 - `0ea58c2b9c0`)

- **Diagnosis (from Test 12 ramoops & pmsg)**:
  - **Milestone Reached**:
    - Kernel ran stably for **44.87 seconds**.
    - Microcode loading succeeded seamlessly from `/vendor/firmware/a300_pm4.fw`.
    - Android services initialized: `time_daemon`, `vold`, `apexd`, `hwservicemanager`, `rild`.
    - `SurfaceFlinger` successfully loaded proprietary Adreno OpenGL drivers (`libEGL_adreno.so`, `libGLESv1_CM_adreno.so`, `libGLESv2_adreno.so`) and initialized Adreno EGL (`QUALCOMM Build: 10/21/16`).
  - **Panic Mechanism at `t = 44.87s`**:
    - SurfaceFlinger called `kgsl_ioctl_gpumem_alloc_id` to allocate the primary display surface buffer.
    - In CAF 3.18, Qualcomm refactored KGSL IOMMU binding: KGSL probes a dedicated `compatible = "qcom,kgsl-smmu-v2"` DT node with child context bank `gfx3d_user`. Because `msm8916-gpu.dtsi` lacked this node, `kgsl_iommu_probe` returned `-ENODEV` and KGSL defaulted to `KGSL_MMU_TYPE_NONE` (NoMMU).
    - In NoMMU mode, `entry->memdesc.pagetable` remained `NULL`.
    - `kgsl_mem_entry_attach_process` invoked `kgsl_mmu_map(entry->memdesc.pagetable, &entry->memdesc)`.
    - `kgsl_mmu_map` did not verify `pagetable != NULL`, resulting in `atomic_inc(&pagetable->stats.entries)` dereferencing offset `0x28` (`NULL + 0x28`), causing:
      `Unable to handle kernel NULL pointer dereference at virtual address 00000028`
      `PC is at kgsl_mmu_map+0x50/0xd0`
- **Fixes Applied**:
  - In `arch/arm/boot/dts/qcom/msm8916-gpu.dtsi`:
    - Added `kgsl_msm_iommu: qcom,kgsl-iommu@1f00000` with compatible `qcom,kgsl-smmu-v2`, hardware protection aperture `qcom,protect = <0xa000 0x1000>`, GCC SMMU clocks, and child `gfx3d_user` (`qcom,gpu-offset = <0xa000>`).
  - In `drivers/gpu/msm/kgsl_mmu.c`:
    - Added defensive `if (!pagetable) return -EINVAL;` guards to `kgsl_mmu_map` and `kgsl_mmu_unmap`.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and generated `dt_test13.img`.
  - Built `boot-3.18-test13.img` and `kernel-3.18-a37f-test13.zip`.
  - Pushed to `/sdcard/` and flashed `boot-3.18-test13.img` to `boot` partition (md5sum `759bda931ad34754007de7f271b3c56f` verified).

---

## Phase 13: Unblock MDSS Framebuffer Commit Ioctls & Enable SW Sync Device (Test 14 - `bbdc274f32a`)

- **Diagnosis (from Test 13 ramoops & pmsg)**:
  - **Milestone Reached**:
    - Kernel ran continuously for **over 5 minutes (304 seconds)** without crashing or panicking.
    - KGSL SMMU probe succeeded; Adreno GPU allocated virtual memory without issues.
    - Full Android 10 userspace was alive: SurfaceFlinger and HWComposer (`hwcomposer.msm8916.so`) were actively rendering and attempting to commit display frames at **60 FPS**.
  - **Root Cause of Splash Screen Persistence**:
    - In `pmsg-ramoops-0`: HWComposer logged continuous failures:
      `qdhwcomposer hwc_sync: ioctl MSMFB_BUFFER_SYNC failed, err=Invalid argument`
      `qdoverlay Failed to call ioctl MSMFB_DISPLAY_COMMIT err=Invalid argument`
    - In `console-ramoops-0`: MDSS driver logged:
      `mdss_fb_compat_buf_sync: failed -22`
      `mdss_fb_do_ioctl: Unsupported ioctl`
    - In `drivers/video/msm/mdss/mdss_fb.c:4817`, Qualcomm had introduced `check_not_supported_ioctl(cmd)` which deliberately intercepted and blocked `MSMFB_DISPLAY_COMMIT`, `MSMFB_BUFFER_SYNC`, `MSMFB_OVERLAY_SET`, `MSMFB_OVERLAY_PLAY`, etc., printing `Unsupported ioctl` and returning `-EINVAL`.
    - This blocked the hardware composer from committing frames to the display controller, preventing continuous splash from releasing.
    - In addition, `/dev/sw_sync` was missing because `CONFIG_SW_SYNC_USER` was not enabled in defconfig.
- **Fixes Applied**:
  - In `drivers/video/msm/mdss/mdss_fb.c`:
    - Updated `check_not_supported_ioctl(cmd)` to `return false;`, allowing all standard MDSS framebuffer and overlay ioctls (`MSMFB_DISPLAY_COMMIT`, `MSMFB_BUFFER_SYNC`, `MSMFB_OVERLAY_SET`, etc.) to pass through to their handlers.
  - In `arch/arm64/configs/lineageos_a37f_defconfig`:
    - Enabled `CONFIG_SW_SYNC=y` and `CONFIG_SW_SYNC_USER=y`.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and generated `dt_test14.img`.
  - Built `boot-3.18-test14.img` and `kernel-3.18-a37f-test14.zip`.
  - Pushed to `/sdcard/` and flashed `boot-3.18-test14.img` to `boot` partition (md5sum `7fc3140035bbaef65e42cc2adf332958` verified).





---

## Phase 14: Fix MDSS Overlay Scaler Struct Mismatch & Force Early Boot ADB (Test 15 - `1ae2422a1ac`)

- **Diagnosis (from Test 14 ramoops & pmsg)**:
  - **Milestone Reached**:
    - With `check_not_supported_ioctl` unblocked, `MSMFB_BUFFER_SYNC` and `MSMFB_DISPLAY_COMMIT` succeeded.
    - Software sync fences were successfully created and signaled via `/dev/sw_sync`.
  - **Root Cause of Splash Screen Persistence**:
    - In `console-ramoops-0`: MDSS overlay kickoff aborted on every commit:
      `__mdss_mdp_validate_pxl_extn: roi_w[0]=538976266 src_w=720`
      `mdss_mdp_overlay_kickoff: unable to start overlay 0 (-1)`
    - In CAF 3.18, `pipe->scaler` was upgraded to `struct mdp_scale_data_v2` (which has a 4-byte `uint32_t enable` as its first field).
    - `mdss_mdp_overlay.c:889` executed a raw `memcpy(&pipe->scaler, &req->scale, sizeof(struct mdp_scale_data))`.
    - Because the legacy userspace structure `struct mdp_scale_data` begins with a 1-byte `uint8_t enable_pxl_ext`, this raw copy shifted all subsequent fields by 3 bytes and contaminated `pipe->scaler.enable` with non-zero garbage.
    - This caused `mdss_mdp_overlay_setup_scaling` to mistakenly run `__mdss_mdp_validate_pxl_extn`, which inspected garbage `roi_w` values (`538976266 = 0x2020200A`) and failed overlay kickoff (`-1`).
  - **Root Cause of Missing ADB**:
    - LineageOS 17.1 has `persist.sys.usb.config=none` hardcoded in `/system/default.prop`, disabling ADB completely on boot.
- **Fixes Applied**:
  - In `drivers/video/msm/mdss/mdss_mdp_overlay.c`:
    - Replaced the blind `memcpy` with `memset(&pipe->scaler, 0, sizeof(pipe->scaler));` and field-by-field copies into `mdp_scale_data_v2` only when `req->scale.enable_pxl_ext != 0`.
  - In Ramdisk:
    - Added `/force_debuggable` to the root directory to signal Android 10 first-stage init to force `ro.debuggable=1` and enable ADB root without key verification.
  - In Device `/system/default.prop`:
    - Updated `ro.secure=0`, `ro.adb.secure=0`, and `persist.sys.usb.config=mtp,adb`.
  - In Kernel Cmdline:
    - Added `buildvariant=eng androidboot.mode=eng androidboot.usbconfig=adb`.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and generated `dt_test15.img`.
  - Built `boot-3.18-test15.img` (MD5: `b5e8db1419c449f70e328f58db205860`) and `kernel-3.18-a37f-test15.zip`.
  - Pushed to `/sdcard/` and flashed to `boot` partition (MD5 verified).


---

## Phase 15: Fix MDSS Overlay Compat 32-bit Struct Shift & Scaler Fallback (Test 16 - `c46057c050b`)

- **Diagnosis (from Test 15 ramoops & pmsg)**:
  - **Milestone Reached**:
    - Kernel ran stably for **138.8 seconds (~2.3 minutes)** without panicking.
    - Android userspace initialized key services: `time_daemon`, `vold`, `hwservicemanager`, `hwcomposer`.
    - No kernel panic occurred.
  - **Root Cause of Persistent `__mdss_mdp_validate_pxl_extn: roi_w[0]=538976266`**:
    - In `console-ramoops-0`, kickoff still failed with `roi_w[0] = 538976266 = 0x2020200A` ('\n   ' in ASCII) and `v_fetch = 1750350160 = 0x68545950` ("PYTh" in ASCII).
    - Reverse engineering `hwcomposer.msm8916.so` and `liboverlay.so` with `gdb` and comparing against Oppo 3.10 kernel source revealed the exact root cause:
      - In 3.10 kernel headers, `struct mdp_overlay_pp_params` ended at `hist_lut_cfg` (size = 156 bytes).
      - In 3.18 kernel headers, Qualcomm inserted `pa_v2_cfg_data` and `pcc_cfg_data` into `struct mdp_overlay_pp_params32` (size = 216 bytes, a **60-byte expansion**).
      - The 32-bit HAL (`hwcomposer.msm8916.so`) was compiled against 3.10 headers where `scale` starts at offset `112 + 156 = 268`.
      - But the 64-bit kernel's compat layer (`mdss_compat_utils.h`) had `struct mdp_overlay32` expecting `scale` at offset `112 + 216 = 328`.
      - When handling `MSMFB_OVERLAY_PREPARE` and `MSMFB_OVERLAY_SET`, `copy_in_user(&ov->scale, &ov32->scale, sizeof(struct mdp_scale_data))` read **60 bytes past the end** of the userspace struct, pulling ASCII strings and garbage heap memory into `scale.enable_pxl_ext` and `scale.roi_w`.
      - Because the garbage byte at offset 328 was `0x20` (ASCII space), `req->scale.enable_pxl_ext` became non-zero, tricking the kernel into attempting pixel extension validation with garbage `roi_w` values!
      - Furthermore, in `__mdss_mdp_validate_pxl_extn`, returning `-EINVAL` aborted `mdss_mdp_overlay_kickoff` completely (`-1`), dropping every frame at 60 FPS and locking the display on continuous splash.
- **Fixes Applied**:
  - In `drivers/video/msm/mdss/mdss_compat_utils.h`:
    - Introduced `struct mdp_overlay_pp_params_v1_32` containing only the 3.10 fields (156 bytes).
    - Updated `struct mdp_overlay32` to use `struct mdp_overlay_pp_params_v1_32 overlay_pp_cfg;`, restoring `scale` to its exact userspace offset (268).
  - In `drivers/video/msm/mdss/mdss_compat_utils.c`:
    - Updated `__from_user_pp_params` and `__to_user_pp_params` to accept `struct mdp_overlay_pp_params_v1_32 *`.
  - In `drivers/video/msm/mdss/mdss_mdp_overlay.c`:
    - In `mdss_mdp_overlay_pipe_setup()`: Added sanity check requiring `req->scale.roi_w[0] > 0 && req->scale.roi_w[0] <= pipe->src.w` before enabling `pipe->scaler.enable`.
    - In `mdss_mdp_overlay_setup_scaling()`: If `__mdss_mdp_validate_pxl_extn` fails for any reason, reset `pipe->scaler.enable = 0` and smoothly fall back to standard MDP phase-step scaling instead of failing overlay kickoff with `-EINVAL`.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and generated `dt_test16.img`.
  - Built `boot-3.18-test16.img` (MD5: `2020f9740156e164a2ca5cd85e5e529d`) and `kernel-3.18-a37f-test16.zip`.
  - Pushed to `/sdcard/` and flashed to raw `boot` partition (MD5 verified).

---

## Phase 16: Fix MDSS SMMU Context Bank Binding & NULL Guards (Test 17 - `3dbedeba50e`)

- **Diagnosis (from Test 16 ramoops)**:
  - **Milestone Confirmed**:
    - `__mdss_mdp_validate_pxl_extn` error count: **0 occurrences** (was thousands per second).
    - `unable to start overlay 0` error count: **0 occurrences** (was 60 FPS drops).
    - The overlay kickoff succeeded and passed the frame into `mdss_fb0` worker for display hardware mapping.
  - **Crash Analysis**:
    - At `t = 59.9s`, panic occurred: `Unable to handle kernel NULL pointer dereference at virtual address 00000000` with `PC is at mdss_smmu_map_v2+0x9c/0xb0`.
    - In CAF 3.18, `mdss_smmu_init()` calls `mdss_smmu_device_create()` which loops through child nodes of `mdss_mdp` matching prefix `qcom,smmu_` to create platform devices that trigger `mdss_smmu_probe()`.
    - `msm8916-mdss.dtsi` was missing child nodes `smmu_mdp_unsec: qcom,smmu_mdp_unsec_cb { compatible = "qcom,smmu_mdp_unsec"; };` and `smmu_mdp_sec`.
    - Without child nodes, no platform devices were created, `mdss_smmu_probe()` never ran for MDSS, and `mdss_smmu->mmu_mapping` and `mdss_smmu->dev` remained `NULL`.
    - When `surfaceflinger` attempted buffer attachment, `dma_buf_attach(dmabuf, NULL)` warned at `dma-buf.c:427`.
    - When `mdss_fb0` called `mdss_smmu_map_v2` to map the continuous splash framebuffer memory (`iova=0x83000000, size=0x1400000`), it executed `iommu_map(mdss_smmu->mmu_mapping->domain, ...)` with a `NULL` domain pointer, dereferencing `0x00000000`.
- **Fixes Applied**:
  - In `arch/arm/boot/dts/qcom/msm8916-mdss.dtsi` (and hardlinked arm64 include):
    - Added context bank child nodes `smmu_mdp_unsec: qcom,smmu_mdp_unsec_cb { compatible = "qcom,smmu_mdp_unsec"; };` and `smmu_mdp_sec: qcom,smmu_mdp_sec_cb { compatible = "qcom,smmu_mdp_sec"; };` under `mdss_mdp`.
    - On device probe, these match and attach to `mdp_0` and `mdp_1` context banks defined in `msm-iommu-v2.dtsi`.
  - In `drivers/video/msm/mdss/mdss_smmu.c`:
    - Added defensive `NULL` guards across `mdss_smmu_dma_buf_attach_v2`, `mdss_smmu_map_dma_buf_v2`, `mdss_smmu_unmap_dma_buf_v2`, `mdss_smmu_dma_alloc_coherent_v2`, `mdss_smmu_dma_free_coherent_v2`, `mdss_smmu_map_v2`, `mdss_smmu_unmap_v2`, `mdss_smmu_dsi_map_buffer_v2`, and `mdss_smmu_dsi_unmap_buffer_v2`.
    - Returning `ERR_PTR(-ENODEV)` / `-EINVAL` instead of crashing on uninitialized structures.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and generated `dt_test17.img`.
  - Built `boot-3.18-test17.img` (MD5: `cab048f2cf6e2ecd050b6b99a79b670e`) and `kernel-3.18-a37f-test17.zip`.
  - Pushed to `/sdcard/` and flashed to raw `boot` partition (MD5 verified).

---

## Phase 17: Fix USB OTG Controller Probe & QSEECOM 3.10 32-bit Compat Ioctl Translation (Test 18 - `0222d424c7b`)

- **Diagnosis (from Test 17 ramoops)**:
  - **Milestone Confirmed**:
    - **Zero kernel panics** and **zero kernel oopses**.
    - MDSS SMMU context banks `mdp_0` and `mdp_1` probed and bound successfully (`iommu v2 domain[0] mapping and clk register successful!`).
    - SurfaceFlinger initialized the Adreno 306 GPU in OpenGL ES 3.0 mode (`OpenGL ES 3.0 V@145.0 AU@`).
    - Full Android userspace booted (Zygote, SystemServer, AudioFlinger, MediaServer, RILD, CameraProvider, Netd, Storaged, etc.).
  - **Identified Failure 1 - USB Gadget & ADB**:
    - `console-ramoops-0`: `msm_otg 78d9000.usb: msm_otg probe` failed with error `-22` (`-EINVAL`).
    - In 3.10, `phy-msm-usb.c` did not require `qcom,max-nominal-sysclk-rate`. In 3.18, CAF added a mandatory read of `qcom,max-nominal-sysclk-rate` that returns `-EINVAL` if omitted from device tree.
    - Because `msm_otg` failed probe, USB core never registered a UDC device, preventing `android_usb` from binding and preventing USB enumeration/ADB on the host PC.
  - **Identified Failure 2 - Keymaster / QSEECOM Stall & Rescue Reboot**:
    - `pmsg-ramoops-0` showed `android.hardware.keymaster@3.0-impl: keymaster app is not loaded, attempt number 0` followed by `Loading keymaster app failied`.
    - Keymaster service never registered, causing `vold` and `wait_for_keymaster` to loop every second: `Waited one second for android.hardware.keymaster@3.0::IKeymasterDevice/default`.
    - Disassembly of `libQSEEComAPI.so` (Oppo 32-bit userspace blob) revealed it calls `ioctl(fd, 0xc024970f, ...)` (`QSEECom_app_load_query`).
    - In 3.10, `MAX_APP_NAME_SIZE` was 32 (`sizeof = 36`, ioctl command `0xc024970f`).
    - In 3.18, Qualcomm expanded `MAX_APP_NAME_SIZE` to 64 and added `app_arch` (`sizeof = 72`, ioctl command `0xc048970f`).
    - The 3.18 `compat_qseecom_ioctl` lacked entries for 3.10 legacy ioctls, returning `-ENOIOCTLCMD` (`ENOTTY`), preventing keymaster from loading and causing Android 10 apexd/rescue party reboot after ~60s.
- **Fixes Applied**:
  - In `arch/arm/boot/dts/qcom/msm8916.dtsi` (and arm64 hardlink):
    - Added `qcom,max-nominal-sysclk-rate = <133330000>;` under `usb_otg: usb@78d9000`.
  - In `drivers/usb/phy/phy-msm-usb.c`:
    - Added fallback default (`133330000`) for `qcom,max-nominal-sysclk-rate` so probe does not fail with `-EINVAL` even if missing in DT.
  - In `drivers/misc/compat_qseecom.h`:
    - Defined legacy 3.10 compat structures `struct compat_qseecom_qseos_app_load_query_32` (size 36) and `struct compat_qseecom_load_img_req_32` (size 48).
    - Defined 3.10 ioctl command macros:
      - `COMPAT_QSEECOM_IOCTL_APP_LOADED_QUERY_REQ_32` (`0xc024970f`)
      - `COMPAT_QSEECOM_IOCTL_LOAD_APP_REQ_32` (`0xc0309707`)
      - `COMPAT_QSEECOM_IOCTL_LOAD_EXTERNAL_ELF_REQ_32` (`0xc030970d`)
  - In `drivers/misc/compat_qseecom.c`:
    - Implemented `compat_get_qseecom_load_img_req_32`, `compat_put_qseecom_load_img_req_32`, `compat_get_qseecom_qseos_app_load_query_32`, and `compat_put_qseecom_qseos_app_load_query_32`.
    - Added cases in `convert_cmd` and `compat_qseecom_ioctl` to properly translate and execute these 3.10 ioctls.
- **Packaging & Deployment**:
  - Recompiled kernel `Image` (18MB) and generated `dt_test18.img`.
  - Built `boot-3.18-test18.img` (MD5: `53f11ae90c96d6ce5fc8b0cca74b692f`) and `kernel-3.18-a37f-test18.zip` (MD5: `9a0aad8756eaf308343652c7fbdb4b87`).
  - Pushed to `/sdcard/` and flashed to raw `boot` partition (MD5 verified).

