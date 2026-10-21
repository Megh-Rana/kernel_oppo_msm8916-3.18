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

---

## Phase 6: Deep Root Cause Analysis & Test 3 Fixes

- **Symptom**: Phone froze at "OPPO" splash screen with Test 2.
- **Root Causes Discovered**:
  1. **Corrupted Kernel Header (`CONFIG_EFI=y`)**:
     - The donor CAF 3.18 tree had `CONFIG_EFI` default to `y`.
     - In ARM64 `head.S`, when `CONFIG_EFI` is enabled, the first instruction is `add x13, x18, #0x16` (forming the PE/COFF "MZ" magic signature) instead of `b stext`, and `stext` is padded out to 4096 bytes with PE/COFF header tables.
     - Little Kernel (LK / `aboot`) on MSM8916 does not use UEFI; it performs `scm_elexec_call` expecting standard ARM64 Linux entry format (`b stext` at offset 0, and `stext` at 0x40).
  2. **Entire PMIC / SPMI Subsystem Was Missing**:
     - In Linux 3.10, the Qualcomm PMIC arbiter lived under upstream `CONFIG_SPMI=y` and `CONFIG_SPMI_MSM_PMIC_ARB=y`.
     - In CAF 3.18, Qualcomm moved their driver to `CONFIG_MSM_SPMI=y` and `CONFIG_MSM_SPMI_PMIC_ARB=y`.
     - Because `lineageos_a37f_defconfig` carried the 3.10 symbol names, `CONFIG_MSM_SPMI` was disabled.
     - Consequently, `OF_SPMI` and `MSM_QPNP_INT` were dropped, which silently disabled ALL PM8916 drivers:
       - `CONFIG_REGULATOR_QPNP` (no LDOs or buck regulators!)
       - `CONFIG_GPIO_QPNP_PIN` (no PMIC GPIOs!)
       - `CONFIG_RTC_DRV_QPNP` (no PMIC RTC!)
       - `CONFIG_QPNP_POWER_ON` (no power-on key handling!)
       - `CONFIG_LEDS_QPNP`
     - Without SPMI and PMIC regulators, early SoC power domains failed to initialize and halted boot.
- **Fixes Applied**:
  - Disabled `CONFIG_EFI`, `CONFIG_EFI_STUB`, and `CONFIG_EFI_ARMSTUB` in `lineageos_a37f_defconfig`.
  - Verified hex layout of `arch/arm64/boot/Image`: byte 0 is now `10 00 00 14` (`b stext`) with `ARM\x64` at offset 0x38, exactly matching stock 3.10.
  - Enabled `CONFIG_MSM_SPMI=y`, `CONFIG_MSM_SPMI_PMIC_ARB=y`, `CONFIG_MSM_SPMI_DEBUGFS_RO=y`, and `CONFIG_MSM_QPNP_INT=y`.
  - Disabled conflicting upstream `CONFIG_SPMI`.
  - Added `static inline` specifiers to stub functions in `include/linux/qpnp/power-on.h`.
  - Successfully recompiled `Image` (with all PMIC regulators, SPMI, and ARMv8 deprecation emulation linked).
  - Regenerated master `dt.img` with `dtbToolOppo` matching chipsets 206, 248, 249, 250 with `oppoId: 15399`.
  - Packaged `kernel-3.18-a37f-test3.zip` and built `boot-3.18-test3.img`.
  - Pushed both to device `/sdcard/`.
  - Pushed `kernel-3.18-a37f-test2.zip` directly to device `/sdcard/kernel-3.18-a37f-test2.zip` via ADB.
  - Committed and pushed to GitHub: commit `42f51fdacb9`.

---

## Phase 7: Display Driver Hardware Diagnostics & Test 4 Fixes

