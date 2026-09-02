---
name: playstore-listing
description: "Produce a Google Play store listing — short/full description, release notes, release name, and the graphic assets (512px icon, 1024x500 feature graphic, 9:16 phone screenshots) built from a project's existing screenshots and logo. Use when the user is preparing a Play Console listing, asks for store descriptions or screenshots, mentions 'playstore assets', or has raw device captures that need to become uploadable screenshots. For the production-access questionnaire, use play-console-answers instead."
---

# /playstore-listing

Turn what a repo already has — device captures, a launcher icon, the product
docs — into a Play Console listing. One markdown file with every text field, one
`out/` directory with every graphic, one script that rebuilds both.

Do not generate new artwork. The launcher icon and the app's own palette are the
brand; a store listing that looks like a different app is a worse listing.

---

## What Play wants

| Field | Limit | Notes |
|---|---|---|
| App name | 30 | |
| Short description | 80 | Leads search results. Put the differentiator in it. |
| Full description | 4000 | |
| Release notes | 500 | Per language, per release |
| Release name | 50 | Console-internal, never shown to a user |
| App icon | 512×512, ≤1 MB, PNG/JPEG | **Must be opaque.** Transparency is rejected. |
| Feature graphic | 1024×500, ≤15 MB, PNG/JPEG | |
| Phone screenshots | 2–8, 9:16 or 16:9, sides 320–3840 px, ≤8 MB each | |

---

## 1 · Find the sources first

```sh
ls **/playstore/ **/*screenshot* 2>/dev/null       # raw captures
find . -path ./build -prune -o -iname "*logo*" -print -o -iname "ic_launcher*" -print
ls android/app/src/main/res/mipmap-xxxhdpi/
```

Look for a larger copy of the launcher artwork before upscaling a small one — a
sibling web project or a `public/`/`dist/` directory often has a 500px or 1024px
version of the same file.

Read the product docs for the app's voice before writing a word of copy: the
PRD, a design handoff, the README. If the project has copy rules (tone, banned
words, no exclamation marks), the listing follows them — the store page is the
first screen of the product.

---

## 2 · Write the copy

Rules that hold regardless of project:

- **Short description is a search field.** Lead with the concrete nouns someone
  would type. "Savings tracker for cash, gold by the gram, property, and
  vehicles" beats "Track your wealth beautifully".
- **Open the full description with the premise, not a feature list.** Why this
  app instead of the twenty free ones. Features go under short ALL-CAPS
  headings after that.
- **State what it does not do.** It filters out the wrong installs, which is the
  cheapest way to protect a rating.
- **Free vs paid goes in the description**, plainly, including what is never
  gated. Surprises there are what one-star reviews are made of.
- No "#1", no "best", no fake urgency, no emoji spam. Play rejects
  superlatives it can't verify, and they read as noise anyway.
- Release notes: what changed, in the app's own words. Never "bug fixes and
  improvements". No version number in the body — the Console shows it already.

**Release name = the version string from the manifest, verbatim** (`0.1.0+2`
from `pubspec.yaml`, `versionName+versionCode` from Gradle). Play defaults it to
`2 (0.1.0)`; using the repo's own string means a Console release and a commit
name the same thing. Re-check it after any version bump — it silently goes stale.

**Count characters with a script, never by eye.** Over-limit text is silently
truncated at the worst possible place.

```sh
python3 - <<'EOF'
import re, pathlib
t = pathlib.Path('playstore/listing.md').read_text()
for label, pat in [('full', r'```\n(Most .*?)\n```'), ('notes', r'```\n(The first release.*?)\n```')]:
    m = re.findall(pat, t, re.S)
    if m: print(label, len(m[0]), 'chars,', m[0].count('!'), 'exclamation marks')
EOF
```

---

## 3 · Build the graphics

macOS has `sips` and usually `ffmpeg`. It does **not** have ImageMagick, and a
homebrew `ffmpeg` is often built without `drawtext` — check before relying on
it:

```sh
ffmpeg -hide_banner -filters | grep drawtext
```

No `drawtext` means ffmpeg cannot render text. Use headless Chrome instead: an
HTML file gives real fonts, gradients and layout, and every mac has Chrome or
can screenshot with any Chromium.

Write `playstore/build.sh` so the assets are reproducible rather than a one-off:

