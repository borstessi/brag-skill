# Step 4: Validate, render, and deliver

## Validate

```bash
cd <output-dir>/composition
npx hyperframes check   # brag's single pre-render gate — fix every error it reports
```

Fix all errors. `check` is brag's single pre-render gate — run it and fix everything it reports, including WCAG contrast failures (they gate as errors, not warnings). Each contrast finding carries a suggested compliant color, so apply it or adjust within the palette family and re-run `check` — most fixes need no screenshot. There is no per-element contrast escape hatch for real text; the only bypass is `check --no-contrast`, which skips the entire WCAG pass (all-or-nothing), not a way to accept one borderline element. For exact contrast thresholds, layout escape hatches, and reporting details, follow the current hyperframes-cli `check` guidance. `check`'s layout pass backstops the "keep all text readable" creative law — fix any reported overflow.

## Layout sweep (before render)

`check` and sampled stills both miss text that overflows only in a later UI state (a button whose label grows from "Interest" to "Interest sent", a counter that gains a digit, typed text). Sweep the whole timeline programmatically. Run the sweep after `check` passes and render only once its report is empty.

**1. Load every font face first.** `document.fonts.ready` resolves without loading faces nothing has rendered yet, so fit-to-width code and this sweep would measure the fallback font. In the composition, before any code measures text, and again in the sweep page before the first measurement:

```js
// Every family and weight the composition uses (brag-plan.md → Visual identity).
const FACES = [["Sora", 800], ["Inter", 400], ["Inter", 600]];
await Promise.all(FACES.map(async ([family, weight]) => {
  const spec = `${weight} 16px "${family}"`;
  const faces = await document.fonts.load(spec);
  if (faces.length === 0 || !document.fonts.check(spec)) {
    throw new Error(`Font face not loaded: ${family} ${weight}`);
  }
}));
```

`faces.length === 0` catches a family name with no matching `@font-face`, which `check()` alone reports as fine.

**2. Step through the timeline.** Open the preview in a headless browser. Seek the composition to each time `t` with the mechanism the Hyperframes renderer uses (see `hyperframes-core`), wait two animation frames, and collect findings:

```js
function layoutFindings(t) {
  const out = [];
  for (const el of document.querySelectorAll("body *")) {
    const hasText = [...el.childNodes].some((n) => n.nodeType === Node.TEXT_NODE && n.textContent.trim());
    if (!hasText || !el.checkVisibility({ opacityProperty: true, visibilityProperty: true })) continue;
    const label = `${t.toFixed(2)}s <${el.tagName.toLowerCase()}> "${el.textContent.trim().slice(0, 40)}"`;
    if (el.scrollWidth > el.clientWidth + 1) {
      out.push(`${label}: scrollWidth ${el.scrollWidth} > clientWidth ${el.clientWidth}`);
    }
    if (el.matches("[data-single-line]")) {
      const cs = getComputedStyle(el);
      const line = parseFloat(cs.lineHeight) || parseFloat(cs.fontSize) * 1.2;
      const height = el.clientHeight - parseFloat(cs.paddingTop) - parseFloat(cs.paddingBottom);
      if (height > line * 1.5) out.push(`${label}: ${height}px tall, one line is ${line}px`);
    }
  }
  return out;
}
```

Sample every 0.25s across the full duration. Where an element is on screen for less than 0.5s (a dropdown, a toast, a flash label), sample at the render frame rate through that window, or the sweep steps over it.

**3. Fix and repeat.** Size stateful UI for its longest state with `white-space: nowrap`, shorten copy, or enlarge the container. Re-run the sweep until it reports nothing.

## Cross-scene variance check

Per-frame polish does not show repetition across scenes. Capture one settled still per scene:

```bash
npx hyperframes snapshot   # PNG key frames
```

Look at the stills in order and ask for each pair: does scene N look like scene N-1 with new content? If two consecutive scenes share layout, camera behavior and transition type, change the shot grammar of one of them (see `step-2-plan.md` → "Variance across scenes") before rendering. Use the same stills as the visual gut-check for collisions and contrast.

## Preview

```bash
npx hyperframes preview
```

Tell the user the preview is running and give them the localhost URL. Invite them to check it before rendering.

If the user approves or asks to render:

## Render

```bash
npx hyperframes render --output ../brag.mp4
```

This outputs to `<output-dir>/brag.mp4` (one level up from the composition directory).

For a faster iteration render:
```bash
npx hyperframes render --quality draft --output ../brag.mp4
```

For final delivery:
```bash
npx hyperframes render --quality high --output ../brag.mp4
```

