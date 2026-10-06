# pYTM5

Scratch repo for a custom PS5 music player. Goal is a player for YouTube Music
(and other streaming sources) that registers with the console's system **Music
playback widget**, so it shows up in the Music card and the Control Center
alongside Spotify, Apple Music and USB media.

**Status: research only.** Nothing built yet. This file exists so the findings
survive; see [open questions](#open-questions) for what's actually blocking.

## what's already solved

Roughly three quarters of the problem is public and doesn't need original work.
No need to re-derive any of this.

### audio output

`libSceAudioOut.so` ships in the payload SDK with ~130 exports. This is a full
audio API, not a stub:

```c
sceAudioOutInit();
sceAudioOutOpen(userId, MAIN, 0, grain, 48000, fmt);
sceAudioOutOutput();
```

`grain` must be one of 256/512/768/1024/…. Opus 5ms frames are 240 samples, so
you accumulate into a legal grain rather than passing 240 directly.

There is also a richer v2 surface worth knowing about: `sceAudioOut2Context*`
(contexts, queue levels, `GetIpcHandle`), `sceAudioOut2MbusSetConnections`,
`sceAudioOutExOpen`, port arbitration and 3D/HRTF speaker-array calls. v1 is
enough to start.

### decoding

- [`ps5-payload-dev/SDL`](https://github.com/ps5-payload-dev/SDL) (branch
  `release-2.30.x-ps5`) is the reference PS5 SDL2 port. Its video backend is what
  the PS5 RetroArch port uses.
- [`tsuramatsu1/retroarch`](https://github.com/tsuramatsu1/retroarch) documents
  prebuilt **SDL2, ffmpeg, freetype, zlib** cross-compiled for PS5, published via
  [`ps5-payload-dev/pacbrew-repo`](https://github.com/ps5-payload-dev/pacbrew-repo).
  That gives AAC/Opus/FLAC/Vorbis decoders without hand-building FFmpeg.
- [`br1cks4t1ts/ps5-moonlight`](https://github.com/br1cks4t1ts/ps5-moonlight)
  uses libopus for audio and FFmpeg for video, with `sceVideodec2` hardware decode
  noted as the later milestone.

Note: `target/include/sys/aac_ioctl.h` in the SDK is *not* a Sony decoder. It's
the inherited FreeBSD AAC driver header
(`$FreeBSD: releng/11.4/sys/sys/aac_ioctl.h`). Dead end — don't get pulled into
it again.

### video output

The v1 registration API fails for anything that isn't a real BIG_APP game slot:

```
sceVideoOutRegisterBuffers -> 0x80290008   // "game slot not assigned"
```

Most projects conclude you need a Sony-signed PKG and call it a dead end. That
conclusion is wrong for us — we already install fake-signed titles. And it's
unnecessary anyway, because the **v2** path works from an ordinary unsigned
payload:

```c
sceSystemServiceHideSplashScreen();                            // REQUIRED
int h = sceVideoOutOpen(0xFF, 0, 0, NULL);                     // 0xFF = SYSTEM user
sceKernelAllocateMainDirectMemory(size, 0x20000, 3, &paddr);  // 3 = WC_GARLIC
sceKernelMapDirectMemory(&va, size, 0x33, 0, paddr, 0x20000);  // 0x33 = CPU+GPU rw
sceVideoOutSetBufferAttribute2(&attr, 0x8000000022000000ULL, 0, w, h, 0, 0, 0);
sceVideoOutRegisterBuffers2(h, 0, 0, vbuf, 2, &attr, 0, NULL); // <- the "2"
sceVideoOutSetFlipRate(h, 0);                                  // AFTER register
```

Three things bite people here:

- `sceSystemServiceHideSplashScreen()` releases the compositor's hold on the flip
  chain. Skip it and `sceVideoOutOpen` appears to succeed but registration fails.
  This is the single most-missed step.
- `sceVideoOutOpen(0xFF, ...)` opens as the SYSTEM user, so it needs no logged-in
  per-user session and a background process can do it.
- The scan-out surface is **tiled, not linear** — 512×128 tiles with an internal
  permutation that SDL recovered into a 128×512 lookup table. You cannot `memcpy`
  into it. A solid colour fill is swizzle-invariant, which makes it a
  swizzle-independent display proof.

Open caveat from the moonlight work: the SDL reference computes `tiles_per_row` as
float `width / 512`, which is only exact for multiples of 512. 1920 is 3.75
tiles. Pad to whole tiles (4 columns, pitch 2048) instead. Unconfirmed on
hardware — suspect this first if frames look sheared or diagonally banded.

### install / signing

Already working on our console: `DRPC00001`, `FAKE02932` and
`IV9999-FAKE00000_00-HOMEBREWLOADER01` all install and run. For reference,
[`br1cks4t1ts/moonlight-ps5`](https://github.com/br1cks4t1ts/moonlight-ps5)
(the sibling repo, not the SDK one) ships `make_pkg.py` + `make_fself.py` and
builds a fake-titled PKG, plus `ps5-pub-tools`, `dump_pkg.c` and `dump_fs.c`.

The SDK sample that matters is
`samples/install_app/payload.c`, which uses
`sceAppInstUtilAppInstallTitleDir(TITLE_ID, "/user/app/", 0)` against a fake
`eboot`.

### prior art for streaming-service UIs

`ps5-payload-dev` already ships 10-foot webapp UIs for streaming services —
`plutotv`, `tv4play`, `svtplay` — plus `tvhp` (a TVHeadend player) and `websrv`
on port 8080, which launches homebrew through the console's own WebKit browser.
This is the fallback route if widget integration proves intractable: it works and
it's precedented, but it gives no system widget integration.

## the actual blocker: PSM is undocumented

Everything music-widget related lives in `libSceSystemService.so`, verified by
scanning every `target/lib/*.so` in the SDK. `libSceAppInstUtil.so` has one
relevant export. Full inventory:

| symbol | notes |
| --- | --- |
| `sceSystemStateMgrEnterMediaPlaybackMode` | entry |
| `sceSystemStateMgrLeaveMediaPlaybackMode` | exit |
| `sceSystemStateMgrTickMusicPlayback` | probably a heartbeat; inferred, unverified |
| `sceSystemStateMgrTickVideoPlayback` / `sceSystemServiceTickVideoPlayback` | separate video path |
| `sceShellCoreUtilPostPsmEventToShellUI` | **carries the metadata** |
| `sceAppMessagingSendMsgToShellUI` / `SendMsgToShellCore` / `ReceiveMsg` | shell messaging |
| `sceSystemServiceDisableMusicPlayer` / `ReenableMusicPlayer` | a kill switch, not registration |
| `sceSystemServiceDisableMediaPlay` / `ReenableMediaPlay` | separate pair, video/media |
| `sceSystemServiceLaunchPlayerDialog`, `sceSystemServiceInitializePlayerDialogParam` | |
| `sceSystemServiceAddLocalProcessForPsmKit`, `GetParentSocketForPsmKit`, `KillLocalProcessForPsmKit` | PSM Kit plumbing |
| `sceKernelGetPsmIntdevModeForRcmgr`, `sceKernelGetDebugMenuModeForPsmForRcmgr` | kernel side, libkernel |
| `sceLncUtilAcquireCpuBudgetOfExtraAudioDevices` | extra-audio budget |
| `sceShellCoreUtilAcquireBgmCpuBudget`, `sceShellCoreUtilIsBgmPlaying` | BGM budget |
| `sceAppInstUtilAppInstallMediaPlayer` | **dedicated media-player install API** |

Two things worth calling out:

- `sceAppInstUtilAppInstallMediaPlayer` sits right next to
  `sceAppInstUtilAppInstallTitleDir` in `libSceAppInstUtil.so`. No header in the
  SDK, no docs, no web hits anywhere. This is very likely how a third-party app
  gets registered as a media player with the ShellUI.
- `sceSystemServiceDisableMusicPlayer` reads like registration but is almost
  certainly a *disable* switch for the built-in player. Don't waste time on it.

There is **no public reverse engineering** of the PSM protocol. Four separate
search angles turned up nothing — no writeup, no sample, no header.

### inferred model

Completely unverified. Worth building probes to confirm each step:

1. Register via `sceAppInstUtilAppInstallMediaPlayer`.
2. On playback start call `sceSystemStateMgrEnterMediaPlaybackMode`; on stop,
   `LeaveMediaPlaybackMode`. This likely also drives the audio mixer /
   `sceAudioOutChangeAppModuleState`.
3. Push metadata with `sceShellCoreUtilPostPsmEventToShellUI` — title, artist,
   album, cover URI, duration, position.
4. Call `sceSystemStateMgrTickMusicPlayback` periodically so the ShellUI knows
   the app is still alive.

Steps 1, 2 and 4 are trial-and-errorable. **Step 3 is the load-bearing unknown** —
the PSM event struct layout. Nothing else blocks the project.

## how to reverse it

Nobody has done this, so we'd be first. Highest yield first:

1. **Capture from the real Media app.** It's on the console and it already drives
   the widget, so valid messages are sitting there. Break on
   `sceShellCoreUtilPostPsmEventToShellUI` under ptrace and dump the argument
   buffer. `elfldr` already bootstraps via ptrace into `SceRedisServer`, so a
   tracer is the same family of technique. By far the best move.
2. **Free experiment via the `Music` folder.** Per psdevwiki, local media is only
   picked up if there's a root folder named exactly `Music` (`Pictures` for
   photos, any folder for video) on a FAT32/exFAT drive. Install a media player,
   drop files there, and watch the system do it with a known-good input — zero RE.
3. **Static RE** of the decrypted system media app / shell UI, reading the caller
   of `PostPsmEventToShellUI` to get the struct directly. Definitive but heavy.
   The SDK ships only import *stubs*, so local `objdump` is useless — this needs
   real modules from a decrypted PUP.
4. **Recover `InstallMediaPlayer`'s signature** by disassembling it in the real
   `libSceAppInstUtil`, not the stub.

## plan

Roughly in order:

- [ ] **Probe A — audio.** Tiny payload: open `sceAudioOut`, output a sine tone.
      Validates audio end-to-end before anything else is built on it.
- [ ] **Probe B — display.** `videotest`-style solid colour cycle through
      `sceVideoOutRegisterBuffers2`. Confirms presentation and pinpoints whether
      the tile swizzle needs padding at 1920.
- [ ] **Probe C — install.** Call `sceAppInstUtilAppInstallMediaPlayer` and see
      what it does. Cheapest way to learn its signature and whether it registers
      anything observable.
- [ ] **Probe D — PSM capture.** ptrace the Media app, dump
      `PostPsmEventToShellUI` args, reconstruct the struct.
- [ ] **Player.** HTTP client for the audio stream, decode, `sceAudioOutOutput`,
      PSM metadata, own UI for browsing/search.

## open questions

- What's the signature of `sceAppInstUtilAppInstallMediaPlayer`? How many args,
  what types? Does it differ from `AppInstallTitleDir` in what it writes?
- What's the PSM event struct? Size, field order, string encoding, whether cover
  art is a URI or an embedded buffer.
- Does the ShellUI require the app to be installed via `InstallMediaPlayer`
  specifically, or will a normally-installed title that calls
  `EnterMediaPlaybackMode` + posts PSM events show up in the widget?
- What are the `sceSystemStateMgrEnterMediaPlaybackMode` preconditions? Does it
  need an active user session, or does it work as SYSTEM like `sceVideoOutOpen`?
- Is `TickMusicPlayback` a heartbeat with a required cadence, or something else?
- Which title ID / app category do we need, and does a fake title ID matter to
  the ShellUI?
- Can ad-free playback work, or does the stream need the official client
  handshake? Untested.

## references

- [`ps5-payload-dev/sdk`](https://github.com/ps5-payload-dev/sdk) — toolchain and
  the local SDK at
  `local/playstation-SecRec/sdk/ps5-payload-sdk`
- [`ps5-payload-dev/SDL`](https://github.com/ps5-payload-dev/SDL) — video backend
  + tile swizzle
- [`br1cks4t1ts/ps5-moonlight`](https://github.com/br1cks4t1ts/ps5-moonlight) —
  `docs/RESEARCH.md` is the single best writeup on PS5 display output
- [`tsuramatsu1/retroarch`](https://github.com/tsuramatsu1/retroarch) — prebuilt
  PS5 SDL2/ffmpeg recipe
- [`br1cks4t1ts/moonlight-ps5`](https://github.com/br1cks4t1ts/moonlight-ps5) —
  PKG/fself tooling
- [psdevwiki Media](https://www.psdevwiki.com/ps5/Media) — local media format
  support and the `Music` folder requirement