- **Symptom**: Phone froze at "OPPO" splash screen with Test 3.
- **Root Causes Discovered**:
  1. **TI TPS65132 Power Regulator Missing from 3.18 Tree**:
     - In the device tree (`msm8916-qrd-skuh.dtsi`), the display panel (`qcom,mdss_dsi_oppo15399boe_ili9881c_720p_video`) requires `vsp-supply = <&tps65132_pos>` and `vsn-supply = <&tps65132_neg>` (LCD panel positive and negative bias rails).
     - In the reference 3.10 tree, `CONFIG_REGULATOR_TPS65132=y` provided `drivers/regulator/tps65132-regulator.c`.
     - In the donor 3.18 tree, `tps65132-regulator.c` was completely missing.
     - Without this driver, the MDSS DSI panel regulator probe deferred indefinitely (`-EPROBE_DEFER`), preventing the display driver and framebuffer from initializing.
  2. **Qualcomm 3.10 TWRP Ramoops DT Node Omission**:
     - TWRP recovery kernel failed `ramoops_probe` (`ramoops: The memory size and the record/console size must be non-zero`) because Qualcomm's 3.10 kernel discards command-line module parameters if `pdev->dev.of_node` is null.
     - Fixed TWRP recovery by decompiling `recovery.img`'s DTB, injecting the `ramoops` DT node (`compatible = "ramoops"` at `0x9ff00000`, 4MB), recompiling with `dtbToolOppo`, and flashing `recovery_ramoops_v2.img`.
  3. **Missing Serial HSL Console & Android Statistics**:
     - Cmdline specifies `console=ttyHSL0,115200,n8`. In 3.18, `CONFIG_SERIAL_MSM_HSL` was not set, so console messages were discarded.
     - Enabled `CONFIG_SERIAL_MSM_HSL=y` and `CONFIG_SERIAL_MSM_HSL_CONSOLE=y`.
     - Enabled `CONFIG_DEVMEM=y` without `STRICT_DEVMEM` to allow inspecting physical memory.
     - Ported `drivers/misc/uid_cputime.c` from 3.10 and enabled `CONFIG_UID_CPUTIME=y` and `CONFIG_UID_SYS_STATS=y` for Android userspace.
- **Fixes Applied**:
  - Ported `drivers/regulator/tps65132-regulator.c` into 3.18, added Kconfig and Makefile targets.
  - Enabled `CONFIG_REGULATOR_TPS65132=y` in `lineageos_a37f_defconfig`.
  - Ported `drivers/misc/uid_cputime.c`, enabled `CONFIG_UID_CPUTIME=y` and `CONFIG_UID_SYS_STATS=y`.
  - Enabled `CONFIG_DEVMEM=y`, `CONFIG_SERIAL_MSM_HSL=y`, `CONFIG_SERIAL_MSM_HSL_CONSOLE=y`.
  - Cleanly rebuilt kernel `Image` (17.94MB) and master `dt.img` (206KB).
  - Built standalone `boot-3.18-test4.img` and AnyKernel3 flashable zip `kernel-3.18-a37f-test4.zip`.
  - Flashed `recovery_ramoops_v2.img` to `/dev/block/bootdevice/by-name/recovery`.
  - Pushed `boot-3.18-test4.img` and `kernel-3.18-a37f-test4.zip` to device `/sdcard/`.

---

## Phase 8: Ramoops Crash Analysis, TLMM Pinctrl & Camera Fixes (Test 5)

- **Diagnosis Breakthrough**:
  - `recovery_ramoops_v2.img` booted with working DT ramoops and captured `dmesg-ramoops-0` (80KB) and `dmesg-ramoops-1` (110KB) up to `t = 0.549s`.
  - Uncompressed with zlib to reveal two major blockers:
- **Root Cause 1: Fatal Kernel Panic in Camera (`cam_ahb_clk_init`)**:
  - At `t = 0.528s`, `msm_cam@1800000` probed `cam_ahb_clk_init`. Because the DT node lacked `bus-vectors`, `of_property_count_strings` returned `-EINVAL` (`-22`).
  - Driver bug: Checked `if (cnt == 0)` instead of `if (cnt <= 0)`. The `-22` was treated as an unsigned integer, prompting `devm_kzalloc` to request an astronomical buffer size. This triggered `kmalloc_order` / `alloc_pages` failure (`-ENOMEM`), probe teardown freed `mdev`, and subsequent `csiphy_probe` dereferenced the NULL `mdev` pointer in `media_device_register_entity`.
  - **Fix**: Updated `cam_hw_ops.c` to check `if (data.cnt <= 0)` and `if (cnt <= 0)`. Added safety NULL guards in `msm.c`.
