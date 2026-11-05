# PixelExperience-Reborn Android 17 — Qualcomm GPS/GNSS ownership audit (mondrian)

**2026-11-05 | P2 row 151: `hardware/qcom/gps` | historical source only, NOT an Android17/Taro GNSS implementation.**

## Verified historical source

- PE `seventeen` head `7c9f0251550c3e1d8a227883279b4516d1174a8a`.
- The old root `Android.mk` computes `TARGET_USES_HARDWARE_QCOM_GPS` and, for some configurations, falls through to generic `core utils loc_api platform_lib_abstractions etc` modules. It **does not explicitly exclude `taro`**. Under the reference device's `BOARD_VENDOR_QCOM_GPS_LOC_API_HARDWARE := default` condition, adding the archived tree to a product could therefore create unintended older location modules.
- Historical `core/Android.mk` defines `libloc_core`, depends on `libgps.utils`, `libprocessgroup`, `$(TARGET_OUT_HEADERS)/gps.utils` and `LOCAL_COPY_HEADERS`. The latter is obsolete/unsupported in the modern Soong-first source architecture. This is a *real duplicate module-name risk* if the old tree is included alongside modern Taro GNSS.
- The official AOSP17 base manifest does **not** include `hardware/qcom/gps`; generic framework location APIs do not implement the required Qualcomm modem/vendor GNSS stack.

## Exact modern reference already present in the user's tree

The existing **read-only** `LF52406/mondrian_android_device_xiaomi_sm8450-common` `lineage-24.0` contains a full `gps/` source subtree, not just configs.

- `gps/gps_vendor_board.mk` selects `BOARD_VENDOR_QCOM_GPS_LOC_API_HARDWARE` subject to board/QMAA flags; `BoardConfigCommon.mk` already sets the relevant default.
- `gps/gps_vendor_product.mk` selects `libloc_core`, `libgps.utils`, `liblocation_api`, `libgnss`, `libbatching`, `libgeofencing`, GNSS antenna/config resources, `android.hardware.gnss@2.1-impl-qti`, `android.hardware.gnss-aidl-impl-qti` and `android.hardware.gnss-aidl-service-qti`. Independently, the device `common.mk` already names modern GNSS packages.
- `gps/android/aidl-impl/Android.bp` defines the AIDL GNSS shared implementation and service; it links `android.hardware.gnss-V1-ndk`, `libbinder_ndk`, vendor `libloc_core`/`libgps.utils`/`liblocation_api`, plus older GNSS HIDL interface libraries for compatibility.
- `gps/android/aidl-impl/android.hardware.gnss-aidl-service-qti.rc` registers `interface aidl android.hardware.gnss.IGnss/default`; VINTF declares that instance.
- The current AIDL implementation and binary both specify the same GNSS VINTF fragment in their `Android.bp` blocks. Check the *effective installed VINTF XML* for duplication and identify the correct sole owner; the duplicate references do not by themselves prove a build failure.
- The source references `hardware/interfaces` GNSS AIDL V1 and old HIDL 2.1 together; check actual Android17 interface availability, vendor blob ABI, binder service registration, header lib namespaces and all required .so files. Do **not** assume these compile simply because they worked with another Android base.

## Technical decision and gating

**Never register both the old PE `libloc_core` source and the modern Taro modules in one Soong/Make graph.** Keep this historical PE tree intact as an archival reference; integrate the device-specific AIDL source separately into the new independently owned PE-Reborn Taro graph after review, without importing the entire current LineageOS product.

1. Preserve AOSP17 `frameworks/base`, `frameworks/av`/location and `hardware/interfaces` interfaces and security behavior.
2. Unify GNSS module owners for `libloc_core`, `libgps.utils`, `liblocation_api`, `libgnss`, AIDL service and any HIDL compatibility libs; no duplicate Soong module names or two default IGnss instances.
3. Reconcile vendor and modem assistance/config files, `gps.conf`, XTRA/AGPS, permissions, init RC, VINTF and SELinux enforcing; verify network/network-location and emergency/location behavior.
4. Compile all required GNSS libs/headers and the service in full AOSP17 Soong, run `checkvintf`, then exercise GNSS cold/hot fix, permissions, background tracking, satellite measurements, assisted GNSS and long-running stability on physical mondrian.

**Status:** actual modern source owner identified; archived PE module must remain disabled. AOSP17 GNSS Soong/APEX/CTS-VTS and device runtime **not yet tested**. No changes to the working device tree.

**Next row:** `hardware/qcom/media`.
