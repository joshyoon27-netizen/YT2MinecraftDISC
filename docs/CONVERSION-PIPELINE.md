# yt2disc — the conversion pipeline, exactly

This is the reference for **what the converter actually runs and writes**: the
exact `ffmpeg` argument vector, the Ogg it produces, and the byte-for-byte
layout of the `.mcaddon` the in-game player loads. It exists so the whole
pipeline can be reproduced with external tooling — plain `ffmpeg` and `zip` —
without reading `webapp/converter.py` and `webapp/packs.py` line by line.

Everything here was read off the shipped source and then **verified by running
the real pipeline** (`ffmpeg 9.0.2` on Windows). The worked example that results
is committed at [`examples/yt2disc-example.mcaddon`](../examples/yt2disc-example.mcaddon);
the commands that rebuild it are in [Reproducing it externally](#reproducing-it-externally).

The two files this documents:

| File | Responsibility |
| --- | --- |
| `webapp/converter.py` | chooses the options, builds the `ffmpeg` argv, runs it, parses progress, names the file |
| `webapp/packs.py` | turns the finished Ogg into the `.mcaddon`: two manifests, `sound_definitions.json`, the menu script, the icon |

`packs.py` never calls `ffmpeg`. By the time it runs, the audio is already an
Ogg the resource pack can read.

---

## 1. The two entry points

`converter.py` splits the work in two so the command line is testable on its own:

```python
converter.build_command(ffmpeg, source, destination, options) -> list[str]
```

A **pure** function: given the ffmpeg path, the input, where the output goes and
the normalised options, it returns the exact argument list. No process is
started. This is the function an external tool should mirror.

```python
converter.convert(source, destination, options, log=None, progress=None,
                  ffmpeg=None, title=None, timeout=None) -> Path
```

The job: probe the input's length, build the command, run it through
`core.run_ffmpeg`, reject an empty result, and — for the pack format — hand the
Ogg to `packs.build_addon`.

`options` is whatever `converter.normalize_options(raw)` returns; it always
carries `format`, `extension`, `mime`, `lossy`, `pack`, `bitrate`,
`sample_rate`, `channels`, `start`, `end`, `normalize`.

---

## 2. The `ffmpeg` argument vector

### 2.1 The fixed prefix

Every conversion starts with the same seven arguments, in this order
(`converter.build_command`, lines 297–309):

```text
<ffmpeg> -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats
```

| Argument | Why |
| --- | --- |
| `-y` | overwrite the part file without asking |
| `-hide_banner` | keep the console quiet |
| `-nostdin` | never read the console — nothing can block on a prompt |
| `-loglevel warning` | only warnings and errors reach the log |
| `-progress pipe:1` | emit machine-readable `out_time=…` lines on stdout |
| `-nostats` | do not also print the human progress line |

`-progress` is independent of `-loglevel`, so the job still gets a percentage
while the console stays quiet — that is the whole reason both are present.

### 2.2 The full template

```text
<ffmpeg> -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats \
  [ -ss <start> ] \
  -i <source> \
  [ -t <duration> ] \
  -vn \
  [ -ar <sample_rate> ] \
  [ -ac <1|2> ] \
  [ -af loudnorm=I=-16:TP=-1.5:LRA=11 ] \
  <codec args> \
  <destination>
```

Every bracketed group is optional and only appears when the matching option is
set. The parts, in the order `build_command` appends them:

| Position | Argument | Condition | Value |
| --- | --- | --- | --- |
| 1 | `-ss <seconds>` | `start` is not `None` | `f"{start:.3f}"` — e.g. `30.000` |
| 2 | `-i <source>` | always | the input file path |
| 3 | `-t <seconds>` | `end` is not `None` | `f"{max(0.0, end - (start or 0.0)):.3f}"` — a **duration**, not a stop-time |
| 4 | `-vn` | always | drop the picture; this is a music file |
| 5 | `-ar <rate>` | `sample_rate` is not `0` | one of `22050`, `44100`, `48000` |
| 6 | `-ac 1` | `channels == "mono"` | — |
| 6 | `-ac 2` | `channels == "stereo"` | — |
| 7 | `-af loudnorm=I=-16:TP=-1.5:LRA=11` | `normalize` is true | EBU R128 loudness target |
| 8 | codec args | always | see the table below |
| 9 | `<destination>` | always | **the last argument** — `packs.py` and the browser both rely on this |

Two details the source calls out, and that matter if you are reproducing it:

* **`-ss` goes *before* `-i`.** That makes ffmpeg seek instead of decoding
  everything up to the start point first.
* **`-t` is computed as `end - start`**, so a `30 → 90` trim is
  `-ss 30.000 … -t 60.000`, not `-t 90.000`.

### 2.3 Codec arguments, per output format

`OUTPUT_FORMATS` in `converter.py` (lines 58–120) is the one table. `{bitrate}`
is the only placeholder, and only the lossy formats use it:

| `format` | Label | Extension | `args` (exact) | Lossy |
| --- | --- | --- | --- | --- |
| `ogg` **(default)** | Ogg Vorbis | `.ogg` | `-c:a libvorbis -b:a {bitrate}` | yes |
| `opus` | Opus | `.opus` | `-c:a libopus -b:a {bitrate}` | yes |
| `mp3` | MP3 | `.mp3` | `-c:a libmp3lame -b:a {bitrate}` | yes |
| `m4a` | AAC (m4a) | `.m4a` | `-c:a aac -b:a {bitrate}` | yes |
| `wav` | WAV | `.wav` | `-c:a pcm_s16le` | no |
| `flac` | FLAC | `.flac` | `-c:a flac` | no |
| `pack` | Minecraft music player | `.mcaddon` | `-c:a libvorbis -b:a {bitrate}` | yes |

`pack` is not a codec but a *deliverable*: the audio is encoded exactly as Ogg
Vorbis (the same `libvorbis` args), and then `packs.py` wraps it, with a
behavior pack, into the `.mcaddon`. `"pack": True` on that entry is what tells
`convert()` the zip is the answer and the Ogg is only a step on the way.

* **Bitrate** — `BITRATES = ("96k", "128k", "192k", "256k", "320k")`, default
  **`192k`**. A value outside the set falls back to `192k`, and a **lossless**
  format (`wav`, `flac`) zeroes it out, so no `-b:a` is emitted at all.
* **Sample rate** — `SAMPLE_RATES = (0, 22050, 44100, 48000)`; `0` (the default)
  means "leave it alone", so no `-ar` appears.
* **Channels** — `CHANNEL_CHOICES = ("auto", "mono", "stereo")`; `auto` (the
  default) emits nothing, the others emit `-ac 1` / `-ac 2`.
* **Normalise** — `LOUDNORM_FILTER = "loudnorm=I=-16:TP=-1.5:LRA=11"`.


### 2.4 Worked examples (verified output)

For a source `C:\uploads\My Song.mp4`, `build_command` returned exactly:

```text
# ogg, default 192k
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats -i "C:\uploads\My Song.mp4" -vn -c:a libvorbis -b:a 192k out\my_song.ogg

# opus 128k
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats -i "C:\uploads\My Song.mp4" -vn -c:a libopus -b:a 128k out\my_song.opus

# mp3 320k
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats -i "C:\uploads\My Song.mp4" -vn -c:a libmp3lame -b:a 320k out\my_song.mp3

# m4a 192k
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats -i "C:\uploads\My Song.mp4" -vn -c:a aac -b:a 192k out\my_song.m4a

# wav (lossless: no -b:a)
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats -i "C:\uploads\My Song.mp4" -vn -c:a pcm_s16le out\my_song.wav

# flac (lossless: no -b:a)
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats -i "C:\uploads\My Song.mp4" -vn -c:a flac out\my_song.flac

# pack, every optional flag on, trimmed 30s -> 90s
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats \
  -ss 30.000 -i "C:\uploads\My Song.mp4" -t 60.000 \
  -vn -ar 44100 -ac 2 -af loudnorm=I=-16:TP=-1.5:LRA=11 \
  -c:a libvorbis -b:a 128k out\my_song-from30s-to90s.mcaddon
```

### 2.5 What `convert()` actually writes (the part files)

`build_command` is pure, so the destination it is *given* is the final answer.
`convert()` never encodes straight to that name — it encodes to a **part file
that keeps the real suffix**, because ffmpeg picks its muxer from the extension
alone, and only then moves it into place with `os.replace`:

| Format | `build_command` destination | `convert()` encodes to | `convert()` renames to |
| --- | --- | --- | --- |
| `ogg` / `opus` / `mp3` / `m4a` / `wav` / `flac` | `my_song.ogg` | `my_song.part.ogg` | `my_song.ogg` |
| `pack` | (`my_song.mcaddon` — unused) | `my_song.part.ogg` | `my_song.ogg`, then zipped to `my_song.mcaddon` |

For the pack format the sequence is:

1. `audio = <destination stem>.ogg` — e.g. `my_song.ogg`
2. `tmp   = <audio stem>.part<audio suffix>` — e.g. `my_song.part.ogg`
3. run ffmpeg (the argv `build_command` produced, with `tmp` as the destination)
4. `os.replace(tmp, audio)` — the Ogg now exists on disk
5. `packs.build_addon(<destination>, [{slug, name, audio, duration}])` writes the
   `.mcaddon` beside it

> **Note, verified:** the intermediate `<stem>.ogg` is *left on disk* beside the
> `.mcaddon`. A pack conversion therefore produces **two** files —
> `yt2disc-example.ogg` **and** `yt2disc-example.mcaddon`. The `.ogg` is the one
> that goes inside the zip.

The `.mcaddon` itself is written to `<destination>.part` first and `os.replace`d
into place, so a half-written addon can never be handed to a download link.

### 2.6 Probing the length

`convert()` calls `core.probe_duration(source, ffmpeg, strict=True)` once, both
to size the progress bar and to catch a trim pointing off the end of the file:

```text
ffmpeg -hide_banner -i <file>
```

No `ffprobe`. The duration is scraped from ffmpeg's own stderr banner with
`_DURATION_RE = re.compile(r"Duration:\s*(\d+):(\d\d):(\d\d(?:\.\d+)?)")`, read
as `HH:MM:SS.ss`. The subprocess is capped at 60 s. `strict=True` turns every
"no answer" (missing file, timeout, no `Duration:` line) into a `core.InputError`
that names the file; `strict=False`, used by the desktop player, just returns
`None`.


### 2.7 Reading progress

The `-progress pipe:1` flag makes ffmpeg write keyed lines on **stdout**; every
line is handed to the caller's `log` callback, and the ones that look like a
timestamp drive the bar (`converter.progress_seconds`):

```text
out_time=00:00:01.230000
out_time_ms=1230000      # microseconds, despite the name
```

The **text** form is tried first, because `out_time=00` would otherwise match a
numeric pattern and report zero for ever. Two regexes, in order:

```python
_OUT_TIME_TEXT_RE  = re.compile(r"out_time=(\d+):(\d\d):(\d\d(?:\.\d+)?)")
_OUT_TIME_MICROS_RE = re.compile(r"out_time(?:_ms|_us)=(\d+)")
```

The percentage is `min(99.0, max(0.0, seen / total * 100.0))`, where `total` is
`reported_total()`. `convert()` reports a final `100.0` itself once the file is
in place, so the bar always lands on 100.

`reported_total(full, options)` is the *output* length:

* no trim → the input's own length;
* a trim → `max(0.0, (full if no end else min(end, full)) - start)`;
* an unmeasurable input with only an `end` → `end - start`.

### 2.8 Naming rules

Two helpers from `core`/`converter` decide the file names:

**`core.slugify(text)`** (used for the output stem *and* the pack slug/sound id):
NFKD-normalise → drop non-ASCII → lower-case → every run of non-`[a-z0-9]`
becomes `_` → collapse runs → strip leading/trailing `_` → prefix `x` if it
starts with a digit → truncate to 48 chars → empty becomes `"playlist"`.

**`converter.output_name(source_name, options)`**:

| Input | Options | Result |
| --- | --- | --- |
| `My Song.mp3` | default (ogg) | `my_song.ogg` |
| `My Song.mp3` | `start=30, end=90` | `my_song-from30s-to90s.ogg` |
| `x.wav` | `start=0.5` | `x-from0.5s.ogg` |
| `My Song.mp3` | `format=pack` | `my_song.mcaddon` |

The trim is written into the name with `:g` formatting (`30.0` → `30`, `0.5` →
`0.5`), so several cuts of one song sit apart in a folder instead of colliding.

---

## 3. Server and browser run the *same* command

The browser is not a special case in `converter.py` — that file has no browser
branch at all. `browser_ffmpeg.install()` **rebinds three names on the `core`
module**:

```python
core.run_ffmpeg    = run_ffmpeg      # ffmpeg.wasm instead of subprocess
core.probe_duration = probe_duration
core.find_binary    = find_binary
```

Because `converter.py` looks those names up on `core` every time it calls them,
`converter.convert()` then runs on FFmpeg-WASM **with not one line of it
changed**. The argv `build_command` returns is therefore *identical* on a server
and in a tab. The only mechanical differences:

| | Server (`core.run_ffmpeg`) | Browser (`browser_ffmpeg.run_ffmpeg`) |
| --- | --- | --- |
| the binary | a real `ffmpeg` found on disk | `@ffmpeg/core@0.12.10`, built to WebAssembly, compiled with `libvorbis`, `libopus`, `libmp3lame` |
| `argv[0]` | the ffmpeg path | **dropped** — the program is built in (`cmd[1:]`) |
| input path | used as-is | copied into ffmpeg's FS: `/yt2disc-work/fN<suffix>` |
| destination | used as-is | rewritten to `/yt2disc-work/fM<suffix>`, copied back after the run |
| how path arguments are found | n/a | an argument naming an existing file is an input; the **last** argument is the destination (`build_command` always ends with it) |
| exit code + tail | from `Popen` | from the synchronous `exec()` return and `setLogger` |
| `timeout` | kills the child past the deadline | accepted and ignored — the wasm call is synchronous, and the tab is its own ceiling |

The CDN URLs, pinned:

```text
https://cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.10/dist/esm/ffmpeg-core.js
https://cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.10/dist/esm/ffmpeg-core.wasm
https://cdn.jsdelivr.net/npm/@ffmpeg/core@0.12.10/dist/umd/ffmpeg-core.js   (fallback)
```

`core.find_binary` returns `Path("ffmpeg.wasm")` in a browser — a fiction, since
there is no binary; the value only ever reaches a log line.

**Byte-identical output is not guaranteed** (the wasm core is a different
ffmpeg build than any given host's), but the *command* is the same, the *format*
is the same, and `probe_duration` and the progress parsing reuse `core`'s own
regexes, so the numbers cannot drift. See `webapp/README.md` for the full
browser story.


---

## 4. The `.mcaddon` layout

A `.mcaddon` is **a ZIP archive** (`zipfile.ZIP_DEFLATED`) holding **two packs**,
one folder each. The folder names come from `packs.pack_folder_names(pack_name)`
— `slugify(pack_name)` plus the `_RP` / `_BP` suffix — so the default pack name
`yt2disc music player` becomes the stem `yt2disc_music_player`.

```text
<name>.mcaddon                       (a zip; default name: yt2disc music player)
├── yt2disc_music_player_RP/         ← resource pack: the sound + its definition
│   ├── manifest.json
│   ├── sounds/
│   │   ├── sound_definitions.json
│   │   └── <slug>.ogg               ← one per track (the converted Ogg)
│   └── pack_icon.png
└── yt2disc_music_player_BP/         ← behavior pack: the menu script
    ├── manifest.json
    ├── scripts/
    │   ├── main.js                  ← the player (command + form + playSound)
    │   └── tracks.js                ← the generated song list
    └── pack_icon.png
```

Every entry is written by `packs.build_addon` (lines 483–502). The two icons are
the **same bytes** — `packs.pack_icon()`, a 128×128 PNG drawn with `zlib` and
`struct` alone (no Pillow, because this also runs on Pyodide).

Written in this order (so the zip's entry order is deterministic):

| # | Entry | Content |
| --- | --- | --- |
| 1 | `<RP>/manifest.json` | `resource_manifest()` |
| 2 | `<RP>/sounds/sound_definitions.json` | `sound_definitions()` |
| 3 | `<RP>/sounds/<slug>.ogg` | the converted Ogg, one per track |
| 4 | `<RP>/pack_icon.png` | `pack_icon()` |
| 5 | `<BP>/manifest.json` | `behavior_manifest()` |
| 6 | `<BP>/scripts/main.js` | `main_script()` |
| 7 | `<BP>/scripts/tracks.js` | `tracks_script()` |
| 8 | `<BP>/pack_icon.png` | `pack_icon()` (same bytes as #4) |

### 4.1 The version constants (one place)

All in `webapp/packs.py` (lines 56–73); they decide whether the game loads the
addon, so they are not buried in a template:

| Constant | Value | Meaning |
| --- | --- | --- |
| `MANIFEST_FORMAT_VERSION` | `2` | `format_version` in both manifests |
| `MIN_ENGINE_VERSION` | `[1, 21, 0]` | `min_engine_version` — the addon needs Minecraft 1.21+ |
| `PACK_VERSION` | `[1, 0, 0]` | the pack's own `version` |
| `SOUND_FORMAT_VERSION` | `"1.14.0"` | `format_version` of `sound_definitions.json` |
| `SERVER_MODULE_VERSION` | `"2.0.0"` | `@minecraft/server` dependency |
| `SERVER_UI_MODULE_VERSION` | `"2.0.0"` | `@minecraft/server-ui` dependency |
| `NAMESPACE` | `"yt2disc"` | prefixes every sound id |
| `COMMAND_NAME` | `"yt2disc:music"` | the custom command |
| `MENU_SCRIPT_EVENT` | `"yt2disc:menu"` | the `/scriptevent` fallback |
| `DEFAULT_PACK_NAME` | `"yt2disc music player"` | folder stem + manifest name |
| `SOUND_CATEGORY` | `"music"` | the `category` in each sound definition |
| `ICON_SIZE` | `128` | the pack icon, in pixels |

`@minecraft/server` / `@minecraft/server-ui` **2.x** are the current *stable*
modules, so the form UI needs no Beta APIs experiment; the custom command is
stable too.


### 4.2 The manifests

Both are `json.dumps(..., indent=2)`. These are the **real** files from the
committed example (folder stem `yt2disc_music_player`, one track named
`Example Tone`):

`yt2disc_music_player_RP/manifest.json`:

```json
{
  "format_version": 2,
  "header": {
    "name": "yt2disc music player",
    "description": "1 song(s) for the in-game music player.  Run /yt2disc:music to open the menu.",
    "uuid": "b1ee3192-9d3b-5929-8d64-46e5a78bae9c",
    "version": [1, 0, 0],
    "min_engine_version": [1, 21, 0]
  },
  "modules": [
    {
      "type": "resources",
      "description": "yt2disc music player sounds",
      "uuid": "ff43dad1-0e50-51b5-b9c9-206afdac64eb",
      "version": [1, 0, 0]
    }
  ]
}
```

`yt2disc_music_player_BP/manifest.json`:

```json
{
  "format_version": 2,
  "header": {
    "name": "yt2disc music player (behavior)",
    "description": "1 song(s) for the in-game music player.  Run /yt2disc:music to open the menu.",
    "uuid": "cf91b1e8-9284-5616-a288-ae560045d042",
    "version": [1, 0, 0],
    "min_engine_version": [1, 21, 0]
  },
  "modules": [
    {
      "type": "data",
      "description": "yt2disc music player behavior",
      "uuid": "c8ef02e5-2b99-5d5a-b435-09f9b9949cbc",
      "version": [1, 0, 0]
    },
    {
      "type": "script",
      "language": "javascript",
      "entry": "scripts/main.js",
      "description": "yt2disc music player menu script",
      "uuid": "17b0bb6a-70db-56cc-bb1d-990aa7da9f4e",
      "version": [1, 0, 0]
    }
  ],
  "dependencies": [
    { "module_name": "@minecraft/server", "version": "2.0.0" },
    { "module_name": "@minecraft/server-ui", "version": "2.0.0" }
  ]
}
```

The `script` module plus the two `@minecraft/server*` `dependencies` are what let
a pack run JavaScript and draw a form; without them Minecraft treats the `.js`
as decoration.

If no `description` is passed, `build_addon` generates
`"<N> song(s) for the in-game music player.  Run /yt2disc:music to open the menu."`.

### 4.3 Stable UUIDs

A pack that changes its UUIDs on every build is a *different* pack to the game —
an update installs beside the old one instead of over it. `packs.py` therefore
derives **version-5** UUIDs, deterministically:

```python
_UUID_NAMESPACE = uuid.uuid5(uuid.NAMESPACE_DNS, "yt2disc.minecraft.addon")

def _stable_uuid(role, pack_name=DEFAULT_PACK_NAME):
    seed = f"yt2disc:{core.slugify(pack_name)}:{role}"
    return str(uuid.uuid5(_UUID_NAMESPACE, seed))
```

The five roles, and the seed each hashes:

| Field | `role` | Seed |
| --- | --- | --- |
| RP `header.uuid` | `rp/header` | `yt2disc:yt2disc_music_player:rp/header` |
| RP `modules[0].uuid` | `rp/module` | `yt2disc:yt2disc_music_player:rp/module` |
| BP `header.uuid` | `bp/header` | `yt2disc:yt2disc_music_player:bp/header` |
| BP `modules[0].uuid` (data) | `bp/data` | `yt2disc:yt2disc_music_player:bp/data` |
| BP `modules[1].uuid` (script) | `bp/script` | `yt2disc:yt2disc_music_player:bp/script` |

So the same addon built twice — on any machine, in the browser or on a server —
carries the same five UUIDs, and `_selftest.py` asserts exactly that.


### 4.4 `sound_definitions.json` — the sound ids

This is where a **sound id** is bound to an **Ogg file inside the pack**.
`sound_definitions()` (lines 254–265) writes one entry per track, keyed by the
track's `id` (`f"{NAMESPACE}.{slug}"`):

`yt2disc_music_player_RP/sounds/sound_definitions.json` (real, from the example):

```json
{
  "format_version": "1.14.0",
  "sound_definitions": {
    "yt2disc.yt2disc_example": {
      "category": "music",
      "sounds": [
        { "name": "sounds/yt2disc_example", "stream": true }
      ]
    }
  }
}
```

The three-part chain, spelled out:

```text
sound id  yt2disc.<slug>            e.g.  yt2disc.yt2disc_example
   │  (key in sound_definitions.json)
   ▼
sounds[0].name  "sounds/<slug>"     e.g.  sounds/yt2disc_example
   │  (a path inside the RP, no extension; Bedrock appends .ogg)
   ▼
file  <RP>/sounds/<slug>.ogg        e.g.  …_RP/sounds/yt2disc_example.ogg
```

`"stream": true` keeps a long track on disk instead of loading all of it into
memory at once. `category` is always `"music"` (`SOUND_CATEGORY`).

### 4.5 How a track's `slug` is chosen

`build_addon` validates its rows through `_clean_tracks`, which is where the
slug — and therefore the sound id and the file name — is finalised:

1. `slug = core.slugify(entry["slug"] or entry["name"] or audio.stem)`, then
   `strip("_") or "track"`.
2. **Duplicate slugs are nudged apart**, not lost: a second `tone` becomes
   `tone_2`, a third `tone_3`, and so on.
3. `id = f"{NAMESPACE}.{slug}"` (e.g. `yt2disc.tone_2`).
4. `name` (the menu label) is `entry["name"] or slug.replace("_", " ")`.
5. A row with no readable `audio` file is **skipped** with a log line.

`convert()` passes exactly one row, whose `slug` is
`safe_stem(destination.stem)` and whose `name` is the uploaded title
(`title or source.name`), truncated to its stem. That is why the sound id in the
example is `yt2disc.yt2disc_example` (from the file name `yt2disc-example…`),
while the menu shows `Example Tone` (the title).

### 4.6 `scripts/main.js` and `scripts/tracks.js`

`tracks.js` is the data the player imports, so the list stays data
(`tracks_script()`, real output):

```js
// Generated by yt2disc - the songs this pack can play.
export const TRACKS = [
  { id: "yt2disc.yt2disc_example", name: "Example Tone" },
];
```

`main.js` is the `MAIN_JS` template with four placeholders filled in by
`main_script()` — `{{COMMAND}}`, `{{COMMAND_DESCRIPTION}}`, `{{MENU_EVENT}}`,
`{{MENU_EVENT_RAW}}` (the raw one is the unquoted `yt2disc:menu` for the comment).
Its behaviour:

* **Play:** `player.playSound(track.id, { volume: 1.0, pitch: 1.0 })` — no
  location, so the sound is not pinned to a point in the world and does not fade
  as the player moves. `track.id` is precisely the `sound_definitions` key.
* **Menu:** a `@minecraft/server-ui` `ActionFormData` listing every
  `TRACKS[].name`, with a trailing **"Stop the music"** button
  (`player.stopAllSounds()`).
* **Way in:** `system.beforeEvents.startup` →
  `event.customCommandRegistry.registerCommand({ name: "yt2disc:music",
  permissionLevel: CommandPermissionLevel.Any, cheatsRequired: false }, …)`.
  A custom command is a stable API, so no experiment is needed, it works
  offline, and a plain (non-operator) player can use it. The form is opened on
  the next tick via `system.run`, because a form cannot open inside the command
  itself.
* **Fallback:** `system.afterEvents.scriptEventReceive` matches
  `event.id === "yt2disc:menu"`, so `/scriptevent yt2disc:menu` opens the same
  form on a game too old for custom commands.
* **Privacy:** the play call carries no coordinates, so a player wandering away
  never has the music fade out.

The icon PNG (`pack_icon()`) is a dark disc with a green ring and a white
eighth-note, rendered pixel-by-pixel and encoded with a single `zlib.compress`
`IDAT` chunk — no image library.


---

## 5. Reproducing it externally

You need **ffmpeg** and **zip** (or any zipper). Nothing else — no ffprobe, no
Python, no image library. These are the pipeline's own steps, done by hand.

### 5.1 The bare Ogg (a plain audio conversion)

```bash
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats \
  -i "My Song.mp4" -vn -c:a libvorbis -b:a 192k my_song.ogg
```

Add a trim / resample / loudnorm exactly as in §2.2:

```bash
ffmpeg -y -hide_banner -nostdin -loglevel warning -progress pipe:1 -nostats \
  -ss 30.000 -i "My Song.mp4" -t 60.000 \
  -vn -ar 44100 -ac 2 -af loudnorm=I=-16:TP=-1.5:LRA=11 \
  -c:a libvorbis -b:a 128k my_song-from30s-to90s.ogg
```

### 5.2 The `.mcaddon`

Encode the Ogg (only `libvorbis` matters for a pack), then lay out the tree and
zip it. Using the default pack name and a track named `Example Tone`:

```bash
stem="yt2disc_music_player"          # slugify(pack_name)
slug="my_song"; label="My Song"      # the track's slug and menu label

mkdir -p "${stem}_RP/sounds" "${stem}_BP/scripts"
ffmpeg -y -hide_banner -nostdin -loglevel warning \
  -i "My Song.mp4" -vn -c:a libvorbis -b:a 192k "${stem}_RP/sounds/${slug}.ogg"

# manifest.json (RP), sounds/sound_definitions.json, and BP manifest.json +
# scripts/ exactly as printed in §4.2–4.4.
# pack_icon.png (128x128) goes in both packs — any PNG works for a hand build;
# the app draws its own with packs.pack_icon().

zip -r -X "my_song.mcaddon" "${stem}_RP" "${stem}_BP"
```

The zip must contain the two pack folders at its **root** (as above); that is
what the game expects from a `.mcaddon`.

### 5.3 The one step that is app-shaped, not tool-shaped

Everything above is standard; these are the parts that are *yt2disc's own choice*
and are only "unreproducible" in the sense that you would otherwise type them by
hand:

| Part | Where it is defined | Reproduce it with |
| --- | --- | --- |
| the icon | `packs.pack_icon()` — a deterministic 128×128 PNG drawn with `zlib`/`struct` | run it once to dump the bytes, or drop in any 128×128 PNG |
| the UUIDs (§4.3) | `packs._stable_uuid()` — `uuid5` over `yt2disc.minecraft.addon` | the exact formula in §4.3 |
| the JS (§4.6) | `packs.MAIN_JS` / `tracks_script()` | copy the templates |
| the naming (§2.8) | `core.slugify()` / `converter.output_name()` | the rules in §2.8 |
| the manifests (§4.2) | `packs.resource_manifest()` / `behavior_manifest()` | the JSON in §4.2 |

There is no hidden tool: `packs.py` is pure standard library precisely so the
same bytes are produced on a server, in a browser, and by hand.

---

## 6. The committed example

[`examples/yt2disc-example.mcaddon`](../examples/yt2disc-example.mcaddon) is the
output of the real pipeline — `converter.convert()` with `format=pack`,
`bitrate=128k`, `title="Example Tone"`, over a 3-second 440 Hz sine generated by
ffmpeg. It is 10,998 bytes and its 8 zip entries are:

```text
     size   name
      567   yt2disc_music_player_RP/manifest.json
      243   yt2disc_music_player_RP/sounds/sound_definitions.json
   12,335   yt2disc_music_player_RP/sounds/yt2disc_example.ogg
      683   yt2disc_music_player_RP/pack_icon.png
    1,035   yt2disc_music_player_BP/manifest.json
    3,213   yt2disc_music_player_BP/scripts/main.js
      142   yt2disc_music_player_BP/scripts/tracks.js
      683   yt2disc_music_player_BP/pack_icon.png
```

Drop it into Minecraft (1.21+), turn both packs on for a world, and run
`/yt2disc:music` — it lists `Example Tone` and plays the tone.