```sh
#!/bin/sh
set -eu
cd "$(dirname "$0")"

LOGO=path/to/logo-512.png          # the launcher artwork, largest copy available
OUT=out
rm -rf "$OUT"; mkdir -p "$OUT/screenshots"

# --- icon: 512x512, alpha flattened (Play rejects transparency) ---
sips -z 512 512 "$LOGO" --out "$OUT/.icon.png" >/dev/null
ffmpeg -v error -y -i "$OUT/.icon.png" -vf "color=c=0xRRGGBB:s=512x512[bg];[bg][0]overlay" \
  -frames:v 1 -pix_fmt rgb24 "$OUT/icon-512.png"
rm "$OUT/.icon.png"

# --- feature graphic: 1024x500 from feature.html ---
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless --disable-gpu --hide-scrollbars --force-device-scale-factor=2 \
  --window-size=1024,500 --screenshot="$OUT/feature-graphic.png" \
  "file://$PWD/feature.html" 2>/dev/null
sips -z 500 1024 "$OUT/feature-graphic.png" >/dev/null

# --- screenshots: crop system chrome off, float on a brand gradient at 9:16 ---
i=1
for f in <captures in the order they should appear>; do
  ffmpeg -v error -y -f lavfi -i "gradients=s=${CW}x${CH}:c0=0xTOP:c1=0xBOTTOM:x0=0:y0=0:x1=0:y1=${CH}:d=1" \
    -i "$f" -filter_complex "[1]crop=w=${SW}:h=${SH}:x=0:y=${TOPCROP}[s];[0][s]overlay=${OX}:${OY}" \
    -frames:v 1 -q:v 3 "$OUT/screenshots/$(printf '%02d' $i).jpg"
  i=$((i+1))
done
```

`feature.html` — the app's own gradient, type and accent, the launcher mark, a
one-line value statement. No stock imagery, no invented tagline. Keep it to
`<style>` plus a flex row; it is rendered once, at one size.

```html
<!doctype html>
<meta charset="utf-8">
<style>
  * { margin:0; padding:0; box-sizing:border-box; }
  body { width:1024px; height:500px; overflow:hidden;
         background:linear-gradient(155deg, #TOP 0%, #BOTTOM 100%);
         font-family:"Helvetica Neue", Helvetica, Arial, sans-serif;
         display:flex; align-items:center; gap:56px; padding:0 72px; }
  img { width:252px; height:252px; border-radius:24%;
        box-shadow:0 18px 48px rgba(0,0,0,.55); }
  h1 { font-size:64px; font-weight:500; letter-spacing:-1.5px; color:#TEXT; }
  p  { font-size:21px; line-height:1.5; color:#MUTED; margin-top:14px; max-width:520px; }
</style>
<img src="path/to/logo.png">
<div><h1>App Name</h1><p>One line of what it is.</p></div>
```

### The screenshot geometry

Modern phone captures are ~9:20, outside Play's stated ratio. Do not scale to
fit — that shrinks the UI. Crop the system chrome, then centre the result on an
exactly-9:16 canvas painted with the app's own background gradient, so the added
margin is invisible and no content is lost.

Worked example, source 1260×2800:

```
crop  1260×2628  at y=112       112 px status bar, 60 px gesture pill
canvas 1575×2800                 1575 = 2800 × 9/16, exact
overlay at 157,86                (1575-1260)/2, (2800-2628)/2
```

Sample the real background colour rather than guessing — the app's surface is
usually a gradient, and a flat pad shows a seam:

```sh
for y in 300 1400 2700; do
  ffmpeg -v error -i shot.jpg -vf "crop=w=2:h=2:x=2:y=$y" -f rawvideo -pix_fmt rgb24 - | xxd -p
done
```

Cropping the status bar is not only about ratio — raw captures leak the
developer's notification icons, battery percentage and clock into the store
listing.

**Order the screenshots as an argument**, not as a file listing: the hero screen
first, then what it holds, then the primary action, then depth, then trust
(history, backup, settings). Play shows the first two or three in search.

---

## 4 · Deliver

```
playstore/
  listing.md        every text field, with its character count stated
  build.sh          rebuilds out/ from source
  feature.html      the feature graphic's source
  out/
    icon-512.png
    feature-graphic.png
    screenshots/01..NN.jpg
```

`listing.md` states the actual measured character count next to each field, the
screenshot order with a one-line caption each (for whoever uploads them — Play
shows no captions), and the rebuild command. Verify every output's real
dimensions before reporting done:

```sh
for f in out/*.png out/screenshots/*.jpg; do
  printf '%-34s %s %s\n' "$f" \
    "$(sips -g pixelWidth -g pixelHeight "$f" | awk '/pixel/{printf "%s ", $2}')" \
    "$(du -h "$f" | cut -f1)"
done
```

---

## Traps

- **`sips` cannot write to its own input file** — it exits with "same as Input".
  Stage through a temp name.
- **A 500×500 logo upscaled to 512 is fine**; a 192×192 launcher icon upscaled
  to 512 is not. Go find the bigger copy.
- **Icon transparency is a rejection, not a warning.** Flatten onto the
  artwork's own background colour, never white.
- **Play's "undeclared permission" and "app optimization" errors are about the
  bundle, not the listing.** They belong to the release, not this skill — but if
  one appears, check where the permission actually comes from
  (`build/app/outputs/logs/manifest-merger-*-report.txt`) before writing a
  justification for it. A permission a plugin injected and the app never uses
  should be stripped with `tools:node="remove"`, not explained.
- **An app-level declaration persists while any active release on any track
  still requests it.** Re-uploading to one track does not clear it.
