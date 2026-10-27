---
title: "Ad-Free Media Playback Controlled by Agents"
slug: agent-media-playback
category: guides
tags: ["youtube", "mpv", "yt-dlp", "firefox", "policies", "audio", "bluetooth"]
sources:
- sessions/2026-10-26-minipc-ops-printer-youtube-docker-security-scan.md
- sessions/2026-10-27-deco-backhaul-printer-audio-memory-plan.md
last_updated: "2026-10-27"
version: 1
---

# Ad-Free Media Playback Controlled by Agents

Making a home mini PC play YouTube (and other media) on command from chat agents — without ads, without writing video to disk, and with a reliable audio path to the speakers.

## The `yt-player`: mpv + yt-dlp, Streaming Only

The player is **mpv driven by yt-dlp in pure streaming mode** (nothing saved to disk), exposed to agents through mpv's **IPC socket**:

- commands: play, pause, volume, next, status;
- a `--video` mode routes output to the monitor instead of audio-only;
- yt-dlp handles extraction/format selection, mpv handles decoding and the control surface.

Because control is a Unix socket, any agent (or cron job) can drive playback with a one-line `echo '{ "command": ["set_property", "pause", true] }' | socat - /tmp/mpv-socket` style call — no GUI automation, no browser remote control.

### Installing yt-dlp Properly

The distribution package is chronically outdated (YouTube breaks old extractors fast). Install via `uv tool` instead — and configure yt-dlp to run its JavaScript deciphering on Node rather than on bundled runtimes (`--js-runtimes node`). The session records the install as "`uv tool` + `--js-runtimes node`"; check `yt-dlp --help` for the exact flag placement in your version.

Add a weekly timer to keep it current — an outdated yt-dlp is the single most common "suddenly nothing plays" cause:

```bash
uv tool upgrade yt-dlp
```

## Firefox, Hardened by Policy

For the browser path, uBlock Origin is **force-installed via enterprise policy** rather than relying on manual extension management. On Ubuntu's Firefox snap, create `/etc/firefox/policies/policies.json`:

```json
{
  "policies": {
    "ExtensionSettings": {
      "uBlock0@raymondhill.net": {
        "install_mode": "force_installed",
        "install_url": "https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/addon-607454-latest.xpi",
        "default_area": "menupanel"
      }
    },
    "Permissions": {
      "Autoplay": { "Default": "allow-audio-video" }
    }
  }
}
```

`force_installed` survives profile resets and cannot be uninstalled from the UI — the right posture for a shared, agent-facing machine. The `Autoplay` permission avoids the "first play is silent" trap.

## Silent Testing

To exercise the pipeline without audible output, point `MPV_HOME` at a directory containing an `mpv.conf` with:

```ini
ao=null
vo=null
```

This runs the full yt-dlp → mpv chain (so extraction failures still surface) while muting both audio and video.

## Audio Output: Skip the Onboard Bluetooth

The integrated Bluetooth (MediaTek MT7902) is **not supported by the kernel**. Exact kernel message:

```
Unsupported hardware variant (00007902)
```

Two working alternatives:

1. **USB Bluetooth dongle** with a supported chip (Realtek RTL8761BU works out of the box);
2. **Analog: stereo TRS splitter on the AUX output** feeding two speakers in parallel (tested with a Logitech pair and a Bose SoundLink Mini on AUX input).

Splitter lessons:

- With a splitter there is **one system-level volume** — balance between the two speakers is set on the speakers themselves. (A "mute" speaker is usually just its local volume at minimum.)
- Set the system level explicitly during tests:

```bash
wpctl set-volume @DEFAULT_AUDIO_SINK@ 0.5
```

## Related

- [Hardware and OS Setup](../getting-started/hardware-and-os-setup.md) — base machine and OS.