- **Root Cause 2: Complete Pinctrl Registration Failure**:
  - At `t = 0.252s`, TLMM driver failed with:
    ```
    pinctrl core: pin 0 already registered on 1000000.pinctrl
    msm-tlmm-pinctrl 1000000.pinctrl: error during pin registration
    msm-tlmm-pinctrl 1000000.pinctrl: Couldn't register pinctrl driver
    ```
  - Upstream 3.18 replaced `pinctrl-msm.c` with the new Linaro driver expecting `struct msm_pinctrl_soc_data`, while `pinctrl-msm-tlmm.c` (MSM8916 TLMM) called `msm_pinctrl_probe` passing `struct msm_tlmm_desc`.
  - This symbol collision passed wrong struct offsets to `pinctrl_register`, failing pin registration.
  - Consequently, **every peripheral** (MDSS display, USB OTG, I2C, SPI, UART, WiFi) deferred probe (`-EPROBE_DEFER`) waiting for `1000000.pinctrl`.
  - **Fix**: Ported 3.10's true TLMM core probe into `drivers/pinctrl/qcom/pinctrl-msm-tlmm-gp.c`, updated to 3.18 pinctrl ops (`set_mux` and multi-config `pin_config_set`), renamed to `msm_tlmm_pinctrl_probe` to resolve symbol clash, and hooked into `pinctrl-msm-tlmm.c` and `Makefile`.
- **Artifacts Produced**:
  - Built `Image` (17.95MB) and master `dt.img` (206KB).
  - Built `boot-3.18-test5.img` and `kernel-3.18-a37f-test5.zip`.
  - Pushed both to device `/sdcard/`.

---

## Phase 9: Display Clock Probe Deferral & TWRP Simple-Bus Fix (Test 6)

- **Diagnosis Breakthrough 1: Simple-Bus `soc.0` Enumeration Trap**:
  - In `drivers/of/platform.c:of_device_make_bus_id`, DT nodes without `reg` under root `/` increment an atomic counter:
    `magic = atomic_add_return(1, &bus_no_reg_magic);`
    `dev_set_name(dev, "%s.%d", node->name, magic - 1);`
  - When `ramoops` lacked `reg`, it took index `0` and bumped `/soc` to `/sys/devices/soc.1`!
  - Android 10 userspace `init.qcom.rc` and TWRP `init.recovery.qcom.rc` both run `wait /dev/block/platform/soc.0/${ro.boot.bootdevice}`. Because it became `soc.1`, the symlink broke, `/dev/block/bootdevice/by-name/` was missing, and partition mounting failed (`Actual block device: ''`).
  - **Fix**: Added `reg = <0x0 0x9ff00000 0x0 0x00400000>;` and named `ramoops: ramoops@9ff00000` in both recovery DTB and `msm8916-common-15399.dtsi`.
- **Diagnosis Breakthrough 2: MDSS MDP Driver Swallowed `-EPROBE_DEFER`**:
  - In `mdss_mdp_irq_clk_setup()` and `mdss_mdp_res_init()`, if `bus_clk` (`clk_gcc_mdss_axi_clk`) or GDSC regulator was not yet registered by `clock_gcc`, `devm_clk_get` returned `-EPROBE_DEFER` (-517).
  - The driver hardcoded `return -EINVAL;` (-22) and left `mdata->res_init = true`.
  - This destroyed `-EPROBE_DEFER`, causing the driver core to drop MDSS MDP from the deferred probe list forever (`probe failed with error -22`). The display never initialized and remained on the bootloader splash screen.
  - **Fix**: Propagated actual return codes from `devm_clk_get` and `devm_regulator_get` in `mdss_mdp.c`, and reset `mdata->res_init = false` on error so deferred probe can succeed.
- **Ramoops Robustness**:
  - Updated `fs/pstore/ram.c` to skip `ramoops_register_dummy()` when a DT node exists, preventing zeroed cmdline arguments from hijacking ramoops.
  - Added hung task detection (`CONFIG_DETECT_HUNG_TASK=y`, `CONFIG_DEFAULT_HUNG_TASK_TIMEOUT=30`, `CONFIG_BOOTPARAM_HUNG_TASK_PANIC=y`).
- **Artifacts Produced**:
  - Built `Image` (17.92MB) and master `dt.img` (206KB).
  - Built `boot-3.18-test6.img` and `kernel-3.18-a37f-test6.zip`.
  - Flashed `recovery_ramoops_v3.img` to recovery.
  - Pushed `boot-3.18-test6.img` and `kernel-3.18-a37f-test6.zip` to device `/sdcard/`.