## Pick the poster frame

The poster is the still shown before the video plays — the first thing anyone sees when it's idle or unplayed. Don't leave it to the raw first frame or an arbitrary timestamp; those land on fades, mid-transitions, blank intro backgrounds, or half-rendered text.

You built this composition, so you already know its strongest moment and exactly when it lands — the hook line, the hero reveal, or the final logo. Pick that beat at a **settled** point: text fully animated in, before it exits (the storyboard timings tell you the safe window). Then extract that one frame full-res with ffmpeg. From `<output-dir>/composition`:

```bash
# use the timestamp of your strongest settled beat, e.g. 3.2s
ffmpeg -ss 3.2 -i ../brag.mp4 -frames:v 1 -q:v 2 ../brag.jpg
```

Aim for a frame that's postable on its own (the "show the thing" law — any frozen frame should be shareable). If the pulled frame lands on a transition or mid-animation, nudge the timestamp a few tenths of a second and re-extract.

### Bake the poster as frame 0

A bare `.mp4` has no `poster` attribute — every player and platform picks its own idle thumbnail, and almost all of them grab **frame 0**. Slack, Twitter/X, and Discord regenerate thumbnails server-side and ignore embedded cover-art metadata, so the *only* reliable way to control the idle image everywhere is to make frame 0 *be* the poster.

Replace **only** the first frame's pixels with `brag.jpg`, leaving every other frame and all timing untouched — same duration, same frame count, audio copied through. At 30fps the poster shows for 1/30s before the intro rolls, so it's imperceptible on playback but it's what every thumbnail grabber sees. From `<output-dir>`:

```bash
ffmpeg -y -i brag.mp4 -i brag.jpg \
  -filter_complex "[0:v][1:v]overlay=0:0:enable='eq(n,0)'[v]" \
  -map "[v]" -map 0:a? -c:v libx264 -crf 18 -preset slow -pix_fmt yuv420p \
  -c:a copy -movflags +faststart brag.poster.mp4 \
  && mv brag.poster.mp4 brag.mp4
```

The poster (`brag.jpg`) matches the video's dimensions because it was pulled from the same render, so the overlay lines up exactly. Keep `brag.jpg` alongside — it's the custom-thumbnail asset for platforms that accept an upload (Instagram, TikTok, YouTube, Facebook, and the LinkedIn post editor) and the `poster="brag.jpg"` image for any `<video>` that embeds the brag (a gallery card, the user's site).

## Write share copy

Write `<output-dir>/share-copy.txt`.

The share copy should be:
- One to three sentences max
- Postable as-is to Twitter/X, LinkedIn, or Discord
- Specific to the project — no generic "excited to share" language
- Tone-matched to the brag video

`share-copy.txt` is the canonical single caption. Do not put multi-platform variants, long launch notes, or Product Hunt copy in this file.

If variants are useful, write them to a separate optional file:

```text
<output-dir>/share-copy-variants.md
```

### Share copy by tone

**`default`:**
```
Made [App Name]. It's [what it does, in the project's own absurd terms].
[The best line from the product.]
```

**`polished`:**
```
Introducing [App Name]: [clean one-liner from the site].
Built with [stack if notable].
```

**`yc-parody`:**
```
We built [App Name] to solve [problem stated completely seriously].
[Deadpan feature or stat.]
```

**`chaotic`:**
```
[ALL CAPS CLAIM].
[App Name] is [wildly overstated description].
Link below.
```

**`deadpan`:**
```
I made [App Name].
It [what it does].
```

**`cinematic`:**
```
[App Name].
[Tagline from the site, verbatim or lightly adapted.]
```

**`app-store`:**
```
[App Name] is now live.
[Feature 1], [Feature 2], and [Feature 3] — all in one place.
```

### Example: Taxi for Taxis

```
Every day, taxis carry us. But who carries the taxis?
Taxi for Taxis: the ride-hailing app for ride-hailing assets.
Available in 12 metros.
```

## Final output structure

After this step, `<output-dir>/` should contain:

```
<output-dir>/
  brag.mp4                — the rendered video
  brag.jpg                — the poster (best frame, for <video poster>)
  brag-plan.md            — the plan and storyboard
  composition-brief.md    — the Hyperframes handoff brief
  share-copy.txt          — the share caption
  composition/            — the Hyperframes project
    index.html
    ...
```

## Telling the user

After everything is done, tell the user:
- Where the video is (`<output-dir>/brag.mp4`)
- Where the share copy is
- One sentence on what the video does creatively
- Optionally: offer to re-roll a scene, change tone, or try a different angle
