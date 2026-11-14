pPlay (PS4 fork)
======

|                            Filer view                             |                            Player view                             |
|:-----------------------------------------------------------------:|:------------------------------------------------------------------:|
| ![](https://github.com/Cpasjuste/pplay/raw/master/screenshot.png) | ![](https://github.com/Cpasjuste/pplay/raw/master/screenshot2.png) |

pPlay is a video player for the Nintendo Switch and the PS4, built on [MPV](https://mpv.io/) (libmpv) and
[libcross2d](https://github.com/Cpasjuste/libcross2d). It plays most popular formats, has subtitles, http/ftp/smb
streaming and a file browser with TMDB covers.

This is a fork of [Cpasjuste/pplay](https://github.com/Cpasjuste/pplay) focused on **making it work properly on a
jailbroken PS4** (tested on a PS4 Slim, HEN 11.52 + GoldHEN v2.4b18.10): hardware H.264 decoding, menus and options that work,
subtitles that behave, and a phone as a remote control. Everything that is not PS4 specific keeps working as upstream.

> Unofficial build. Not affiliated with Sony. Use at your own risk. The pull request that sends these changes to the
> original repository lives in the `upstream-pr` branch.

Download and install (PS4)
----
1. Get the latest `.pkg` and `pplay-*_ps4.zip` from the [Releases](https://github.com/EFelipitowo/pplay/releases) page
   (each push to `master` builds a release `ps4-build-N`).
2. Install the `.pkg` (GoldHEN / HEN package installer).
3. Copy the `data` folder from the zip to `/data/pplay/` on the console (FTP works).
4. Start pPlay. The first run creates `/data/pplay/pplay.cfg`.

Switch: copy the "pplay" folder to the sd card ("/switch/pplay"), as in the original project.

What this fork adds
----
| Area                  | What                                                                                                                                                                                                                  |
|-----------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Video                 | **Hardware H.264 decoding** through the console's `libSceVideodec2` (`h264_orbis`, see [`ffmpeg-orbis/`](ffmpeg-orbis/README.md)). Automatic fallback to software for anything the hardware cannot do (Hi10, HEVC, interlaced, 4K...). |
| Software decoding     | `SW_DECODE_SPEED` (quality / balanced / fast) to trade picture quality for fluidity on Hi10 / HEVC; memory defaults that keep big 1080p files from exhausting the console's RAM.                                       |
| Menus                 | Long lists scroll with the cursor centered and a position counter (`12/48`); a working **Options menu** on PS4.                                                                                                       |
| Subtitles             | External subtitles next to the video, legacy text encodings converted to UTF-8, **PGS / VobSub / DVB (image) subtitles** drawn on the CPU (no more solid blue screen), and **automatic selection** by language order.   |
| Controls              | Fixed OSD rewind buttons (they moved forward), L1/R1 seek ±10 s, **left analog stick** navigation next to the D-pad, a controller switched on after the app started is detected.                                      |
| **Phone remote**      | A web page served by the console: open it on your phone and use it as the controller (handy when the pad turns itself off).                                                                                           |
| Network               | Remote (http / ftp / smb) listings run in a thread with a timeout and can be cancelled with O, instead of freezing the app.                                                                                           |
| Tests and CI          | Host tests for the menus, subtitles, language choice, analog stick, remote protocol/server and the decoder (against a libSceVideodec2 mock). A GitHub Actions workflow builds the `.pkg` and publishes releases.      |

Controls (PS4)
----
| Where   | Button                                                                                                              |
|---------|---------------------------------------------------------------------------------------------------------------------|
| Browser | D-pad or **left stick**: move. X: open. O: back. Left / Right: switch menus and windows.                            |
| Player  | X or D-pad Down: show the OSD. O or D-pad Left: leave full screen. D-pad Right: video / audio / subtitle menu.      |
| Player  | **L1 / R1**: back / forward 10 s. L2: normal speed. R2: faster (x2 each press, up to x100).                         |
| OSD     | Left / Right to select, X to press: -10 min, -1 min, pause, stop, +1 min, +10 min.                                  |

Phone remote control
----
When the controller turns itself off to save battery, the phone can be the controller.

1. Phone (or PC) and PS4 on the same network.
2. Start pPlay: a message shows `http://<console ip>:8080` (also shown while **Options → Phone remote** is open).
3. Open that address in the phone browser. **It must be `http://`, not `https://`** (the server has no certificate;
   if the browser forces https, turn off "always use secure connections" / HTTPS-only mode).
4. You get a D-pad, OK, Back, Start, play / pause, -10 s / +10 s, +-1 min, +-10 min, stop, speed and the title and progress of
   the video.

The buttons are pressed exactly like the controller ones, so everything works: browser, menus and player.
It is on by default; **Options → Phone remote** turns it off (`REMOTE = "off"`). The port is `REMOTE_PORT` (default 8080; if it
is taken the next ones are tried and the address shown is the real one). There is **no password**: anyone on your network who
knows the address can press those buttons (nothing else: no files, no settings), so turn it off on shared networks.
The server runs in its own thread and writes `/data/pplay/remote.log`; read it by FTP if the phone cannot connect.
Some routers isolate WiFi clients from each other, which blocks it.

Subtitles
----
* Text subtitles (srt, ass...) and image subtitles (PGS, VobSub, DVB) work. Image subtitles are drawn by the CPU (the console's
  libmpv has only five precompiled shaders), which can cost some fluidity on heavy files.
* **Automatic selection** (`SUB_AUTO = "on"`): when a video loads, the track is chosen by `SUB_LANGS`, an order of
  preference. Default `"es-419,es,en,en-GB"` = Latin American Spanish, Spanish, English, English (UK). Tracks are told apart by
  language code and title ("Spanish (Latin America)" vs "Spanish (Spain)"). Full tracks are preferred to forced / signs-only
  ones, text to picture subtitles, normal to SDH, and commentary tracks are never chosen. Change the list in the Options menu
  or in `pplay.cfg` (examples: `"pt-br,pt,en"`, `"en"`).
* External files next to the video (`movie.es.srt`, `movie.lat.srt`...) are found, and old encodings (Windows-125x, ISO-8859-x...)
  are converted to UTF-8.

Hardware decoding
----
H.264 8-bit 4:2:0 progressive video up to 1080p is decoded by the console's hardware (the CPU is left free for audio, subtitles and the
UI). Everything else is decoded in software automatically. Set it in **Options → HW decoding** or `HWDEC`:

| Value  | Meaning                                                                                                  |
|--------|----------------------------------------------------------------------------------------------------------|
| `auto` | Default. Hardware when possible, software otherwise.                                                     |
| `no`   | Always software.                                                                                         |
| `force`| Insist on the hardware path. **Some `.mp4` files close the app in this mode**; leave `auto` unless testing. |

**Safety guard:** the first seconds of every hardware-decoded playback are guarded. If the console stops during them, the next start turns
hardware decoding off by itself (it creates `/data/pplay/hwdec.disabled` and shows a notice) instead of crashing again. Delete that file, or
set `HWDEC = "force"`, to try again. Details, design and limits:
[`ffmpeg-orbis/README.md`](ffmpeg-orbis/README.md).

Configuration (`/data/pplay/pplay.cfg`)
----
| Option             | Default                  | Meaning                                                                      |
|--------------------|--------------------------|------------------------------------------------------------------------------|
| `NETWORK`          | `http://samples.ffmpeg.org/` | http / ftp server shown in the browser (`ftp://user:password@ip:port/`)  |
| `HOME_PATH`        | data folder              | Starting folder                                                              |
| `HWDEC`            | `auto`                   | `auto`, `no`, `force`                                                        |
| `SW_DECODE_SPEED`  | `quality`                | `quality`, `balanced`, `fast` (software decoding)                            |
| `SUB_AUTO`         | `on`                     | Choose the subtitles when a video loads                                      |
| `SUB_LANGS`        | `es-419,es,en,en-GB`     | Order of preference for the automatic choice                                 |
| `TMDB_LANGUAGE`    | `en-US`                  | Language of the TMDB covers and synopsis                                     |
| `REMOTE`           | `on`                     | Phone remote control                                                         |
| `REMOTE_PORT`      | `8080`                   | Port of the remote page (1024-65535)                                         |

Most of them are also in the **Options** menu (main menu, LEFT on it). `HW decoding` and `SW speed` apply the next time pPlay starts.

As pPlay uses [MPV](https://mpv.io/), you can put a file named `mpv.conf` in the `mpv` folder; see
[`ffmpeg-orbis/mpv.conf.example`](ffmpeg-orbis/mpv.conf.example) and the mpv manual
([configuration files](https://mpv.io/manual/master/#configuration-files), [options](https://mpv.io/manual/master/#options)).
Without one, pPlay limits mpv's demuxer cache to 64 MiB ahead / 16 MiB behind and does not load embedded fonts, to protect the
console's memory.

Known limits
----
* Hardware decoding is only for H.264; Hi10 and HEVC use the CPU (try `SW_DECODE_SPEED = "fast"` for 10-bit 1080p).
* `HWDEC = "force"` can close the app with some `.mp4` files (see above).
* With the hardware decoder, the last 1-3 pictures of a file may not be shown.
* The phone remote has no password and no https.
* Image subtitles cost CPU.

Build
----
The PS4 build uses [OpenOrbis](https://github.com/OpenOrbis/OpenOrbis-PS4-Toolchain) through [PacBrew](https://github.com/PacBrew).
The easiest way to see the exact steps is the CI workflow, [`.github/workflows/ps4.yml`](.github/workflows/ps4.yml): it installs the
toolchain, rebuilds FFmpeg with the hardware decoder (`ffmpeg-orbis/build-ffmpeg.sh`), builds the `.pkg` and publishes a release.

```
git clone --recursive https://github.com/EFelipitowo/pplay
mkdir cmake-build-ps4 && cd cmake-build-ps4
OPENORBIS=/opt/pacbrew/ps4/openorbis cmake -G "Unix Makefiles" -DPLATFORM_PS4=ON -DCMAKE_BUILD_TYPE=Release ..
make pplay_ps4_release
```
`pscrap/` needs a `tmdb.key` file with a [TMDB](https://www.themoviedb.org/) API key (CI uses the repository secret `TMDB_API_KEY`; without a real key the
app runs and only the covers / synopsis are missing).

Tests (on a PC, no console needed): `./tests/menu/run.sh`, `./tests/subpref/run.sh`, `./tests/subtitles/run.sh`, `./tests/stick/run.sh`,
`./tests/remote/run.sh`, `./ffmpeg-orbis/test/run.sh`.

Branches
----
* `master`: the fork's main line (everything above, with the CI workflow).
* `upstream-pr`: the same changes without the fork-only workflow, to be proposed to [Cpasjuste/pplay](https://github.com/Cpasjuste/pplay).

Usage (Switch, from the original project)
-----
- Use left/right to switch menu's and windows... select a media and... have fun.
- When playing a media, you can press the "DOWN" or "A" button to bring the UI, the "R" shoulder for fast forward ("L" button to stop), and "LEFT" to access video, audio and subtitles selection.
- pPlay can stream media from an http server with directory listing enabled, or an ftp server: edit `pplay.cfg` (`NETWORK = "http://samples.ffmpeg.org/";` or `NETWORK = "ftp://user:password@ip:port/";`).
- While 720p and 1080p medias should be fine, you can unlock all the Switch cpu speed for heavy medias in the main options menu (LEFT button).

Credits and license
----
pPlay is by [Cpasjuste](https://github.com/Cpasjuste) (see `LICENSE`). The PS4 work in this fork was done with the help of Claude (Anthropic's
coding assistant). The hardware decoder follows the approach of the open source PS4 streaming projects that use `libSceVideodec2`.
