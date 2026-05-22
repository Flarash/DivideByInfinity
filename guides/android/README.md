# Android

Hands-on guides for **rooting an Android phone** and turning it into a useful
piece of kit — currently focused on Samsung Galaxy S25 Ultra (international
Exynos, SM-S938B), but the post-root half generalises to most rooted Androids.

> [!CAUTION]
> Rooting voids your warranty, trips Knox, breaks banking apps and Samsung
> Pay, and can hard-brick a phone if you skip steps. The pentest content
> additionally requires **written authorization** for every target you touch.
> Your phone, your responsibility.

## Guides

- [**Rooting Samsung Galaxy S25 Ultra (SM-S938B)** on One UI 7 via firmware downgrade + Magisk](./root-samsung-s25-ultra.md) — end-to-end. ADB setup, debloat, One UI 7 firmware downgrade to re-enable OEM Unlock, bootloader unlock, Magisk root, Play Integrity bypass (Shamiko + PIF + TrickyStore). International Exynos model only.
- [**Building a phone-based pentest environment on a rooted Android**](./android-pentest-environment-on-rooted-phone.md) — what to do with the rooted phone afterwards. Termux + Kali (`proot-distro` chroot) + XFCE over VNC + Frida. Headless-drive via `adb shell run-as com.termux`, the gotcha table, smoke test.

The pentest guide assumes the rooting guide (or equivalent) is done. Read in
order if you want the full path from stock phone to working pentest rig.
