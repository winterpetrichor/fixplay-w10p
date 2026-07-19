[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc/4.0/)
# fixplay-w10p
Guide to jailbreaking the Nixplay W10P photo frame to run ImmichFrame

> ⚠️ **Note:** This guide was written with AI assistance. All steps were personally tested and verified, but use your own judgment and proceed at your own risk.

# Jailbreak a Nixplay W10P Photo Frame
### Using Windows 11 + VirtualBox Ubuntu 20.04 VM + Home Assistant

Based on 
[Wil Clouser's guide](https://micropipes.com/2025/09/14/jailbreak-a-nixplay-w10p-photo-frame/), 
[Yo-less YouTube walkthrough](https://www.youtube.com/watch?v=TN5errM5UbA), 
[ThomasWildeTech ImmichFrame Frameo setup](https://thomaswildetech.com/blog/2025/12/13/frameo-with-immichframe-an-amazing-gift/#initial-frameo-setup)
Thank you

> **Disclaimer:** There is a lot I don't know. I followed guides and bumbled my way through over the course of a few days and it worked out for me. It may not for you. This guide is provided to assist, but it's at your own risk. I may not be able to answer questions definitely or at all. This is not an area of my expertise.

---

## References

- https://micropipes.com/2025/09/14/jailbreak-a-nixplay-w10p-photo-frame/
- https://www.youtube.com/watch?v=TN5errM5UbA
- https://thomaswildetech.com/blog/2025/12/13/frameo-with-immichframe-an-amazing-gift/#initial-frameo-setup

---

## What You Need

### Hardware

- Nixplay W10P photo frame (2023 model)
- Windows 11 PC with a spare, reliable USB port
- USB-A to USB-A cable (data capable — not charge only)
- AC adapter for the frame (not strictly required)
- Home server running Ubuntu (this guide uses an Odroid HC4 with Ubuntu 20.04) for Immich to replace the Nixplay service for storing and serving images *(See Appendix for other options)*

### Software — Windows

- VirtualBox with Extension Pack installed and an Ubuntu 20.04 VM set up *(in theory you could do this whole process with a single Ubuntu machine if you have one)*
- Android Platform Tools for Windows (adb.exe, fastboot.exe)
  - I installed mine in `C:\Android\` — you could also do all the ADB work on the VM if you pass through the ADB USB device
- Vysor desktop app (vysor.io) for mouse control of the frame *(Optional)*
  - I did this on the Windows machine
  - In theory, you could just use the touchscreen on the W10P instead and skip Vysor altogether. I used Vysor.

### Software — Ubuntu VM

- At least 15GB free disk space in the VM
- Internet access for both Windows and the Ubuntu VM

### Server Side *(See Appendix for other options)*

- Immich running via Docker Compose on your home server (not the VM). I disabled the ML container because I just wanted to set up image serving for the Nixplay frame, and I used an Odroid HC4, which is probably not the best machine for the ML service resource-wise.
- ImmichFrame running alongside Immich — I used the same docker-compose file
- Cloudflare tunnel exposing ImmichFrame publicly — I used my Home Assistant install on a separate Odroid M1 running HAOS and the Cloudflare add-on to do this
- Cloudflare account with WAF access

### Frame Specs (for reference)

- Chipset: Rockchip RK3126
- Android 7.1.2, API level 25
- USB Vendor ID: 2207, Product ID: 310D

---

## Overview

If you're here, you know why I did this. This guide jailbreaks the frame so you can run ImmichFrame — a dedicated photo frame app that pulls from your self-hosted Immich photo library using a separate ImmichFrame service installed on the same machine as Immich.

The process has five main phases:

1. Phase 1 — Prepare your environment
2. Phase 2 — Access the bootloader and back up the frame
3. Phase 3 — Patch the system and boot images to enable ADB
4. Phase 4 — Install apps and configure the frame
5. Phase 5 — Set up Immich, ImmichFrame, and Cloudflare WAF

> 📝 **NOTE:** The W10P process is significantly different from the W10E. The W10E only requires plugging in a USB cable to get ADB access. The W10P requires patching both the system and boot images before ADB works at all. Do not follow W10E instructions for this model.

**The software side is reversible in principle:**
- You'd use the same environment and your ADB access to re-enable the Nixplay apps and disable ImmichFrame. You can also restore the original system and boot images if you backed them up before patching.
- I did not attempt to reverse it.

---

## Phase 1 — Prepare Your Environment

### Step 1.0 — Back Up Your Data

**Do this before declining terms or deleting your Nixplay account.**

A factory reset of the frame is likely to happen during this process. In theory you can just log back in, but regardless, I recommend backing up your albums — you can then use these to upload to Immich or one of the solutions in the Appendix.

In the Nixplay website app: **Photos → Content Library → choose album → Actions → Download Album**

### Step 1.1 — Configure VirtualBox USB

> ⚠️ **STUMBLING BLOCK:** USB passthrough WILL NOT work without the VirtualBox Extension Pack. Download it from virtualbox.org and install it via **File → Tools → Extension Pack Manager**. The version must match your VirtualBox version exactly.

> ⚠️ **STUMBLING BLOCK:** USB 2.0 (OHCI + EHCI) controller is required. USB 3.0 (xHCI) was tested and did not work reliably for this process. In VirtualBox VM **Settings → USB**, select **USB 2.0 (OHCI + EHCI) Controller**.

> ⚠️ **STUMBLING BLOCK:** USB filtering is mandatory. Without it, VirtualBox cannot claim the Rockchip device. You must set up filters before attempting any bootloader operations.

With your Ubuntu VM shut down:

1. Go to **Settings → USB**
2. Select **USB 2.0 (OHCI + EHCI) Controller**
3. Do not add any USB filters yet — you will add them after first seeing the device (Step 2.2)
4. Click OK and start the VM

### Step 1.2 — Install Tools in Ubuntu VM

Open a terminal in your Ubuntu VM and run:

```bash
sudo apt update
sudo apt install -y abootimg cpio gzip android-tools-adb usbutils build-essential libusb-1.0-0-dev git
sudo apt install -y rkflashtool
```

> 📝 **NOTE:** rkflashtool is available directly from apt on Ubuntu 20.04 — no need to build from source.

Verify rkflashtool is working:

```bash
rkflashtool 2>&1 | head -1
```

Should print usage info, not "command not found".

### Step 1.3 — Create Working Directory

```bash
mkdir ~/nixplay && cd ~/nixplay
```

All subsequent work happens in this directory.

---

## Phase 2 — Access the Bootloader

### Step 2.1 — Understand the Bootloader Window

> ⚠️ **STUMBLING BLOCK:** The Nixplay W10P does not have a clean standalone bootloader mode. The Rockchip USB interface appears very briefly before/during the factory reset/recovery sequence. Missing this window triggers a factory reset. If a factory reset starts, wait for it to complete fully — do not interrupt it. Once the animated Nixplay logo appears after the reset completes, you can try again.

> ⚠️ **STUMBLING BLOCK:** Factory resets may happen multiple times during this process. This is normal. The frame will return to the language selection screen each time. This does not affect the partitions you are working with.

The bootloader window appears as follows:

- Frame powers off completely
- You hold the reset button and power on
- Screen goes black — this is the bootloader window. It's very short (a few seconds at most) and executing rkflashtool here is the key to staying in it.
- There are accounts of a red rectangle with a blue progress bar — this is supposedly the recovery/factory reset screen; mine was plain white and black
- The Rockchip USB device appears in Windows Device Manager during this sequence
- After the bar fills twice, the animated Nixplay logo appears and normal boot begins
- The Rockchip device disappears at some point during or after the normal boot sequence

### Step 2.2 — Enter Bootloader Mode and Set Up USB Filters

> ⚠️ **STUMBLING BLOCK:** You must set up VirtualBox USB filters or the device is unlikely to be passed through to the VM in time. "Can't attach USB device" errors will appear without filters even when the device is visible in Windows Device Manager.

**First time only — discover the device entries:**

1. Unplug both USB and power from the frame completely
2. Ensure the A end of the USB cable is plugged into your PC
3. Hold the reset button (paperclip in the pinhole)
4. Plug in power via USB while holding reset — power is provided by your PC over USB, no AC adapter needed
5. Keep holding reset for up to 5 seconds
6. Watch the **VirtualBox Devices → USB** menu for new entries
7. You should see **"Unknown device 2207:310D"** and/or **"rockchip w10p03"** appear

> 📝 **NOTE:** If nothing appears after 5 seconds, try again but release the reset button as soon as the screen goes black rather than holding the full 5 seconds. Both timings work on different frames.

**Now add USB filters:**

1. Shut down the VM
2. Go to **Settings → USB**
3. Click the **"Add USB filter"** icon
4. Add a filter for **"Unknown device 2207:310D"** — Vendor ID: 2207, Product ID: 310D
5. Add a second filter for **"rockchip w10p03"** — same IDs
6. Both filters must be checked/enabled
7. Boot the VM

> 📝 **NOTE:** With both filters active, VirtualBox will automatically claim the Rockchip device the instant it appears during the bootloader sequence — no manual clicking required.

### Step 2.3 — Verify Bootloader Connection

Trigger the bootloader sequence again (unplug power and USB, hold reset, plug power), then in the Ubuntu VM:

```bash
lsusb | grep -i rock
```

You should see:

```
Bus 00X Device 00X: ID 2207:310d Fuzhou Rockchip Electronics Co., Ltd.
```

> 📝 **NOTE:** Replace `YOUR_USER` with your actual Ubuntu username in all paths throughout this guide.

Test rkflashtool can connect:

```bash
sudo bash -c 'rkflashtool r 0x00002000 0x00002000 > /home/YOUR_USER/nixplay/test.img && echo SUCCESS'
```

> ⚠️ **STUMBLING BLOCK:** Plain sudo redirection does not work with rkflashtool — always use `sudo bash -c 'command > file'` syntax for all rkflashtool read and write operations. This applies throughout the entire guide.

> ⚠️ **STUMBLING BLOCK:** By the time you have tested the connection the bootloader window may have passed. In my trials, only using rkflashtool kept the device in bootloader mode.

### Step 2.4 — Back Up All Partitions

> ⚠️ **STUMBLING BLOCK:** Do not skip the backup. If anything goes wrong during flashing, these images are your only way to restore the frame.

> ⚠️ **STUMBLING BLOCK:** rkflashtool may lose connection if the bootloader times out mid-dump. If this happens, re-enter bootloader mode and re-run the script — already completed images will be overwritten but that is fine. My VM even crashed during a read with no ill effects during this step.

Save the following as `~/nixplay/dump.sh`:

```bash
#!/bin/bash
# Partition offsets for Nixplay W10P ONLY
# Do NOT use these offsets for other Nixplay models
RKFLASHTOOL=rkflashtool
echo "Dumping uboot..."
$RKFLASHTOOL r 0x00002000 0x00002000 > uboot.img
echo "Dumping trust..."
$RKFLASHTOOL r 0x00004000 0x00002000 > trust.img
echo "Dumping misc..."
$RKFLASHTOOL r 0x00006000 0x00002000 > misc.img
echo "Dumping resource..."
$RKFLASHTOOL r 0x00008000 0x00008000 > resource.img
echo "Dumping kernel..."
$RKFLASHTOOL r 0x00010000 0x00006000 > kernel.img
echo "Dumping boot..."
$RKFLASHTOOL r 0x00016000 0x00006000 > boot.img
echo "Dumping recovery..."
$RKFLASHTOOL r 0x0001C000 0x00010000 > recovery.img
echo "Dumping system..."
$RKFLASHTOOL r 0x0019E000 0x00280000 > system.img
echo "Done."
```

Run it:

```bash
chmod +x ~/nixplay/dump.sh
cd ~/nixplay && sudo bash -c 'bash dump.sh'
```

> 📝 **NOTE:** Skip userdata — it takes 2+ hours and is not needed.

---

## Phase 3 — Patch System and Boot Images

### Step 3.1 — Extract adbd from Recovery Image

> ⚠️ **STUMBLING BLOCK:** The W10P system image does not include the adbd executable needed for ADB. It exists in the recovery image and must be copied across. This is the key difference from the W10E.

```bash
mkdir -p ~/nixplay/recovery_wd/ramdisk
cd ~/nixplay/recovery_wd
cp ../recovery.img .
abootimg -x recovery.img
cd ramdisk
gzip -dc ../initrd.img | cpio -i
ls sbin/adbd   # confirm it exists
```

### Step 3.2 — Inject adbd into System Image

> ⚠️ **STUMBLING BLOCK:** Always work from a freshly dumped system image. Verify the write succeeded by reading back from the device before rebooting — never assume a write succeeded. If a read doesn't show what you intended to write, you can just try again; this is what worked for me.

```bash
cd ~/nixplay
sudo modprobe loop
cp system.img system-new.img
mkdir -p simg
sudo mount -t ext4 -o loop system-new.img simg
sudo mkdir -p simg/sbin
sudo cp recovery_wd/ramdisk/sbin/adbd simg/sbin/adbd
sudo chmod 755 simg/sbin/adbd
ls -la simg/sbin/   # verify adbd is present
sync
sudo umount simg
```

### Step 3.3 — Flash System Image

Enter bootloader mode, then:

```bash
sudo bash -c 'rkflashtool w 0x0019E000 0x00280000 < /home/YOUR_USER/nixplay/system-new.img'
```

> ⚠️ **STUMBLING BLOCK:** Writing takes several minutes. Do not interrupt it. Do not touch the frame, cable, or VM while it is running.

Verify the write by reading back:

```bash
sudo bash -c 'rkflashtool r 0x0019E000 0x00280000 > /home/YOUR_USER/nixplay/system-verify.img'
mkdir -p ~/nixplay/sverify
sudo mount -t ext4 -o loop ~/nixplay/system-verify.img ~/nixplay/sverify
ls ~/nixplay/sverify/sbin/adbd   # must show the file
sudo umount ~/nixplay/sverify
```

Only proceed if adbd is confirmed present. Then reboot:

```bash
sudo rkflashtool b
```

### Step 3.4 — Understand the Boot Image Patch

> ⚠️ **STUMBLING BLOCK:** The original guide's patch uses `write /sys/class/android_usb/android0/...` sysfs paths. These **DO NOT WORK** on the W10P. The W10P uses `init.rk30board.usb.rc` for USB configuration via property triggers. Using the original guide's patch will result in ADB never starting.

The correct patch was determined by reading `init.rk30board.usb.rc` from the actual device and finding how it triggers adbd. The correct and verified patch for the `on boot` section of `init.rc` is:

```
    setprop sys.usb.configfs 0
    setprop sys.usb.config adb
    setprop service.adb.root 1
    start adbd
```

### Step 3.5 — Extract and Patch Boot Image

Re-enter bootloader mode, then dump the current boot image directly from the frame:

```bash
sudo bash -c 'rkflashtool r 0x00016000 0x00006000 > /home/YOUR_USER/nixplay/boot-live.img'
mkdir -p ~/nixplay/boot_wd/ramdisk
cd ~/nixplay/boot_wd
abootimg -x ~/nixplay/boot-live.img
cd ramdisk
gzip -dc ../initrd.img | cpio -i
```

Edit init.rc:

```bash
nano init.rc
```

Scroll to the end of the `on boot` section. It ends just before `on nonencrypted`. Add the four lines above the `class_start core` line so the end of the section looks like this:

```
    [... existing lines ...]
    setprop sys.usb.configfs 0
    setprop sys.usb.config adb
    setprop service.adb.root 1
    start adbd

    class_start core

on nonencrypted
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`). Repack:

```bash
find . | cpio -o -H newc | gzip > ../initrd-new.img
cd ~/nixplay/boot_wd
abootimg --create ~/nixplay/boot-new.img -f bootimg.cfg -k zImage -r initrd-new.img
```

### Step 3.6 — Verify Patch Before Flashing

> ⚠️ **STUMBLING BLOCK:** Always verify the patch is present in the new image before flashing. Skipping this verification means you may flash a broken image and need to repeat the entire process.

```bash
mkdir -p ~/nixplay/boot_verify/ramdisk
cd ~/nixplay/boot_verify
abootimg -x ~/nixplay/boot-new.img
cd ramdisk
gzip -dc ../initrd.img | cpio -i
grep -A 10 "start adbd" init.rc
```

You must see the four patch lines present before proceeding.

### Step 3.7 — Flash Boot Image

Re-enter bootloader mode, then:

```bash
sudo bash -c 'rkflashtool w 0x00016000 0x00006000 < /home/YOUR_USER/nixplay/boot-new.img'
```

> ⚠️ **STUMBLING BLOCK:** Use explicit hex offsets for boot partition writes — `rkflashtool w boot < file.img` with partition name failed during testing. Always use the explicit offsets shown above.

Verify before rebooting by reading back from the device:

```bash
sudo bash -c 'rkflashtool r 0x00016000 0x00006000 > /home/YOUR_USER/nixplay/boot-verify.img'
mkdir -p ~/nixplay/boot_final_check/ramdisk
cd ~/nixplay/boot_final_check
abootimg -x ~/nixplay/boot-verify.img
cd ramdisk
gzip -dc ../initrd.img | cpio -i
grep -A 10 "start adbd" init.rc
```

You must see all four patch lines:

```
    setprop sys.usb.configfs 0
    setprop sys.usb.config adb
    setprop service.adb.root 1
    start adbd
```

Before running `sudo rkflashtool b` — otherwise try repeating from Step 3.5, and if that doesn't work, redo Steps 2.2–2.3 then try from Step 3.5 again.

Then reboot:

```bash
sudo rkflashtool b
```

### Step 3.8 — Confirm ADB Connection

Let the frame boot fully. Plug in the USB cable normally (no bootloader mode). The frame should appear in Windows Device Manager as `w10p03` under **Universal Serial Bus devices** with no yellow warning triangle.

On Windows in `C:\Android\`:

```cmd
adb devices
```

You should see:

```
List of devices attached
XXXXXXXXXX    device
```

> 📝 **NOTE:** All subsequent ADB commands are run from Windows (`C:\Android\`) using the Windows Platform Tools, not from the Ubuntu VM. The Ubuntu VM is only needed for rkflashtool partition operations.

---

## Phase 4 — Configure the Frame

### Step 4.1 — Disable Nixplay Apps

> ⚠️ **STUMBLING BLOCK:** Do this before installing a new launcher. If you install the launcher first the screen may go blank with no way to interact with it until you disable Nixplay.

```cmd
adb -d shell pm disable-user --user 0 com.kitesystems.nix.prod
adb -d shell pm disable-user --user 0 com.kitesystems.nix.frame
```

### Step 4.2 — Install Nova Launcher (optional)

You can use this to help navigate and troubleshoot the Android device — it's not really needed but can be useful. You can disable or uninstall it once ImmichFrame is set up.

Download Nova Launcher APK (select arm architecture, Android 7.0+ compatible version). Save to `C:\Android\` then:

```cmd
cd C:\Android
adb -d install nova-launcher-x.x.x.apk
```

Launch it:

```cmd
adb -d shell monkey -p com.teslacoilsw.launcher -c android.intent.category.LAUNCHER 1
```

> 📝 **NOTE:** The exact package name is `com.teslacoilsw.launcher` — verify with: `adb -d shell pm list packages | findstr tesla`

### Step 4.3 — Install and Use Vysor

Download and install Vysor from vysor.io on your Windows PC. Connect to the frame via Vysor — it detects the ADB device automatically and gives you full mouse control of the frame screen.

> ⚠️ **STUMBLING BLOCK:** Vysor and Windows ADB (command prompt) cannot be used simultaneously. When Vysor is connected, adb commands in cmd.exe will not work and vice versa. In practice, Vysor may appear frozen after using an ADB command — just disconnect and reconnect. Similarly, close Vysor before running ADB commands.

### Step 4.4 — Install ImmichFrame Android App

Download the ImmichFrame Android APK from the [ImmichFrame GitHub releases page](https://github.com/immichframe/ImmichFrame/releases). Save to `C:\Android\` then:

```cmd
adb -d install immichframe.apk
```

> 📝 **NOTE:** ImmichFrame detects itself as a launcher and starts automatically on boot, showing photos in a fullscreen slideshow. No additional launcher configuration is needed — it replaces the need for Nova Launcher for the final setup. Nova Launcher is only useful during initial setup for navigating the Android UI.

### Step 4.5 — USB and Power Notes

> 📝 **NOTE:** Once ADB setup is complete, you can disconnect the USB cable and run the frame on AC power only. ImmichFrame will continue showing photos without USB connected.

> ⚠️ **STUMBLING BLOCK:** ADB will NOT work when both AC power and USB are connected simultaneously. To use ADB again: disconnect AC power (which will power off the frame), then power it via USB only. Plugging both AC and USB at the same time prevented ADB from working for me.

---

## Phase 5 — Immich, ImmichFrame Server, and Cloudflare WAF

### Step 5.1 — Set Up Immich (Without Machine Learning)

> ⚠️ **STUMBLING BLOCK:** The Odroid HC4 has no GPU acceleration for ML tasks. Leaving the machine learning container running will consume significant RAM and CPU with no benefit for this use case. Disable it properly — not just via environment variable.

In your `docker-compose.yml`, comment out the entire immich-machine-learning service:

```yaml
  # immich-machine-learning:
  #   container_name: immich_machine_learning
  #   image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}
  #   volumes:
  #     - model-cache:/cache
  #   env_file:
  #     - .env
  #   restart: always
```

Also update the volumes section at the bottom — if model-cache was the only entry, replace it with:

```yaml
volumes: {}
```

> ⚠️ **STUMBLING BLOCK:** Leaving an empty `volumes:` section or only a commented-out entry causes a YAML validation error: "volumes must be a mapping". Use `volumes: {}` if no volumes remain.

> ⚠️ **STUMBLING BLOCK:** Do not put the postgres database inside your photos folder. Keep `DB_DATA_LOCATION` separate from your photo storage path.

Configure your `.env` file — specify your path:

```env
UPLOAD_LOCATION=/yourpath/immich/uploads
DB_DATA_LOCATION=/yourpath/immich/postgres
```

```bash
docker compose up -d
docker ps | grep immich   # should show immich_server, immich_postgres, immich_redis only
```

> ⚠️ **DO NOT SKIP THIS STEP:** Access the web UI at `http://<odroid-ip>:2283`. The first user registered becomes admin.

### Step 5.2 — Create a Read-Only Frame User and API Key

In Immich web UI, create a dedicated user for the frame:

- **Profile → Administration → Users → Create User**
- Email: `frame@home.local` (or any fake email)
- Name: Frame
- No storage quota needed

Log in as the frame user and create an API key with these permissions only:

- `asset.read`, `asset.view`, `asset.download`
- `album.read`, `album.download`
- `folder.read`
- `timeline.read`
- `user.read`
- `library.read`
- `server.about`

Copy the API key — shown only once.

Back as admin, create an album, upload some test photos, and share it with the frame user as **Viewer (read-only)**.

### Step 5.3 — Add ImmichFrame to Docker Compose

> ⚠️ **STUMBLING BLOCK:** ImmichFrame is a separate service/container that sits between the Android app and Immich. The Android app connects to ImmichFrame, which connects to Immich. It is not a feature of Immich itself.

> ⚠️ **STUMBLING BLOCK:** All environment variables must use the same format. Mixing `Key: value` and `Key=value` styles in the same environment block causes a YAML error: "unexpected type map[string]interface {}". Use list style (`Key=value` with dash) throughout.

Add this service to your `docker-compose.yml`:

```yaml
  immichframe:
    container_name: immich_frame
    image: ghcr.io/immichframe/immichframe:latest
    ports:
      - 2284:8080          # host:container — do not change 8080
    environment:
      # Connection to Immich — use local Docker service name, not Cloudflare URL
      - ImmichServerUrl=http://immich_server:2283
      - ApiKey=<API KEY FOR READ-ONLY FRAME USER>
      - AuthenticationSecret=<YOUR_STRONG_RANDOM_SECRET>
      # Album — get UUID from Immich album URL
      - Albums=<ALBUM UUID>
      # Display settings
      - Layout=single
      - Framerate=60
      - ShowVideos=true
      - ShowProgressBar=false
      - ShowClock=false
      - ShowPhotoDate=false
      - ShowImageLocation=false
    restart: always
```

> ⚠️ **STUMBLING BLOCK:** Port mapping must be `2284:8080` — ImmichFrame listens internally on port 8080. Using `2284:2284` will not work.

> 📝 **NOTE:** Get the album UUID from the Immich web UI — open the album and copy the UUID from the URL: `http://<odroid-ip>:2283/albums/THIS-PART-HERE`

```bash
docker compose down && docker compose up -d
docker logs immich_frame --tail 20   # check for errors
```

Access ImmichFrame at `http://<odroid-ip>:2284/` to confirm photos are showing.

### Step 5.4 — Expose ImmichFrame via Cloudflare Tunnel

- In **Home Assistant → Settings → Add-ons → Cloudflare → Configuration**
- Add your ImmichFrame subdomain to the `additional_hosts` list:

```yaml
additional_hosts:
  - hostname: imfr.yourdomain.com
    service: http://<odroid-ip>:2284  # LAN IP of your Odroid
```

The add-on handles the tunnel automatically.

### Step 5.5 — Configure ImmichFrame App

Using Vysor, open ImmichFrame settings and enter:

- **Server URL:** your Cloudflare tunnel URL (e.g. `https://imfr.yourdomain.com`)
- **Authorization Secret:** the secret you set in `AuthenticationSecret` in docker-compose

### Step 5.6 — Configure Cloudflare WAF Rule

> ⚠️ **STUMBLING BLOCK:** The ImmichFrame Android app sends the AuthenticationSecret as an `Authorization: Bearer` header automatically. Browser access uses a URL parameter instead (`?authsecret=xxx`) which is less secure. The WAF rule blocks all requests that do not include the correct Authorization header, effectively blocking browser access while allowing the Android app to work transparently.

> ⚠️ **STUMBLING BLOCK:** The Cloudflare WAF visual rule builder does not work correctly for this use case — it generates an expression with an empty header name field that matches incorrectly. You must use the expression editor.

In **Cloudflare dashboard → select your domain → Security → WAF → Custom Rules → Create Rule**.

*This is separate from the Zero Trust tunnel configuration.*

1. Click **"Edit expression"** to open the expression editor
2. Paste this expression exactly:

```
not any(http.request.headers["authorization"][*] eq "Bearer YOUR_STRONG_RANDOM_SECRET") and http.host wildcard "imfr.yourdomain.com"
```

- Header name must be lowercase `authorization`
- `Bearer` must be capitalized exactly as shown
- The value is case-sensitive
- Replace `YOUR_STRONG_RANDOM_SECRET` with your AuthenticationSecret value (no angle brackets)
- Replace `imfr.yourdomain.com` with your actual ImmichFrame subdomain
- **Action:** Block
- **Status:** Active

**Verify the rule works:**

- Try accessing `https://imfr.yourdomain.com` in a browser → should be blocked by Cloudflare
- Open ImmichFrame app on the Nixplay → should connect and show photos normally
- If the app stops working after enabling the rule, disable it temporarily and check that `AuthenticationSecret` in docker-compose exactly matches what is entered in the app

> 📝 **NOTE:** The rule only applies to your ImmichFrame subdomain. All other subdomains (Home Assistant, Immich, etc.) are completely unaffected.

### Step 5.7 — Force Refresh Photos

To force ImmichFrame to fetch a fresh set of photos:

```bash
docker restart immich_frame
```

Then restart the app on the frame via ADB (USB connected, AC disconnected):

```cmd
adb -d shell pm clear com.android.webview
adb -d shell am force-stop com.immichframe.immichframe
adb -d shell am start -n com.immichframe.immichframe/.MainActivity
```

---

## Phase 6 — Theoretical Reversal

Re-enable the Nixplay apps and disable ImmichFrame:

```cmd
adb -d shell pm enable com.kitesystems.nix.prod
adb -d shell pm enable com.kitesystems.nix.frame
adb -d shell pm disable-user --user 0 com.immichframe.immichframe
```

Then reboot and the Nixplay launcher takes over again:

```cmd
adb -d reboot
```

**The one caveat** — the patched `boot.img` and `system.img` are still on the frame. They don't affect normal Nixplay operation since the Nixplay apps just override the launcher, but ADB root access via adbd is still running in the background.

For practical purposes, re-enabling the Nixplay apps and disabling ImmichFrame is sufficient to restore full Nixplay functionality without touching the partition images.

**If you wanted to fully revert to factory state** including removing the patches, flash your original backup images. Get back to bootloader mode and reconnect from the VM, then:

```bash
sudo bash -c 'rkflashtool w 0x0019E000 0x00280000 < /home/(YOUR VM USERNAME)/nixplay/system.img'
sudo bash -c 'rkflashtool w 0x00016000 0x00006000 < /home/(YOUR VM USERNAME)/nixplay/boot.img'
sudo rkflashtool b
```

---

## Appendix

Immich can be run on a NAS (Synology, TrueNAS) or even a Raspberry Pi 4 for those without an Odroid. The ML container is optional on any hardware, not just the HC4.

### Apps That Did Not Work

#### Fotoo

Fotoo is purpose-built for digital photo frames and supports Google Photos, Dropbox, and other cloud sources natively. However, all cloud source integrations require Google Play Services for authentication, which is not present on the stripped Android 7 build on the W10P. Cloud sources consistently fail. Local folder mode (adb push) works but requires manual updates. You could try installing Google Play Services to get these solutions to work and skip Immich altogether — this is non-trivial and outside the scope of this guide.

#### Aerial Views

Aerial Views 1.8.3 is the latest version as of this writing. It successfully lists Immich albums but fails with "failed to fetch selected albums" when using Immich v3. This is an API incompatibility — Immich v3 changed how album assets are enumerated. Watch for a future Aerial Views update that addresses Immich v3 compatibility.

#### Google Photos

Google Photos requires Google Play Services to authenticate. Not available on this device without significant additional work.

---

## Quick Reference — Key Commands

### rkflashtool (Ubuntu VM)

```bash
# Read partition
sudo bash -c 'rkflashtool r 0x00016000 0x00006000 > /home/USER/nixplay/file.img'

# Write partition
sudo bash -c 'rkflashtool w 0x00016000 0x00006000 < /home/USER/nixplay/file.img'

# Reboot frame
sudo rkflashtool b
```

### Partition Offsets (W10P Only)

| Partition | Start | Size |
|-----------|-------|------|
| boot | 0x00016000 | 0x00006000 |
| recovery | 0x0001C000 | 0x00010000 |
| system | 0x0019E000 | 0x00280000 |

### ADB (Windows `C:\Android\`)

```cmd
adb devices
adb -d shell pm list packages | findstr <name>
adb -d install app.apk
adb -d shell pm disable-user --user 0 com.package.name
adb -d reboot
adb -d reboot -p   # power off
```

### Docker (Odroid)

```bash
docker compose down && docker compose up -d
docker ps | grep immich
docker logs immich_frame --tail 50
docker restart immich_frame
```

---

*End of guide.*
