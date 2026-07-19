# Resources

All tools, software, and references used in this guide.

---

## Frame Tools

### rkflashtool
Used to read and write partitions on the Rockchip RK3126 chipset.
- **Install (Ubuntu 20.04):** `sudo apt install rkflashtool`
- No download needed — available directly from apt

### abootimg
Used to unpack and repack Android boot and recovery images.
- **Install (Ubuntu 20.04):** `sudo apt install abootimg`
- No download needed — available directly from apt

---

## Android Tools — Windows

### Android Platform Tools (ADB)
Used for all ADB operations on the frame after jailbreak.
- **Download:** https://developer.android.com/tools/releases/platform-tools
- Select: Platform Tools for Windows
- I installed mine in `C:\Android\`

### Vysor
Used for mouse control of the frame's Android UI during setup.
- **Download:** https://www.vysor.io
- Windows desktop app — free tier is sufficient

---

## APKs

### ImmichFrame (required)
The photo frame app that replaces Nixplay. Connects to your ImmichFrame server and displays photos in a fullscreen slideshow.
- **Download:** https://github.com/immichframe/ImmichFrame/releases
- Select the `.apk` file from the latest release
- Confirmed working on Android 7.1.2 (API 25)

### Nova Launcher (optional)
Useful for navigating the Android UI during initial setup. Not needed for final configuration — ImmichFrame replaces it as the launcher.
- **Download:** Search APKMirror or another reputable site for `com.teslacoilsw.launcher`
- Select: arm architecture, Android 7.0+ compatible version
- Can be uninstalled after ImmichFrame is set up

---

## Server Software

### Immich
Self-hosted photo management platform. Used as the photo library backend.
- **Documentation:** https://immich.app/docs
- **Docker Compose setup:** https://immich.app/docs/install/docker-compose
- ML container can be disabled — see guide for details

### ImmichFrame Server
Middleware service that sits between the Android app and Immich.
- **GitHub:** https://github.com/immichframe/ImmichFrame
- **Documentation:** https://immichframe.dev/docs/getting-started/configuration
- Runs as a Docker container alongside Immich

---

## Infrastructure

### VirtualBox
Used to run the Ubuntu VM on Windows for partition operations.
- **Download:** https://www.virtualbox.org/wiki/Downloads
- **Extension Pack:** Download separately from the same page — version must match VirtualBox exactly

### Ubuntu 20.04 Server/Desktop ISO
Used for the VirtualBox VM.
- **Download:** https://releases.ubuntu.com/20.04/

### Cloudflare
Used to expose ImmichFrame publicly via tunnel and secure it with a WAF rule.
- **Cloudflare Tunnel:** https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- **WAF Custom Rules:** https://developers.cloudflare.com/waf/custom-rules/
- Free plan is sufficient for this use case

### Home Assistant Cloudflare Add-on
Used to manage the Cloudflare tunnel from Home Assistant.
- **Add-on:** Search "Cloudflare" in Home Assistant Add-on Store
- **Documentation:** https://github.com/brenner-tobias/ha-addons

---

## References

| Source | URL |
|--------|-----|
| Original W10P jailbreak guide | https://micropipes.com/2025/09/14/jailbreak-a-nixplay-w10p-photo-frame/ |
| Yo-less YouTube walkthrough | https://www.youtube.com/watch?v=TN5errM5UbA |
| ThomasWildeTech ImmichFrame + Cloudflare WAF | https://thomaswildetech.com/blog/2025/12/13/frameo-with-immichframe-an-amazing-gift/ |
| ImmichFrame configuration docs | https://immichframe.dev/docs/getting-started/configuration |
| Cloudflare WAF custom rules docs | https://developers.cloudflare.com/waf/custom-rules/use-cases/require-specific-headers/ |

---

## Apps That Were Tested But Did Not Work

| App | Reason |
|-----|--------|
| Fotoo | Cloud sources (Google Photos, Dropbox) require Google Play Services — not present on this device |
| Aerial Views 1.8.3 | Incompatible with Immich v3 API — album asset fetch fails |
| Google Photos | Requires Google Play Services to authenticate |

---

*All software listed here is the property of their respective owners. This guide does not redistribute any APKs or proprietary software.*
