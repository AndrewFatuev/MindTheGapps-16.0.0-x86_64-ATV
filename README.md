# MindTheGapps-16.0.0-x86_64-ATV
Build manual and scripts for MindTheGapps 16.0.0 (Android 16 / LineageOS 23.x) x86_64 Android TV recovery zip with Classic Android TV Home, Recommendations, and Google Assistant.

# Building MindTheGapps 16.0.0 `x86_64` ATV Recovery Zip
Manual for building a recovery-flashable, signed **MindTheGapps 16.0.0 (`Android 16` / `LineageOS 23.x`) `x86_64` Android TV** package with **Classic Android TV Home (`TVLauncher`)**, **Recommendations (`TVRecommendations`)**, **Google Play Store (`Tubesky`)**, and **Google Search / Assistant (`Katniss`)**.

---

## Target ROM & Sources

* **Target ROM Build**: `lineage-23.2-20261115-UNOFFICIAL-x86_64_tv.iso` ([Download Mirror](https://fuckingfast.net/7cb4uj0rcasj))
* **Build System & Installer Scripts**
  * Host: `github.com`
  * Repository: `MindTheGapps` / `vendor_gapps_tv` (`baklava` / Android 16 branch)
* **Native `x86` / `noarch` GMS Core & Store Binaries**
  * Host: `github.com` (organization: `WayDroid-ATV`)
  * Host: `sourceforge.net` (path: `projects` / `waydroid-atv` / `files` — `lineage-23.0-*-GAPPS-waydroid_tv_x86_64-system.zip`)
* **Classic Android TV Home & `arm64-v8a` Leanback APKs**
  * Host: `github.com`
  * Repository: `MindTheGapps` / `16.0.0-arm64-ATV` (`MindTheGapps-16.0.0-arm64-ATV-full-*.zip`)
* **Static Recovery `toybox-x86_64` Binary**
  * Host: `landley.net` (path: `toybox` / `bin` / `toybox-x86_64`)

---

## Package Origins Breakdown

Because official MindTheGapps stopped shipping `x86_64` builds after Android 13, this package combines native `x86`/`noarch` GMS core binaries from **WayDroid-ATV** with `noarch`/`arm64-v8a` leanback apps from **MindTheGapps `16.0.0-arm64-ATV`**, packaged via **`MindTheGapps/vendor_gapps_tv`**:

| Component / APK | ABI | Source | Role |
| :--- | :--- | :--- | :--- |
| `PrebuiltGmsCorePano.apk` | `x86` | WayDroid-ATV `x86_64` image | Google Play Services for TV |
| `Tubesky.apk` | `x86` / `x86_64` | WayDroid-ATV `x86_64` image | Google Play Store for Android TV |
| `GoogleServicesFramework.apk` | `noarch` | WayDroid-ATV `x86_64` image | Google Services Framework (GSF) |
| `GoogleExtShared.apk` | `noarch` | WayDroid-ATV `x86_64` image | Google Shared Services |
| `GoogleCalendarSyncAdapter.apk` | `noarch` | WayDroid-ATV `x86_64` image | Google Calendar Sync |
| `GoogleTTS.apk` | `x86` | WayDroid-ATV `x86_64` image | Google Text-to-Speech |
| `talkback.apk` | `x86` | WayDroid-ATV `x86_64` image | Android Accessibility Suite |
| `AtvRemoteService.apk` | `noarch` | WayDroid-ATV `x86_64` image | Android TV Remote Service |
| `GoogleOneTimeInitializer.apk` | `noarch` | WayDroid-ATV `x86_64` image | Google Account Initializer |
| `TVLauncher.apk` | `noarch` | MindTheGapps `16.0.0-arm64-ATV` | Classic Android TV Home |
| `TVRecommendations.apk` | `arm64-v8a` | MindTheGapps `16.0.0-arm64-ATV` | Android TV Core Services / Recommendations |
| `Backdrop.apk` | `noarch` | MindTheGapps `16.0.0-arm64-ATV` | Ambient Mode / Screensaver |
| `Katniss.apk` | `arm64-v8a` | MindTheGapps `16.0.0-arm64-ATV` | Google Search & Assistant |
| `SetupWraithPrebuilt.apk` | `arm64-v8a` | MindTheGapps `16.0.0-arm64-ATV` | Android TV Setup Wizard |
| `build/`, `overlay/`, `addond_*` | — | `MindTheGapps/vendor_gapps_tv` | RRO compiler, `update-binary`, OTA survival & signing |

*(32-bit `armeabi-v7a` APKs and platform-signed `TvFeedbackConsent` are intentionally excluded to avoid `INSTALL_FAILED_NO_MATCHING_ABIS` `-113` scan failures and `android.uid.system` signature mismatches.)*

---

## Host Requirements

* Linux environment with `git`, `curl`, `zip`, `unzip`, `python3`, `make`, and **OpenJDK 17+** (`java`).
* Target `x86_64` Android 16 TV ROM (`lineage-23.2-20261115-UNOFFICIAL-x86_64_tv.iso`) includes 64-bit ARM translation (`ro.dalvik.vm.native.bridge=libndk_translation.so` / `binfmt_misc_arm64`) to run `arm64-v8a` leanback packages (`Katniss`, `TVRecommendations`, `SetupWraithPrebuilt`).

---

## Step-by-Step Build Manual

### 1. Clone `vendor_gapps_tv` & Download `toybox-x86_64`

```bash
git clone <VENDOR_GAPPS_TV_REPO_URL> ~/vendor_gapps_tv
cd ~/vendor_gapps_tv

# Fetch static x86_64 toybox binary used by META-INF/com/google/android/update-binary
curl -fL <TOYBOX_X86_64_URL> -o toybox-x86_64
chmod +x toybox-x86_64
```

### 2. Extract Source Archives

1. **Extract WayDroid-ATV `x86_64` system image** and mount it read-only:
   ```bash
   mkdir -p ~/wd_atv ~/mnt_wd
   unzip ~/Downloads/lineage-23.0-*-GAPPS-waydroid_tv_x86_64-system.zip -d ~/wd_atv
   sudo mount -o loop,ro ~/wd_atv/system.img ~/mnt_wd
   ```
2. **Extract the official `MindTheGapps-16.0.0-arm64-ATV-full-*.zip`**:
   ```bash
   mkdir -p ~/mtg_arm64
   unzip ~/Downloads/MindTheGapps-16.0.0-arm64-ATV-full-*.zip -d ~/mtg_arm64
   ```

### 3. Populate `x86_64/proprietary/`

```bash
cd ~/vendor_gapps_tv
mkdir -p x86_64/proprietary/{app,product/app,product/priv-app,system_ext/priv-app}

# 1. Native x86 / noarch core GMS & Play Store packages (from WayDroid-ATV)
sudo cp -r ~/mnt_wd/system/app/GoogleExtShared x86_64/proprietary/app/
sudo cp -r ~/mnt_wd/system/product/app/GoogleCalendarSyncAdapter x86_64/proprietary/product/app/
sudo cp -r ~/mnt_wd/system/product/app/GoogleTTS x86_64/proprietary/product/app/
sudo cp -r ~/mnt_wd/system/product/app/talkback x86_64/proprietary/product/app/
sudo cp -r ~/mnt_wd/system/product/priv-app/AtvRemoteService x86_64/proprietary/product/priv-app/
sudo cp -r ~/mnt_wd/system/product/priv-app/GoogleOneTimeInitializer x86_64/proprietary/product/priv-app/
sudo cp -r ~/mnt_wd/system/product/priv-app/PrebuiltGmsCorePano x86_64/proprietary/product/priv-app/
sudo cp -r ~/mnt_wd/system/product/priv-app/Tubesky x86_64/proprietary/product/priv-app/
sudo cp -r ~/mnt_wd/system/system_ext/priv-app/GoogleServicesFramework x86_64/proprietary/system_ext/priv-app/

sudo chown -R (id -u):(id -g) x86_64/proprietary
sudo umount ~/mnt_wd

# 2. Classic Android TV Home, Recommendations, Search (Katniss) & Setup Wizard (from MindTheGapps arm64-ATV)
cp -r ~/mtg_arm64/system/product/priv-app/TVLauncher x86_64/proprietary/product/priv-app/
cp -r ~/mtg_arm64/system/product/priv-app/TVRecommendations x86_64/proprietary/product/priv-app/
cp -r ~/mtg_arm64/system/product/priv-app/Backdrop x86_64/proprietary/product/priv-app/
cp -r ~/mtg_arm64/system/product/priv-app/Katniss x86_64/proprietary/product/priv-app/
cp -r ~/mtg_arm64/system/product/priv-app/SetupWraithPrebuilt x86_64/proprietary/product/priv-app/
```

*(Optional sanity check — confirm zero `armeabi-v7a` APKs exist in `x86_64/proprietary/`:)*
```bash
find x86_64/proprietary -name "*.apk" -exec unzip -l {} \; | grep armeabi-v7a || echo "OK: No armeabi-v7a APKs"
```

### 4. Add `TvSettingsTwoPanel` Privileged Permission Allowlist

Add `MODIFY_AUDIO_ROUTING` for `com.android.tv.settings` so LineageOS TV's `TvSettingsTwoPanel` passes Android 16's privileged permission check at boot:

```bash
cd ~/vendor_gapps_tv
mkdir -p common/proprietary/system_ext/etc/permissions

cat << 'EOF' > common/proprietary/system_ext/etc/permissions/privapp-permissions-tvsettings-fix.xml
<?xml version="1.0" encoding="utf-8"?>
<permissions>
    <privapp-permissions package="com.android.tv.settings">
        <permission name="android.permission.MODIFY_AUDIO_ROUTING"/>
    </privapp-permissions>
</permissions>
EOF
```

### 5. Add the `gapps_tv_x86_64` Target & Build the Signed Zip

```bash
cd ~/vendor_gapps_tv

cat << 'EOF' >> build/main.mk

gapps_tv_x86_64:
	@echo "Compiling GApps with GMS launcher for x86_64..."
	@bash \$(BUILD_GAPPS) x86_64 full 2>&1
EOF

make gapps_tv_x86_64
```

The signed recovery zip and `.sha256sum` file are created in `out/MindTheGapps-16.0.0-x86_64-ATV-full-*.zip`.

---

## Flashing via Recovery

1. Install/flash a clean, unmodified **`lineage-23.2-20261115-UNOFFICIAL-x86_64_tv.iso`**.
2. Boot into **Recovery** (LineageOS Recovery or TWRP).
3. Sideload the generated zip:
   ```bash
   adb sideload ~/vendor_gapps_tv/out/MindTheGapps-16.0.0-x86_64-ATV-full-*.zip
   ```
   *(If LineageOS Recovery reports a signature verification prompt for testkeys, select **Yes** to install.)*
4. Perform a **Factory Reset / Wipe Data** in recovery before first boot.
5. Reboot into Android TV.
