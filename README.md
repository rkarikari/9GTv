# 9GTv

Android terrestrial digital TV (DVB-T / DVB-T2) player for RTL-SDR dongles
(RTL-SDR Blog **V3**, R820T2 tuner), connected via USB-OTG.

## Features

- **Live DVB-T / DVB-T2 playback** of whatever your antenna picks up,
  decoded by AndroidX Media3 (ExoPlayer) directly from the tuner's raw
  MPEG-TS transfer socket -- no re-encoding. The player's standard OSD
  next/previous buttons switch to the next/previous stored channel
  (wrapping around at the ends) instead of their normal ExoPlayer-playlist
  behavior. Under the hood this closes and reopens the driver session for
  each switch (the bundled driver only supports one transfer-socket client
  per session, so it can't be re-tuned in place the way the IPTV
  restream's channel switching can) -- since the USB device is already
  open/permitted, this is fast and doesn't show a picker dialog. The
  buttons only appear/enable when more than one channel is stored, and
  are disabled while a recording is in progress. A channel that's
  rejected outright (e.g. an unsupported tune) or never achieves signal
  lock within a few seconds (temporarily off air, a stored channel
  that's no longer broadcasting, etc.) is automatically skipped in favor
  of the next one in the same direction, rather than leaving the screen
  stuck or erroring out on a single bad station -- cheaply, on the same
  driver session, when the failure happened before playback ever started,
  falling back to a full session reopen only when it didn't.
- **Channel scanning** across a configurable frequency range: tries DVB-T2
  first at each frequency and falls back to DVB-T, storing every channel
  with a locked signal and a discoverable video PID.
- **Recording** the currently-playing channel to a raw `.ts` file (the
  exact bytes handed to ExoPlayer, so recordings play back through the
  same TS path as live channels). Every recording is started with a
  max-duration and max-size safety limit, like a hardware PVR, so an
  unattended recording can't silently fill the disk. Recordings can be
  written to app-private storage or to a user-picked folder (e.g. a USB
  drive or SD card) via Storage Access Framework, with automatic fallback
  to app-private storage if that folder's permission is ever revoked.
  A dedicated **Recordings** screen (reachable from either the main screen
  or the player's diagnostic panel) lists, plays back, and deletes past
  recordings, and lets you change the storage location and limits.
- **On-device diagnostic panel**, shown during scanning and live playback,
  with a fixed set of tiles (control socket, tune request, tuner lock/
  signal, transfer socket, PID filter, TS data flow, raw PSI content,
  ExoPlayer state, tracks/errors, recording) that update live from every
  stage of the tune -> transfer-socket -> extractor -> renderer pipeline.
  Meant to let a black-screen report be diagnosed from one glance instead
  of a logcat capture.
- **IPTV restreaming to other devices on the same network** -- turns the
  tuner into a small IPTV headend so a smart TV, VLC, Kodi/Jellyfin, or
  another IPTV box can watch the same live channel without installing
  9GTv itself. Two independent delivery paths run at the same time from
  the same PID-filtered byte stream, started together from the single
  "Start IPTV server" control:
  - **HTTP + M3U playlist** (`http://<this device's IP>:8080/playlist.m3u`,
    each entry pointing at `/stream.ts?ch=<index>`): works with any player
    that speaks plain HTTP, and lets a multi-entry-playlist-aware player
    like VLC switch which 9GTv channel is live from its own playlist UI or
    next/previous-track controls -- the app re-tunes the dongle in place,
    no need to return to the 9GTv screen. Only one HTTP client can be
    attached to the live stream at a time (the driver's transfer socket
    accepts exactly one connection per session).
  - **UDP multicast** (`udp://@239.9.7.84:5004`, fixed group/port, join
    from VLC's Open Network Stream): any number of LAN devices can join at
    once, since fan-out happens in the network's own multicast
    routing/switching rather than in this app. Multicast performance is
    inherently at the mercy of the Wi-Fi network it runs on -- most access
    points transmit multicast frames at a fixed, low, unacknowledged
    "basic rate" rather than each client's normal negotiated link rate, so
    expect noticeably worse and loss-prone playback over Wi-Fi compared to
    HTTP, especially on a busy or weak-signal network. A wired/Ethernet
    path (or an AP with a configurable multicast rate and IGMP snooping
    enabled) will perform far better than typical Wi-Fi.

## How it actually works (read this first)

RTL-SDR's normal "SDR mode" (`librtlsdr` / `rtl_sdr` / `rtl_tcp`) tops out
around 3 MHz of instantaneous bandwidth on the host side. DVB-T and DVB-T2
need 6-8 MHz. So generic SDR IQ capture **cannot** receive real digital
terrestrial broadcasts -- this is a hardware/USB throughput limit, not a
software one.

What actually works is the RTL2832U chip's **built-in hardware DVB-T/T2
demodulator**, accessed through a dedicated driver. That driver is now
**bundled directly into 9GTv** as three vendored Gradle library modules,
sourced verbatim from AndroidDvbDriver (GPLv2+):

```
RTL-SDR dongle (USB-OTG)
        │  (raw USB, hardware DVB-T/T2 demod happens on-chip)
        ▼
:usbxfer   -- native (JNI) bulk USB transfer glue
:drivers   -- R820T2 tuner + RTL2832U demodulator drivers
:dvbservice -- DeviceChooserActivity + DvbService: opens the tuner and
               exposes 2 local TCP ports, all in-process
  vendored from https://github.com/signalwareltd/AndroidDvbDriver (GPLv2+)
        │  in-process activity call -> returns 2 local TCP ports
        │  (control port: tune/status/PID-filter commands)
        │  (transfer port: raw MPEG-2 transport stream)
        ▼
com.radiosport.ninegtv.app (this app's own code)
  - DvbDriverContract.kt  -- launches the bundled DeviceChooserActivity
  - DvbControlSocket.kt   -- byte-exact client for the control protocol
  - DvbTsDataSource.kt    -- Media3 DataSource reading the TS socket
  - PsiParser.kt          -- minimal PAT/PMT/SDT parser, scan-time only
  - ExoPlayer/Media3 1.11 -- does ALL real demuxing/decoding of the TS
  - IptvServer.kt         -- optionally fans the same TS bytes out over
                             HTTP and UDP multicast to other LAN devices
  - RecordingTap.kt / RecordingSession.kt -- optionally taps the same TS
                             bytes to a file
```

9GTv does not implement any USB, tuner, or DVB-T/T2 demodulation code
itself, and does not implement MPEG-TS demuxing or H.264/HEVC/AAC/AC-3
decoding itself. Those are handled by, respectively, the bundled
AndroidDvbDriver modules and by AndroidX Media3 (ExoPlayer). Only the
*distribution model* changed: the driver used to live in a separately
installed app talking over a `dtvdriver://` intent; it now compiles
straight into this APK and is reached via an in-process activity call
with the exact same TCP wire contract, so no separate driver install step
is needed anymore.

## Requirements

- An Android device with USB Host / OTG support.
- An RTL-SDR V3 dongle + a USB-OTG adapter/cable + a TV antenna.
- That's it -- the driver ships inside the 9GTv APK, so there's no
  separate "DVB-T Driver" app to install alongside it anymore.
- For the optional IPTV restream/multicast features: a Wi-Fi or Ethernet
  network shared with whatever device you want to watch on. No extra
  hardware or accounts -- both paths are served directly by the app.

## Driver license note

The bundled `:usbxfer`, `:drivers`, and `:dvbservice` modules are copied
verbatim (source, unmodified) from
https://github.com/signalwareltd/AndroidDvbDriver and remain licensed
GPLv2+ under their original copyright (Signalware Ltd). Because 9GTv now
links this GPLv2+ code directly into its own binary rather than talking
to it as a separate process, the combined APK is itself subject to
GPLv2+'s terms -- see the License section below and the upstream
project's COPYING file before redistributing.

## Building

The Gradle wrapper (`gradlew` / `gradlew.bat` / `gradle/wrapper/`) is
bundled in this repo and pinned to Gradle 8.12
(`gradle/wrapper/gradle-wrapper.properties`), so no separate Gradle
install is required.

**Option A -- Android Studio (recommended)**
Just open this folder in Android Studio and build/run directly using the
IDE's bundled Gradle.

**Option B -- command line**
```
./gradlew assembleDebug
```

© RNK 9G5AR RadioSport
