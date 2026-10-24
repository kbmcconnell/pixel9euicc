# GrapheneOS: Fix broken eSIM (eUICC) on a Pixel 9 (tokay)

I was trying to install GrapheneOS on a Pixel 9 when I had trouble activating the eSIM. I came across these two resources that helped get me through fixing the issue. I've taken what I learned from them and made this write up to guide others who might run into the same issue.

- [spwoodcock.dev: How to fix the eSIM eUICC on a Google Pixel running GrapheneOS](https://spwoodcock.dev/blog/2026-08-05-graphene-pixel-esim-fix/) which uses LineageOS to get root
- [XDA: [GUIDE] Fix Pixel eSIM / eUICC](https://xdaforums.com/t/guide-fix-pixel-esim-euicc.4797083/) which covers choosing the right firmware and the thread's later findings

This guide assumes you're using a Mac. Linux should work the same way. On Windows you will need to install the Google USB Driver.

> **Warning**
> - Unlocking the bootloader and formatting data **wipes the phone**, so back it up first
> - Your **EID will probably change.** Expect to request a new eSIM from your carrier afterward
> - Firmware flashing carries some risk. Read every confirmation dialog, and stop if you see the *"non-compliant image … will break your device"* warning

***READ THROUGH ALL OF THE INSTRUCTIONS BEFORE STARTING***

## Confirm the problem

Dial `*#*#4636#*#*` → **Phone information** → eUICC info

If **available memory is `-1`**, then you have the issue this guide fixes.

## Prepare your computer

- Install the latest **platform-tools**: `brew install --cask android-platform-tools` or download them from Google
- Quit Chrome (Cmd+Q) when you use `adb` or `fastboot`
  - WebUSB pages such as Android Flash Tool or the GrapheneOS web installer can take over the USB connection. When that happens, `fastboot` hangs at `waiting for any device` and `fastboot devices` comes back empty. To check, run `ioreg -r -c IOUSBHostDevice -l -w0 | grep UsbExclusiveOwner`
- Plug the phone directly into the Mac with a cable that carries data

## Unlock the bootloader

1. On the phone, open Settings → About phone and tap **Build number** 7 times. This unlocks developer mode.
2. Go to Developer options then turn on **OEM unlocking** and **USB debugging**
3. Run `adb reboot bootloader`, then `fastboot flashing unlock`, and confirm on the phone. This wipes the phone

> **Avoid Android Flash Tool** (flash.android.com). The XDA guide reports that it can *cause* this eSIM failure. If you need to restore stock Android, use the official factory image and its `flash-all` script.

## Install LineageOS + MindTheGapps

Follow the [LineageOS tokay install guide](https://wiki.lineageos.org/devices/tokay/install/). Gapps is required as the firmware updater app crashes on LineageOS without Google components.

Use matching versions:

| LineageOS | Android | MindTheGapps |
|---|---|---|
| 24.x | 17 | 17.0.0 **arm64** |
| 23.x | 16 | 16.0.0 **arm64** |

Do the steps in this exact order and **don't boot into Android in between**:

1. Flash the images the wiki lists and boot into LineageOS recovery on the phone
2. Choose **Factory reset → Format data/factory reset**
3. Choose **Apply update → Apply from ADB**, then run `adb sideload MindTheGapps-<ver>-arm64-<date>.zip`. Accept the "Signature verification failed" prompt. If the terminal stops at about 47%, that's normal
4. Choose **Reboot system now**

Common problems at this stage:

| Symptom | Fix |
|---|---|
| `can't load android system … init_user0_failed` | Data wasn't formatted. Go back to recovery, choose Format data, sideload gapps again, then reboot |
| Setup wizard stuck on **Navigation** (Next does nothing) | Redo the clean sequence above. At the Navigation screen, pick **3-button navigation**, tap Next once, and wait about 30 seconds |
| `adb: failed to run abb_exec … closed` | The phone is in recovery or sideload mode. Boot fully into Android |

## Enable debugging in LineageOS

1. Connect to Wi-Fi. The updater downloads the firmware from Thales's server
2. In Developer options, turn on **USB debugging** and **Rooted debugging**
3. Run `adb devices`. The phone must show as `device`, not `recovery` or `unauthorized`

## Install a current firmware updater (D.2.0.23 or newer)

An old updater app will be refused by the firmware server. **D.2.0.22 (the blog's copy) failed at this step; D.2.0.23 and D.2.0.24 worked.**

The best source is the **current tokay factory image** from [developers.google.com/android/images](https://developers.google.com/android/images#tokay). You don't need Docker:

```bash
# from the factory zip, extract only system_ext.img (Python can read the nested zip in place)
python3 - <<'EOF'
import zipfile, shutil, glob
outer = zipfile.ZipFile(glob.glob('tokay-*-factory-*.zip')[0])
inner_name = [n for n in outer.namelist() if '/image-tokay-' in n][0]
inner = zipfile.ZipFile(outer.open(inner_name))
with inner.open('system_ext.img') as s, open('system_ext.img', 'wb') as d:
    shutil.copyfileobj(s, d, 1 << 20)
EOF

file system_ext.img      # ext4 on cp3a.261005.005 (Oct 2026)
# ext4: brew install sevenzip && 7zz x system_ext.img -oout
# (Linux: debugfs -R "rdump /priv-app/EuiccSupportPixel-P23 ." system_ext.img)
# EROFS: brew install erofs-utils && fsck.erofs --extract=out system_ext.img
find out -iname "EuiccSupportPixel*.apk"
```

The APK is `priv-app/EuiccSupportPixel-P23/EuiccSupportPixel-P23.apk`. The name says P23 even on a Pixel 9. Factory image `cp3a.261005.005` contains **D.2.0.24**.

Install it and confirm the version:

```bash
adb install -r EuiccSupportPixel-P23.apk
adb shell pm list packages --show-versioncode -f | grep com.google.euiccpixel
adb shell dumpsys package com.google.euiccpixel | grep -E "versionCode|versionName"
```

If you get a signature error, run `adb uninstall com.google.euiccpixel` and install again.

## Open the updater with root and SELinux permissive

```bash
adb root
adb shell setenforce 0
adb shell getenforce        # must print: Permissive
adb shell am start -S com.google.euiccpixel/.ui.OtaUpdateMenu
```

You can also open the updater by dialing `*#*#3987#*#*`. On stock Android with Magisk, follow the XDA guide's `su` and `resetprop` steps instead.

`setenforce 0` resets on every reboot, so run it again if you restart the phone.

## Choose the firmware: let the confirmation dialog decide

1. Set the config key to **P24 (ST54L)**. This applies to the entire Pixel 9 family
2. Open **Upgrade Firmware** and choose an entry that starts with `GGL_P24_ST54L_GSMALIVE_534-1.1_DVT_...`
3. Tap **SE FIRMWARE UPGRADE** and **read the dialog before tapping Continue**:

| Dialog says | Meaning |
|---|---|
| "You are flashing to the same image again" | That image is already installed. Choose another entry |
| "You are flashing to a **non-compliant image** … It will break your device!" | Wrong config. **Back out** |
| Only "eSIM ID (EID) will change after upgrade" | The updater considers it compatible. This notice is expected |

**What happened with this Pixel 9:**

- The chip identified itself as `GGL_P24_ST54L_GSMALIVE_DVT_CG7_v2` in logcat. The dialog showed `unknown_image` as the current firmware.
- `..._534-1.1_DVT_CG7_v2` **failed every time**. The server returned `No image available (HTTP code 404)`, and the chip recovered on its own each time.
- **`GGL_P24_ST54L_GSMALIVE_534-1.1_DVT_CC9_v2` installed successfully.** Other Pixel 9-series owners in the XDA thread reported the same: CG7 failed and CC9 worked.
- Avoid `..._QrTST_FMCT2`. It appears to be a test build.

To see the error when an upgrade fails:

```bash
adb logcat -c
adb logcat > euicc_upgrade_log.txt     # tap Continue on the phone, wait for the result, then Ctrl+C
grep -E "Image Label|HTTP response|Exception|Recovery back" euicc_upgrade_log.txt
```

To reflash an image the updater says is "the same image again" (rooted stock Android, from XDA):

```bash
adb shell su -c "am broadcast -n com.google.euiccpixel/com.google.android.flib.phenotype.receivers.OverrideReceiver -a com.google.euiccpixel.OVERRIDE_EXPERIMENT --es Ota__should_check_image_type_before_upgrade false"
# ...flash, then clear the override:
adb shell su -c "am broadcast --user 0 -n com.google.euiccpixel/com.google.android.flib.phenotype.receivers.OverrideReceiver -a com.google.euiccpixel.OVERRIDE_EXPERIMENT --esn Ota__should_check_image_type_before_upgrade"
```

## Reboot, then verify

**Reboot after a successful upgrade.** Immediately after flashing, eUICC memory will still show `-1`. After a reboot it should show a positive number.

Dial `*#*#4636#*#*` again. A **positive available memory** value means the fix worked. Then request a new eSIM from your carrier, because the EID has changed.

## Avoid it breaking again

- **Relocking the bootloader can bring the `-1` back.** XDA users saw this after relocking on Android 17 firmware
  - On **stock Android**, the workaround that worked was flashing an older **Android 16** factory image with `flash-all` (for example a BP2A.250605.031.A2 build), relocking, and then updating to 17 over the air
  - On **GrapheneOS** (Android 17-based), there's no confirmed safe order at this time. After installing, boot once and check `*#*#4636#*#*` *before* locking.  Lock, then check again. If it broke, you'll need to unlock (which wipes the phone) and repeat this guide from LineageOS. Release 2026101600 says it fixes eSIM firmware updates on 9th-gen Pixels, but it doesn't mention relocking
- Don't use Android Flash Tool. Avoid unnecessary unlock and relock cycles
- GrapheneOS needs sandboxed Google Play plus the eSIM toggle in Settings → Network & internet → SIMs to activate an eSIM
