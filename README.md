# Force 120 Hz on All Apps — HyperOS 3 (ADB)

ADB-based method to force 120 Hz refresh rate on compatible Xiaomi/Redmi devices running HyperOS 3, including diagnostics, verification, and rollback commands.

> **Tested setup:** Redmi 17 5G, HyperOS 3, 720×1600 120 Hz display  
> **Method:** ADB — no root required  
> **Goal:** Keep the phone's display policy at 120 Hz across apps, including apps that HyperOS may otherwise run at 60 Hz.

---

## ⚠️ Important

This guide was developed by testing the phone's actual Android display state with `dumpsys display`.

- It changes Android/HyperOS display settings.
- Results can vary between Xiaomi/Redmi models and HyperOS versions.
- A 120 Hz **display refresh rate does not guarantee that every app renders at 120 FPS**. An app may still render fewer frames.
- Keep a record of your original settings before experimenting if you want an easy rollback.
- Commands below assume Windows Command Prompt and an ADB executable at:

```text
D:\Android\platform-tools\adb.exe
```

Replace that path with your own `adb.exe` path if necessary.

---

# 1. Requirements

You need:

1. A Redmi/Xiaomi phone running HyperOS 3.
2. A display that actually supports 120 Hz.
3. A Windows PC.
4. Android Platform Tools / ADB.
5. USB debugging enabled.

On the phone:

**Settings → About phone → tap OS/Build number repeatedly to enable Developer Options**

Then:

**Settings → Additional settings → Developer options → USB debugging → ON**

Connect the phone by USB and authorize the computer when prompted.

---

# 2. Verify ADB

Open Command Prompt:

```bat
"D:\Android\platform-tools\adb.exe" devices
```

You should see your phone listed as:

```text
xxxxxxxxxxxxxxxx    device
```

If it says `unauthorized`, unlock the phone and accept the USB debugging authorization prompt.

---

# 3. Check the display's available refresh rates

Before changing anything, inspect the display:

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "supportedRefreshRates mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

The important thing is to confirm that 120 Hz is actually supported by the hardware.

A typical supported display may expose:

```text
120 Hz
90 Hz
60 Hz
```

Do **not** try to force 120 Hz if the panel does not support it.

---

# 4. Set Android's preferred display mode to 120 Hz

Android provides a native display-manager shell command:

```bat
"D:\Android\platform-tools\adb.exe" shell cmd display set-user-preferred-display-mode 720 1600 120 0
```

The `720 1600` values correspond to the tested phone's display resolution.

Verify:

```bat
"D:\Android\platform-tools\adb.exe" shell cmd display get-user-preferred-display-mode 0
```

Expected:

```text
User preferred display mode: 720 1600 120.0
```

### Important

This alone did **not** force the tested HyperOS 3 phone to actually run at 120 Hz.

The active display initially remained at 60 Hz because HyperOS's own refresh-rate policy was overriding the preference.

---

# 5. Inspect HyperOS's refresh-rate policy

Run:

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "PRIORITY_MIUI_REFRESH_RATE mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

Before the successful change, the MIUI policy repeatedly produced votes such as:

```text
PRIORITY_MIUI_REFRESH_RATE -> RenderVote{ RefreshRateVote{ mMinRefreshRate=0.0, mMaxRefreshRate=60.0 } }
```

This showed that HyperOS/MIUI's refresh-rate policy was imposing a 60 Hz ceiling.

The display dump also showed that the phone had 120/90/60 Hz modes available, so the limitation was policy rather than lack of 120-Hz hardware.

---

# 6. The key command

The command that successfully changed the HyperOS MIUI refresh-rate policy was:

```bat
"D:\Android\platform-tools\adb.exe" shell settings put secure miui_refresh_rate 120
```

After running it, immediately check:

```bat
"D:\Android\platform-tools\adb.exe" shell settings get secure miui_refresh_rate
```

Then:

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "PRIORITY_MIUI_REFRESH_RATE mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

On the tested device, this changed the active mode to:

```text
mActiveModeId=1
```

and:

```text
peakRefreshRate=120.00001
mActiveRenderFrameRate=120.00001
```

The MIUI policy history also began showing 120-Hz votes:

```text
PRIORITY_MIUI_REFRESH_RATE -> RenderVote{ RefreshRateVote{ mMinRefreshRate=0.0, mMaxRefreshRate=120.0 } }
```

---

# 7. Verify the actual active refresh rate

Use:

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

Success looks like:

```text
mActiveModeId=1
mActiveSfDisplayMode=DisplayMode{..., peakRefreshRate=120.00001, vsyncRate=120.00001, ...}
mActiveRenderFrameRate=120.00001
```

The important fields are:

- `mActiveModeId=1`
- `peakRefreshRate=120`
- `mActiveRenderFrameRate=120`

This verifies that the display is **actually operating in the 120-Hz mode**, rather than merely having a 120-Hz preference stored.

---

# 8. Test individual apps

Open the app you want to test 

Use the app normally for 20–30 seconds.

Then run:

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

On the tested app, the display remained:

```text
mActiveModeId=1
peakRefreshRate=120.00001
mActiveRenderFrameRate=120.00001
```

