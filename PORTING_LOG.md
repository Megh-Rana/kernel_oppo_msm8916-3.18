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
