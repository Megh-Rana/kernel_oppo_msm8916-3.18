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