This demonstrates that the HyperOS policy was not immediately dropping the panel back to 60 Hz when that app was opened.

---

# 9. Reboot test — IMPORTANT

Do not assume the modification is persistent until you test it.

Reboot:

```bat
"D:\Android\platform-tools\adb.exe" reboot
```

After the phone completely boots and ADB reconnects, check:

```bat
"D:\Android\platform-tools\adb.exe" shell settings get secure miui_refresh_rate
```

Then:

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

On the tested Redmi 17 5G / HyperOS 3 configuration, the result after reboot remained:

```text
mActiveModeId=1
peakRefreshRate=120.00001
mActiveRenderFrameRate=120.00001
```

Therefore the modification survived a full reboot on the tested device.

---

# 10. Additional settings encountered during testing

During troubleshooting, the following settings were also modified/inspected:

### System

```text
min_refresh_rate=0
peak_refresh_rate=120
plugin_refresh_rate=-1
user_refresh_rate=1
```

### Secure

```text
miui_refresh_rate=1
user_refresh_rate=1
```

These values were part of the troubleshooting process.

**Do not blindly copy every experimental setting from this section.**

The key successful intervention was:

```bat
"D:\Android\platform-tools\adb.exe" shell settings put secure miui_refresh_rate 120
```

The exact interpretation of Xiaomi/HyperOS internal refresh-rate values can differ from the literal numeric refresh rate, so changing unrelated settings unnecessarily is not recommended.

---

# 11. Verify everything in one go

After setup, these commands provide a useful final check:

```bat
"D:\Android\platform-tools\adb.exe" shell settings get secure miui_refresh_rate
```

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

Expected result on the tested device:

```text
120
```

and:

```text
mActiveModeId=1
peakRefreshRate=120.00001
mActiveRenderFrameRate=120.00001
```

---

# 12. What this actually accomplishes

There are three different concepts that should not be confused:

### Display refresh rate

The physical display refreshes at 120 Hz.

Verified with:

```text
mActiveRenderFrameRate=120.00001
```

### Android display policy

HyperOS's MIUI refresh-rate policy permits the 120-Hz mode instead of imposing a 60-Hz ceiling.

### App frame rendering

An app may still render at 60 FPS even while the panel is running at 120 Hz.

Therefore:

> **This guide forces the display/policy to 120 Hz. It cannot guarantee that every application internally produces 120 frames per second.**

---

# 13. Useful diagnostic commands

### Check current MIUI refresh-rate setting

```bat
"D:\Android\platform-tools\adb.exe" shell settings get secure miui_refresh_rate
```

### Check active display mode

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

### Check MIUI refresh-rate votes

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "PRIORITY_MIUI_REFRESH_RATE"
```

### Check desired display mode specifications

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "mDesiredDisplayModeSpecs"
```

### Check refresh-related system settings

```bat
"D:\Android\platform-tools\adb.exe" shell settings list system | findstr /i "refresh rate"
```

### Check refresh-related secure settings

```bat
"D:\Android\platform-tools\adb.exe" shell settings list secure | findstr /i "refresh rate"
```

---

# 14. Rollback / troubleshooting

If you want to undo the main modification:

```bat
"D:\Android\platform-tools\adb.exe" shell settings delete secure miui_refresh_rate
```

Then reboot:

```bat
"D:\Android\platform-tools\adb.exe" reboot
```

**However, don't use this as a first troubleshooting step if the phone is currently working.**

HyperOS may recreate its own default value after reboot.

If you experimented with other settings, inspect them first rather than deleting them blindly:

```bat
"D:\Android\platform-tools\adb.exe" shell settings list system | findstr /i "min_refresh_rate peak_refresh_rate user_refresh_rate plugin_refresh_rate"
```

```bat
"D:\Android\platform-tools\adb.exe" shell settings list secure | findstr /i "miui_refresh_rate user_refresh_rate"
```

---

# 15. Quick version

For a compatible HyperOS 3 device, the essential workflow is:

```bat
"D:\Android\platform-tools\adb.exe" devices
```

```bat
"D:\Android\platform-tools\adb.exe" shell cmd display set-user-preferred-display-mode 720 1600 120 0
```

```bat
"D:\Android\platform-tools\adb.exe" shell settings put secure miui_refresh_rate 120
```

Verify:

```bat
"D:\Android\platform-tools\adb.exe" shell dumpsys display | findstr /i "mActiveModeId mActiveSfDisplayMode mActiveRenderFrameRate"
```

Expected:

```text
mActiveModeId=1
peakRefreshRate=120.00001
mActiveRenderFrameRate=120.00001
```

Then reboot and verify again.

---

## Tested result

On the tested Redmi 17 5G running HyperOS 3:

**Before:** HyperOS policy could hold the active display at 60 Hz despite the panel supporting 120 Hz.

**After:** The MIUI refresh-rate policy accepted 120 Hz, SurfaceFlinger selected the 120-Hz mode, apps remained at 120 Hz, and the configuration survived a full reboot.

---

## Disclaimer

This is an experimental ADB configuration for Xiaomi/Redmi HyperOS.

Commands and internal settings may change between HyperOS versions, devices, regions, and firmware builds. Use only on hardware that genuinely supports the requested refresh rate.

