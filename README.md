# Design Playbook

Working reference for design projects across media: print PDFs, signage, social and digital graphics, branded collateral. Covers the HTML→PDF pipeline, typography, photography, layout craft, and repeatable design signature. Read this before starting any design project. Add only what changes future behavior or capability; cut anything that's just narrative.

---

> **For AI agents and future sessions — read before adding anything:**
>
> This file is publicly shared so any designer working with an AI agent can benefit from the lessons documented here. Follow these rules strictly when updating it:
>
> 1. **Design only.** Every entry must be about design craft, process, or tooling. No web dev, no DevOps, no app architecture — those belong in project-specific files.
> 2. **No names.** Never write a client name, project name, person's name, or company name anywhere in this file. Use generic descriptions instead ("a multi-page deck", "a print client", "confirmed in practice").
> 3. **No personal references.** No tool names specific to one workflow (e.g. a specific CLI, a specific AI platform), no account names, no file paths tied to a specific machine. Write as if any designer using any AI agent will read this.
> 4. **Lessons, not logs.** Document what changes future behavior — a discovered rule, a recurring trap, a technique that works. Don't log what happened on a specific project.
> 5. **Keep both copies in sync.** The local copy (`DESIGN_PLAYBOOK.md`) and the GitHub copy (`design-playbook/README.md`) must always match. After editing locally, copy changes to the GitHub version and push.

---

## 1. The Formula — steps for any new design request

1. **Find the client folder.** Check `specs.md`, `Logos/`, `Design Examples/` first. If `specs.md` is missing or thin, build/update it from actual logo files, design examples, and brand PDFs before designing anything. Never guess brand colors/fonts when source files are available.
2. **Find the project brief.** Reference files, sometimes a `Photos/` subfolder — confirm what's available before starting.
3. **Decide the destination before building:**
   - Standalone print PDF → **HTML → headless Chrome print-to-pdf** (§3). Default for print-bound/custom-branded work.
   - Client-editable Adobe Express doc → build **natively in Express** (`search_design` + `fill_text` + `change_background_color`), not HTML-import (§4).
   - If unsure, ask — don't assume Express just because the tools exist.
4. **Gather assets before writing HTML**: logo (SVG, both color + white/reversed), fonts (§5), photos/illustrations (§6 — the biggest time sink, budget for it).
5. **Build the HTML**, one fixed-size `.page` div per page at exact target dimensions in inches, `page-break-after: always` between pages.
6. **Verify visually before calling it done** — render to PNG and look, checking for: overflow/clipping at page boundaries, orphans/widows (§7), margins actually respected (measure, §8), webfonts actually rendered (§3).
7. **Generate the final PDF**, verify page count/dimensions with `pypdf`, save to the project folder.
8. **For any visual fix request** ("balance this", "move that") — measure pixels first (§8), calculate the change, re-render, re-measure to confirm, then ship. Don't guess-and-reguess.
9. **Save the working HTML source to the project folder as you go.** A raster-only survivor with no source file means every further edit is a slow rebuild from scratch. Save on first good state, re-save after each accepted edit.

**On feedback:** design notes usually arrive as a feeling ("boxed in," "not enough breathing room"), not a spec. Translating that into a concrete change is the job — see §11 for confirmed translations.

**On modeling a new piece after an existing reference document:** extract the reference's page *arc* (what each page does, not its literal content) but don't port a section that doesn't apply to the new subject. Confirmed in practice: a reference brand guide's page built entirely around a tagline the new brand doesn't have should be replaced with a section the new brand actually needs (e.g. a color-and-typography spec) rather than forced into shape. Preserve the reference's *shape*, retarget each section's *content* to what the new subject actually needs.

---

## 2. Client asset & delivery conventions

- A `specs.md` per client is the source of truth for brand facts. Update it whenever a project reveals a new brand fact *or* a brand-specific fix/gotcha — it's "what we learned building for this client," not just static brand data.
- Deliverables save to the project folder. The editable HTML source follows the same rule — always save the source, not just the final PDF/PNG.
- In-store inkjet print: **0.5in white margin on all four edges**, nothing may cross it (different from vendor bleed).
- Vendor bleed printing: follow the vendor's own bleed/trim/safe-zone spec exactly — don't assume a standard. GotPrint default seen so far: 0.1in bleed/trim, ~0.2–0.25in safe zone if unspecified.
- **Client/office desktop inkjet** (the client prints it themselves — brand guides, one-off certificates, internal reference docs — not a vendor run or in-store signage): **0.4–0.5in uniform white margin, no bleed.** Most office/home inkjets can't print true edge-to-edge and will crop or leave a ragged unprinted strip if a background color runs to the paper edge. Design any full-bleed-looking element (a solid-color cover panel, say) to stop within that margin rather than running to the page edge — see §3 for how to declare it reliably. Confirmed well-received: clients consistently respond well to the clean, printer-friendly white frame this produces.

---

## 3. HTML → PDF pipeline (default workhorse)

Gives full control over typography/spacing/layout and has been the only path that reliably survives complex logos and gradients — Express-import repeatedly mangles both (§4).

```bash
# Render to PNG for visual verification
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --hide-scrollbars \
  --window-size=<px_width>,<px_height> \
  --screenshot="<out>.png" \
  --run-all-compositor-stages-before-draw \
  --default-background-color=FFFFFFFF \
  "file://<path>.html"

# Final PDF
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu \
  --print-to-pdf="<out>.pdf" \
  --no-pdf-header-footer \
  --run-all-compositor-stages-before-draw \
  --virtual-time-budget=9000 \
  "file://<path>.html"
```
- **`--virtual-time-budget`**: with Adobe Fonts (Typekit) via `<link>`, a short budget (3000–6000ms) can render invisible text in a screenshot — the font hasn't loaded yet. False alarm, not a bug, but **always verify against the actual PDF** (`pdftoppm -png`), never trust a quick screenshot when text seems missing. Use 8000–9000ms for webfont-heavy pages.
- Verify page count and physical size after generating (`pypdf`: `mediabox.width/72`, `height/72`).
- `page-break-after: always` on `.page`, omit on the last (or `.page:last-child { page-break-after: auto; }`).
- **Always declare `@page` explicitly — don't rely on Chrome's default margin.** With no `@page` rule in the CSS, headless Chrome's print-to-pdf applies its own undocumented margin, and it's **uneven**: confirmed by isolated single-color test page at ~0.39in on the top/left/right and ~0.61in on the bottom (not a fluke of one document — reproduced on a blank page with nothing else on it). That default happened to look good on one cover (§2, client/office inkjet), but treat it as something to declare on purpose, not inherit by accident:
  - True vendor-bleed job → `@page { size: <W>in <H>in; margin: 0; }`, with bleed art actually extending to the CSS page edge.
  - Client/office inkjet job (§2) → `@page { size: <W>in <H>in; margin: 0.4–0.5in; }` for a clean, symmetric, printer-safe margin — and keep any background color/panel inset within it.
  - Check any *existing* full-bleed deliverable that didn't set `@page` explicitly — it may be carrying this same unintended uneven margin without anyone having noticed.

### Professional stationery bleed (8.5×11 letter, 0.125in bleed all sides)
Complete recipe for a print-ready business letter with a full-bleed colored footer. Canvas = 8.75×11.25in; trim = 8.5×11in; bleed = 0.125in on every edge.
```css
@page { size: 8.75in 11.25in; margin: 0; }

.page {
  width: 8.75in; min-height: 11.25in;
  padding: 0.125in 0.125in 0;  /* shifts body content inward to the trim zone */
  display: flex; flex-direction: column;
}
.letter-body { flex: 1; }       /* fills remaining space; footer stays pinned to bottom */

.footer {
  flex-shrink: 0; height: 2.0in;
  margin: 0 -0.125in;           /* escapes page padding → extends bg color to canvas edge */
  padding: 0 0.125in;           /* restores interior alignment to the trim zone */
  -webkit-print-color-adjust: exact; print-color-adjust: exact;
}
.footer-inner {
  padding: 0 0.52in 0.125in;   /* side padding matches letter body margins */
  height: 100%;
  display: flex; align-items: center;
}
```
Three non-obvious mechanics:
- **`.page { padding: 0.125in ... }` is the key move.** It shifts all letter content 0.125in inward so the body's stated margins are measured from the trim, not the bleed edge — without this, the body's side padding would be 0.125in wider than intended.
- **Footer negative margins + restore padding.** `margin: 0 -0.125in` lets the footer background escape the page's padding and reach the canvas bleed edge; `padding: 0 0.125in` then restores alignment of the footer's *content* back to the trim zone — one line undoes the other's effect on the interior, leaving only the background color change at the bleed.
- **`padding-bottom: 0.125in` on `.footer-inner` centers content to the trim zone, not the full bleed height.** Without it, `align-items: center` centers to the full 2.0in (including 0.125in bleed below the trim), pushing the content visually low. Adding the bottom padding shifts the flex-center calculation upward by 0.125in — content reads as centered within the printed region.

### Clickable links survive headless Chrome PDF rendering
`<a href="...">` tags in the HTML become real clickable hyperlinks in the Chrome-rendered PDF — confirmed on a multi-page linked PDF. This means video thumbnails, "Watch →" text links, and any other anchor are clickable in the final PDF without any Acrobat post-processing. If the user wants richer annotation control (named destinations, link rectangles over images with no visible anchor text) they can still finish in Acrobat — but for standard link behavior, the HTML alone is sufficient.

### Don't trust live CSS effects to survive into every PDF viewer — bake them into pixels instead
Headless Chrome renders the HTML faithfully and `pdftoppm`/your own screenshot will confirm it looks right — but that only proves *Chrome's own* PDF renderer handled it. Confirmed in practice: two separate CSS features looked perfect in every screenshot taken during the build, then failed for the client viewing the same file — a `background-image` on a `<div>` didn't render at all, and `backdrop-filter: blur()` "frosted glass" cards showed no blur. Both are cases where Chrome's print-to-pdf can encode the effect as a live/advanced PDF feature (a soft mask, a transparency group) rather than flattening it to plain pixels, and not every PDF viewer implements that feature — poppler (`pdftoppm`) does, so your own verification pass won't catch it.
- **Photos:** use a real `<img>` tag, never a `background-image` on a div, for anything that must be visible in an arbitrary viewer.
- **Blur/frosted-glass effects:** don't rely on `backdrop-filter`. Bake the blur into the image itself before embedding — e.g. `PIL.ImageFilter.GaussianBlur` on the region behind the card, composited with a feathered alpha mask so the transition isn't a hard edge — then give the card a plain `rgba()` background and border, no filter property at all.
- **Cross-aspect photo framing:** don't rely on the browser's `object-fit: cover` + `object-position` to crop a source image down to a very different final aspect ratio at render time — confirmed subject framing can come out different than what your own render showed. Do the crop yourself (PIL, using known subject coordinates) to the *exact* target pixel box, save that as the final flat image, and let the `<img>` just fill its box at 100%/100% — nothing left for the viewer to compute.
- **General rule:** for a PDF whose exact viewer is unknown, prefer "one flat raster image already showing the final result" over "CSS that computes the final result at view time," for any effect beyond plain color/border/typography (which are universally safe).

### Editing an existing HTML file with embedded base64 images
Once photos are embedded as `data:image/...;base64,...` inline, the file becomes mostly unreadable/un-greppable as text — a single `<img>` line can be megabytes, `Read` will refuse the whole file, and a plain `grep -n` dumps the same megabytes to the terminal.
- **Strip base64 for display, not for editing:** `python3` one-liner with `re.sub(r'(data:image/[a-zA-Z+]+;base64,)[A-Za-z0-9+/=]+', r'\1[BASE64_DATA]', line)` over the lines you need, printed with line numbers — gives a readable structural map (CSS rules, tag nesting, attributes) without ever loading the actual image bytes into context.
- **Use the `Edit` tool only for short, unique, non-base64 text** — CSS rules, class names, paragraph copy, small attribute values (`style="width:...;height:...;"` when it's genuinely unique). If the same short style string repeats identically across multiple `<img>` tags (e.g. three photos in a row all sized the same), `Edit` can't disambiguate — fall back to a `python3` script that edits by line index (`lines[282] = re.sub(...)`) instead of by string match.
- **Inserting a whole new block** (a new section with its own new photos): build the block in Python as a string (base64-encoding new images with `base64.b64encode`), then splice it into the file at a unique text anchor via plain string `.replace(anchor, new_block + anchor, 1)` — never hand-type or paste base64 through the `Edit` tool.
- **Swapping a single existing photo in place** ("replace that left image"): find a nearby unique text anchor (a caption, a class name specific to that photo/section), locate the *next* `<img src="data:image/...;base64,` after it, then find its closing `/>` — replace that whole substring with a freshly base64-encoded new `<img>` tag (same `style`/dimensions as the original, unless the swap calls for a different crop). Same script pattern as insertion, just a slice-replace instead of a splice.

---

## 4. Adobe Express — when it works, when it doesn't

**Works:** native template editing (`search_design`, `fill_text`, `change_background_color`, `animate_design`, `download_design`) — good for client-editable pieces or quick template social posts.

**Doesn't work — importing custom HTML with complex logos** (`export_html_to_express` / `import-claude-design-from-url`):
- Detailed multi-path inline SVG logos import as garbled shapes even after readiness-skill validation passes clean — genuine conversion limitation, not fixable by rasterizing to PNG first (gradients break too).
- Every export call creates a **brand-new** document — no in-place update, so iterating produces doc sprawl. Warn the user up front if they want to keep iterating in Express.
- `get_account_type` must return `auth` (not `guest`) before export works — check before promising an Express deliverable.

**Rule of thumb:** real logo + needs to be pixel-perfect → build in HTML, hand over PDF/PNG. If Express is a hard requirement, rebuild manually inside Express from a template rather than importing.

---

## 5. Fonts

- `find_fonts` (Adobe Fonts) first, using specs.md's documented names, requesting all needed weights in one call.
- Not everything is in Adobe Fonts (e.g. Red Hat Display — Google Fonts only). On `not_found`, don't guess a substitute — use `font_recommend`, or Google Fonts `@import` only if the brand doc explicitly names one.
- `get_fontkit_embed_url` wants PostScript names only (`MyriadPro-Bold`, never "Myriad Pro Bold").
- **A client's website fonts and print/Illustrator fonts can be deliberately different systems** — don't assume the site's font-face declarations are the brand's print typography, or vice versa. Track both separately in specs.md.
- **Check for an actual brand/logo/style guide PDF before trusting what a prior deliverable's HTML used as "the brand font."** Confirmed in practice: specs.md stated a font as "confirmed" because an earlier deliverable's HTML used it — but that HTML's own choice was never checked against a real guide, so the wrong font got laundered into specs.md as fact. A real logo guide existed in the client folder the whole time naming a completely different typeface. Search `find <ClientFolder> -iname "*guide*" -o -iname "*brand*" -o -iname "*style*"` before recording any font as confirmed, and only fall back to "what a past deliverable used" — labeled explicitly as unverified — when no such guide exists. The same applies to brand colors: a logo guide's stated hex can also drift from what the *live* logo files actually use (confirmed same day — the guide's green/navy hex values were close but not identical to the shipped logo SVGs); when they disagree, the live asset files are the more reliable source since they're what's actually in current use.

---

## 6. Stock photos & graphics (the biggest time sink)

### Locating a client-supplied photo referenced without a filename
When the user says "the new photo/image in the folder" (singular or plural) without naming it, don't ask which file — check mtimes: `find <client-or-project-folder> -maxdepth 1 -type f -newermt "<timestamp of the last file you already know about>"` on the folder(s) they'd plausibly have dropped it into (client root folder, project folder, and any `Photos/`/`Images/` subfolder). Confirmed in practice: new files were dropped as loose files directly in the client's root folder rather than into an existing `Photos/` subfolder — check the client's own `specs.md` for a documented drop location before assuming the organized subfolder is where new files land.

### Sourcing
- `asset_search` with `entityScope: "StockAsset"`; filter `contentType` (`Vector`/`Photo`) and `pricing` as needed.
- `renditionURL` is a low-res preview only — call `asset_license_and_download_stock` for the full-resolution file before final use.
- `asset_inline_preview` fails intermittently — fallback: `curl` the `renditionURL` directly and view the result.
- Licensed vector downloads are `.ai` files that are actually PostScript/EPS — not directly viewable/embeddable.

### Rendering licensed .ai/.eps to PNG
`document_render_vector` (Adobe's cloud tool) is unreliable — async jobs don't resolve, no polling mechanism exists. **Use local Ghostscript instead** (`brew install ghostscript` — Inkscape alone can't read EPS/PS without it):
```bash
gs -sDEVICE=pngalpha -dEPSCrop -r300 -dTextAlphaBits=4 -dGraphicsAlphaBits=4 \
   -o output.png input.ai
```

### Hidden opaque backgrounds in "transparent" stock templates
Stock templates frequently have opaque white elements baked in (a card behind placeholder text, an unfinished lower canvas) that are **invisible against a white page** and only surface once placed on a colored background.
- **Always test a "cleaned" transparent asset against a non-white background immediately** — a white-page check proves nothing.
- **Remove by color, not position**: scan for near-white pixels (`r,g,b > 225`, opaque) and zero their alpha, rather than blanking a rectangular region (which risks slicing real artwork).
- **Plain color-matching alone isn't safe** when a foreground detail (white socks, shoe soles) shares the exact background color. Use connected-component labeling (`scipy.ndimage.label`) and only remove components above a size threshold (~0.5–2% of image area) — the real background is one giant region; a same-colored detail is small and separate. `pip3 install scipy --break-system-packages` if needed. Apply this on **every** removal pass, including the first, not just later cleanup.
- **Check the full canvas** — don't assume an unscanned region is already transparent.
- **Zero RGB wherever alpha=0 before any LANCZOS resize** (`arr[mask]=[0,0,0,0]`) — stray non-black RGB in transparent pixels bleeds into visible halos on downscale.
- **Crop to content bounding box** (`Image.getbbox()`) after cleaning, before final resize/encode.
- Test every cleaned asset against **at least two different background colors** — one solid color isn't enough to catch a same-color-coincidence bug.
- **Multi-figure "row of people" stock sheets** often bake in *two* different opaque background elements at different tones — a white canvas fill *and* a separate light-gray floor/shadow strip (e.g. `(244,244,244)`) under every figure's feet. A single tolerance pass tuned to pure white can miss the gray strip (the color distance exceeds a tight tolerance) — run a second targeted removal pass for the floor color. Also, adjacent figures' raised limbs frequently touch or overlap in these sheets, so isolating the "largest connected component" to pull one figure out can drag in part of a neighboring figure. When that happens, don't hand-carve the boundary — pick a different figure/pose from the sheet that stands clear of its neighbors (arms-down poses isolate reliably; poses with both arms raised are the most likely to touch a neighbor). **After isolating any figure this way, re-view it at full size before compositing** — a crop or component boundary that lands mid-limb silently produces a missing hand/arm that's easy to miss until it's already sitting in the final layout.

### Generative expand to manufacture text real estate
When a photo's real subject fills the frame edge-to-edge with no natural empty area for a text panel (confirmed on an aerial team-meeting photo and a letterboxed close-up), `image_generative_expand` (Adobe) can outpaint additional canvas on one side rather than settling for a tight/cropped subject or a heavy branded-color overlay covering the photo. Practical math that trips people up:
- **The generated fraction of your final crop window is capped by how much of the window is occupied by content you must keep, not by how much you expand.** If your crop window must reach all the way to the subject's far edge (near the original photo's own edge), adding more expand pixels just shifts everything sideways by the same amount — the generated fraction stays constant unless you either expand enough to blow past that constraint, or accept trimming a bit of the far edge (usually background/empty space, not the subject) to shrink the required window.
- Work it out with real numbers before iterating blindly: expand amount E, crop window width `Wc = target_height × target_aspect`, required right edge `xEnd` (subject's far edge + margin) — generated pixels shown = `E − xEnd + Wc`, which only grows if E grows *and* xEnd doesn't track it (i.e., you're willing to crop that edge tighter) or Wc grows (needs more height, i.e. a wider aspect or a taller crop).
- Two-pass expand is fine and cheap when the first pass isn't enough — chain the second call off the first's `outputUrl` directly (no need to re-download/re-upload).
- The expand blends seamlessly on both simple textures (wood floor, pavement) and complex ones (office window/skyline) — confirmed twice, no visible seam either time.
- Generative-credit pool is separate from Stock's image-download quota on Adobe's plan — licensing a photo and outpainting it don't draw from the same allowance. Check the account dashboard (Manage Account → Generative AI) for actual remaining balance if the user asks; don't guess a number.

### Re-aspecting a landscape photo to a narrow portrait panel — try the crop before reaching for generative expand
`image_crop_and_resize` (Adobe, `fit:"reframe"`, subject/prompt-aware `focus`) will proactively flag `requires_expand_chain` and hand you pre-filled `image_generative_expand` args whenever the requested aspect would clip the detected subject — worth knowing this is a *conservative* safety flag, not a hard requirement. Confirmed in practice: requesting a narrow portrait ratio (~0.64, matching a sidebar panel) from 1.5:1 landscape sources triggered expand suggestions of **+3900px per side on a 5300px-tall image** — i.e., nearly tripling the canvas with AI-generated content, for scenes (a factory floor, a hospital corridor) where that much fabricated content is a real quality risk.
- **The tool flags clipping relative to the *entire* detected subject bounding box**, which for a "the workers operating the machine"-style prompt can span almost the full frame width — even when the box is looser than what a human would consider essential to keep.
- **Before accepting the expand chain, retry with a request for a less extreme output ratio first** (e.g. ask for `4:5` instead of the panel's actual `~0.64`) — if the source has enough native width/resolution, this alone often returns a clean crop with *no* expansion needed. Confirmed: dropping from the exact target ratio to `4:5` turned "needs ~2× canvas height" into "clean crop, zero expansion" for two of six test photos, and a mild trim for the rest.
- **When the tool's own bbox/focus data is available** (returned in the `metadata` of a call that already ran, even one that suggested clipping), you can skip further Adobe round-trips entirely: crop locally via PIL using that bbox center, full source height, and whatever width the subject can spare — no re-upload, no AI generation, and you keep full control of the framing. This is the safer default whenever the source resolution has width to spare; save the real `image_generative_expand` chain for when the source genuinely lacks the pixels (a tight crop with no slack in any direction).
- Once you have a same-ratio-or-close crop, don't lean on the browser's `object-fit: cover` to finish the job to an exact container size (see §3's CSS-reliability note) — do the final precise crop yourself too.

### Vet subject cleanliness, not just composition/lighting
A photo can have excellent light, composition, and mood and still fail on closer look — confirmed on a golden-hour truck-fleet shot rejected late because the trucks themselves were visibly dirty/grimy at full zoom, unprofessional for a B2B brand even though the overall shot looked premium at thumbnail size. Check the actual subject's physical condition (vehicle/equipment paint, cleanliness, wear) at full resolution before finalizing, not just the mood/lighting/composition.

### Removing backgrounds from real photos (headshots, product shots) — use Adobe `image_remove_background`, not the §6 color-threshold method
The color-threshold/connected-component technique above is for flat-vector stock illustrations with a solid or near-solid backdrop. For **real photos** (staff headshots against a busy background like brick or foliage), use Adobe's `image_remove_background` (Photoshop's select-subject cutout) instead — it handles hair strands and soft edges that a threshold pass can't. Confirmed clean on a batch of real staff headshot photos shot against a brick wall — no fringing, hair detail preserved, at full original resolution (no downscaling below ~1280x1600).

**Getting a local file to Adobe without the interactive file picker:** the tool docs for `asset_add_file`/`asset_initialize_file_upload` say local-machine files require the file-picker UI (`asset_add_file`), which needs a user click per file — slow for a batch. When your AI agent has direct filesystem and network access, there's a faster programmatic path:
1. `asset_initialize_file_upload({path, file_size, media_type})` → returns a block-transfer URL.
2. Upload the actual bytes yourself: `curl -X PUT "<transfer URL>" -H "Content-Type: <mime>" --data-binary @"<local file>"` — expect `200`.
3. `asset_finalize_file_upload({filename, transfer_document})` → returns `presignedAssetUrl`, feed that directly into `image_remove_background`.
4. Download the result with `curl -sL -o out.png "<outputUrl>"`.

One intermittent failure mode seen mid-batch — cleared on a single retry, no root-cause fix needed: the Adobe session token can expire mid-batch ("session token is about to expire") on `image_remove_background`. Don't loop more than once or twice before flagging to the user.

### Photo quality & DPI
Check effective DPI at actual placement size before using any photo as cover/full-bleed — floor is ~150 DPI. Prefer a higher-res original when one exists. When it doesn't, **Upscayl's CLI is installed and confirmed working** (2026-08-15) — AI upscale as a real option now, not just "find a better source":

```bash
/Applications/Upscayl.app/Contents/Resources/bin/upscayl-bin \
  -i input.jpg -o output.png \
  -m /Applications/Upscayl.app/Contents/Resources/models \
  -n upscayl-standard-4x -s 4 -v
```
- Runs fully local, free, no API key — detects and uses the discrete GPU when present (confirmed via Vulkan, not just CPU).
- `-n` model options include `upscayl-standard-4x` (general photos), `digital-art-4x`, `high-fidelity-4x`, `ultrasharp-4x`, `remacri-4x` — pick based on source content, general photos default to `upscayl-standard-4x`.
- `-s` is the final output scale (can differ from the model's native scale); `-r WxH` resizes to an exact target instead.
- The plain GitHub release binary (`xinntao/Real-ESRGAN-ncnn-vulkan`) is stale (2022) and **segfaults** — it ships with no `models/` folder. Don't reach for that path again; the Upscayl app bundle (`brew install --cask upscayl`) ships a current binary with matched models and is the confirmed-working route.
- Upscaling adds detail the source never had — still prefer a genuine higher-res original when one exists. Use this as the fallback, not the default.

### JPEG quality maximization within a file-size budget
Don't guess a "safe" quality value and stop — search from 100 downward for the highest value that still fits the budget:
```python
for q in range(100, 60, -1):
    img.save('/tmp/probe.jpg', quality=q, optimize=True)
    if os.path.getsize('/tmp/probe.jpg')/1024 <= budget_kb:
        img.save(final_path, quality=q, optimize=True)
        break
```
Run this on the **final rendered PNG**, never on a pre-guessed quality.

**Double-compression trap:** if a photo crop was embedded in the HTML as a base64 JPEG at some earlier, lower quality, running the search on a screenshot of that HTML can't recover already-discarded detail — it just optimizes the degraded pixels. **Before the final pass, regenerate the source-photo crop from the pristine original at near-lossless quality (95+) and re-embed it.**

### When stock/AI search is the wrong tool
Not every illustration need is solvable by stock search or AI generation — a bespoke/specific-enough art style (e.g. a particular hand-drawn portrait look) won't survive AI generation and stock won't have it. **Say so and suggest a human illustrator** rather than burning time on searches/prompts that won't converge.

### `asset_search` filters can zero out results that exist
On `StockAsset` searches, adding `filters: {contentType, orientation}` returned `totalHits: 0` for queries that had hundreds of hits with no filters at all. If a stock search comes back empty, retry the same query with filters stripped and check orientation/size from the returned `width`/`height` fields instead of filtering for them.

### Duotone treatment to match an established "dark navy overlay" photography style
When a client's brand calls for desaturated/duotone editorial photography (check specs.md) but the sourced stock photo is full-color, map it through a two-color gradient rather than just desaturating + tinting — a flat desaturate reads muddy, a proper duotone reads intentional:
```python
import numpy as np
gray = np.array(img.convert('L'), dtype=np.float32) / 255.0
gray = np.clip((gray - 0.5) * 1.12 + 0.5, 0, 1)      # slight contrast boost — avoids a flat/muddy result
dark  = np.array([16, 28, 38])                        # shadow color
light = np.array([196, 206, 212])                     # highlight color
duotone = dark + gray[:,:,None] * (light - dark)
```
If the photo needs to blend into a solid-color panel below it (a photo banner sitting on top of a colored section), blend the last N rows toward that panel's *exact* background color in the same pass — fading to transparent instead leaves a visible seam where the photo's own base color shows through.

---

## 7. Orphans & widows

Dedicated pass on **every** print/PDF and long-form document before calling it done — check for a stray word alone on its own line.

### Constrain the container first — don't let the line run long and then fix the orphan

The best orphan fix is structural: **choose a `max-width` that produces balanced line breaks**, not one that lets a sentence run nearly edge-to-edge and drop 1–3 words onto the next line. A reader walking all the way to the end of a long line just to pick up two stray words at the start of the next one is the same cognitive interruption as a printed widow — the medium doesn't change the problem.

In practice: if a text block's natural break point would leave fewer than ~4–5 words on the last line of a sentence, tighten the container width until the break lands further left and both lines carry substantial content. A slightly narrower max-width that produces clean, balanced wraps is always preferable to a wider container that looks "roomy" but generates constant near-orphans. The tighter container reads faster and feels more resolved.

This is a container-design decision, not a per-sentence fix — make it once per section and let it protect every line of copy inside it. Per-sentence kerning/`<br>` fixes (below) are still valid, but they're the last resort, not the first move.

Fix order (cheapest first):
1. **Tighten the container** — reduce `max-width` until the offending line break moves to a balanced position. One change, protects everything inside.
2. **Kerning second** — small negative `letter-spacing` on the paragraph (`-0.2px` to `-0.3px`) often pulls the word back up without touching the container. Invisible to the reader.
3. **Trim a word** that doesn't change the sentence's meaning, if kerning isn't enough.
4. **Manual `<br>`** between sentences is fine when a new sentence should visually start its own line anyway.

Re-render and re-check after any kerning change — confirm the line count actually dropped and nothing else shifted.

### Per-paragraph letter-spacing orphan fixes become visible inconsistencies under justify
The orphan-prevention trick — tighten `letter-spacing` on one paragraph (`-0.3px` → `-0.5px`) to pull a stray last word back up — works invisibly in left-aligned text because word spacing is a fixed character value. In **justified text**, the engine adds or subtracts word spacing on every line to fill the measure, and it does this *on top of* whatever letter-spacing the paragraph already has — so paragraphs with different letter-spacing settings receive different amounts of word-spacing stretch, and the tracking inconsistency becomes legible.
Fix: when switching a document to `text-align: justify`, remove all per-paragraph letter-spacing overrides and restore the global value everywhere. The wider column widths typical of a business letter (7+ inches of text measure) usually eliminate the original orphans without needing the override anyway — verify after removing, but don't pre-add them back.

### Justified text: left-align any paragraph containing an unbreakable string
`text-align: justify` fills each line by stretching word spacing — when a line contains an unbreakable string (a phone number using `&#8209;` non-breaking hyphens, a long email address, a URL), the justify engine has fewer valid break points and can stretch the preceding lines dramatically to compensate.
Fix: add `style="text-align:left"` to *just that paragraph* — the rest of the body stays justified. The contact/sign-off paragraph (phone + email) is the most common trigger in business letters; flag it the moment `&#8209;` or a bare email address appears in it.

### Hyphenated compound words are silent line-break points
A hyphen (`-`) is a valid line-break opportunity in every browser — "in-house" can break to "in-" / "house." just as readily as a space can, and the result reads as a widowed word even though no space caused it. Fix: replace the hyphen with a non-breaking hyphen (`&#8209;`) anywhere a compound word must stay together. Common candidates: in-house, one-off, full-bleed, day-to-day, best-in-class, follow-up. The non-breaking hyphen is visually identical to a regular hyphen.

### Bulleted/approach lists are a bigger orphan source than headlines or paragraphs
When checking a design for orphans, don't stop at the headline and body paragraphs — a multi-item bullet list (a "what we did" list, a features list) throws far more of them, because each bullet is short enough that a 2-line wrap very often leaves just one trailing word. Confirmed in practice: a first orphan pass caught headline/paragraph wraps and missed multiple single-word bullet orphans across the same document — each bullet had looked fine reviewed in isolation. Fix at the source rather than per-bullet: apply the `&nbsp;`-join-last-two-words technique (below) to *every* item in the list programmatically (a template function, not manual edits), so the whole class of failure is closed at once instead of chased instance-by-instance.

### On a responsive website (not a fixed-width print page)
A single fixed-width render can't catch this — the same headline/paragraph wraps differently at mobile/tablet/desktop widths, and an orphan at one width may not exist at another (or a new one appears only at a width you didn't check). Two changes to the process:
- **Check at multiple breakpoints**, not one render — at minimum mobile (~375px), tablet (~768px), desktop (~1280px+). Confirmed on real projects: several orphans only appeared at mobile width and were invisible at desktop, and one appeared only at desktop.
- **Kerning and manual `<br>` are the wrong tools here** — a `<br>` placed for one width breaks the layout wrong at another. Use `&nbsp;` between the last two words of the phrase instead (`the whole&nbsp;deal.`) — it's a hard, width-independent join: those two words simply can never separate onto different lines, at any viewport, so it never needs re-checking per breakpoint once applied.
- **Measuring which lines actually have a stray word is unreliable by eye/screenshot on a long page** — script it instead: walk the element's text nodes, get each word's bounding-rect `top` via `Range.getBoundingClientRect()`, and flag any element whose last line (words sharing the max `top`) has exactly 1 word while the element has more than 1 word total (a single-word element, e.g. a one-word heading or tag pill, is not an orphan and must be excluded via that `totalWords > 1` check). Re-run the same script at each breakpoint.

---

## 8. Measuring, not eyeballing

Any spatial request ("balance this", "move it up", "too close to the edge") — **measure pixel positions before and after**, don't eyeball.

Technique (Python/PIL on a rendered PNG):
```python
from PIL import Image
im = Image.open("rendered.png").convert("L")
w, h = im.size
px = im.load()

def row_has_content(y, x0, x1, threshold=235):
    return any(px[x, y] < threshold for x in range(x0, x1, 2))

# Scan a wide x-band (avoiding decorative corner elements that would
# contaminate the scan) to find every content-start/end transition:
prev = None
for y in range(0, h):
    has = row_has_content(y, int(w*0.20), int(w*0.72))
    if prev is not None and has != prev:
        print(y, "start" if has else "end", "-> in:", round(y / <dpi>, 3))
    prev = has
```
Maps every block's top/bottom edge, so gaps can be read directly. For "balance the gap above X with below Y": measure both gaps, compute combined slack, split evenly, move only the element(s) under your control.

**Watch for decorative elements contaminating the scan** — sanity-check by widening/narrowing the scan band.

**When two elements grow at once, re-check the total space budget jointly, not separately** — a bigger headline and a bigger illustration can each look fine in isolation and still collide together. Re-measure the full stack's height against the canvas after *any* size change.

**Scope leaks from global find/replace** — a plain string `.replace()` on a page with repeated near-identical blocks changes every occurrence, not just the one being edited, even when the text looks unique. Target the specific occurrence by index/position (`str.rfind` + slice) instead.

### Absolute size vs. relative fill-percentage
When comparing how prominent a subject looks across two **differently-sized** compositions, don't measure "% of its own container filled" — that's actively misleading. A subject filling 87% of a small container can be physically smaller, in absolute pixels, than one filling 80% of a container five times the area. **Render both at a shared/native scale (most ad pairs already share a pixel width) and compare absolute pixel dimensions directly.**

### Matching a stylistic treatment across differently-sized compositions
Don't copy a fixed pixel value (a gradient length, a spacing value) from a reference piece into a different-sized one. Measure it as a **percentage of its own containing zone** in the reference (a 115px fade in a 300px zone = 38.3%), then apply that same percentage to the new composition's corresponding zone.

### Color-threshold bounding-box detection — useful, verify against the raw asset
A skin-tone/color-threshold scan gives an objective bounding box for margin measurement, but has real failure modes: a loose threshold can catch warm-toned background texture (false positive), and any overlay/gradient sitting on the subject can push edge pixels below threshold (false negative, reads as more margin than exists). **Always cross-check a suspicious measurement against the RAW, un-composited asset** before trusting it.

### Crop-window direction with bleed is not intuitive
`output_position = (source_position − crop_origin) / scale` correctly predicts that decreasing a crop's origin moves a fully-visible subject down/right. But once part of the subject is intentionally cropped past a frame edge, the formula still holds while what's visually salient can look opposite — revealing more of one edge while cropping more off the other can read as "moved the wrong way" even when the rigid shift was correct. **When a directional-nudge result looks wrong to the user, regenerate the exact "before" state and do a direct raw-pixel before/after comparison** rather than arguing from the formula.

### Asymmetric margins are normal for a subject with a trailing extension
A fist with a wrist/arm reaching toward one corner will never have four equal margins — don't force it. Give the compact core clean space where nothing intrudes, let the extension's side run tighter, and match the same asymmetric convention across companion pieces in a set rather than chasing unachievable symmetry.

### Re-crop when a container's dimensions change
`object-fit: cover` will not preserve intentional framing — if a panel's proportions change, it silently center-crops the stale embed instead. **Regenerate the source crop from the original image at the container's new exact dimensions** whenever the container itself resizes.

### Swapping a photo inside an already-tuned multi-panel system
Don't re-derive each panel's crop from scratch. Measure the new subject's bounding box once, then apply the SAME relative framing ratio (subject-width ÷ zone-width) each panel already uses — this carries the established visual consistency over to the new asset automatically instead of re-litigating it panel by panel.

### Crop nudge/zoom formula — for the common "move it left/zoom in X%" requests
Given a crop box `(left, top, right, bottom)`:
```python
# Zoom in by pct, keeping the same center:
w, h = r-l, b-t
cx, cy = l+w/2, t+h/2
nw, nh = w*(1-pct), h*(1-pct)
new_box = (cx-nw/2, cy-nh/2, cx+nw/2, cy+nh/2)

# Nudge the subject LEFT or DOWN in the output — shift the crop origin
# by a fraction of its own dimension (moving the subject RIGHT/UP is the
# opposite sign):
new_left = left + int(width * shift_pct)   # subject moves left
new_top  = top  - int(height * shift_pct)  # subject moves down
```
Keep track of the actual crop box across a session (not just "the current photo") — every follow-up nudge/zoom request is a transform on that same box, not a fresh derivation.

### Luminance-threshold scanning direction depends on which is lighter, content or background
The standard technique (§8 top) assumes dark content on a light page (`px[x,y] < threshold`). For a dark-UI composition (light text/logo on a dark background — common on digital ads), **invert the comparison** (`px[x,y] > threshold`) or the scan silently finds nothing.

### Fitting a whole new content block onto an already-full fixed-height page
Different situation from the single-element estimate above: when a page is already using nearly all its vertical budget (little to no slack measured) and the new content is a full new block (its own label, photo row, and paragraph) rather than one added element, per-element estimation isn't enough — the new block's own footprint (label + photo + paragraph + margins) usually exceeds whatever slack exists. Don't shrink just one existing element to make room; that reads as one section being punished while its neighbors stay untouched.
- **Compress page-wide, proportionally, in one pass:** reduce photo row heights, section margins, label-row/CTA/band padding, and paragraph line-height/margins by similar small amounts across *every* section on the page (not just the one nearest the new content) — the page-wide rhythm shifts together so no single section looks squeezed relative to its neighbors.
- **Trim paragraph copy where a paragraph is running long relative to its neighbors** — shortening 1–2 sentences buys real vertical space and is usually invisible to the reader if the trimmed clause wasn't load-bearing.
- **Render and measure (§8) before and after** — confirm the freed space actually covers the new block's real rendered height (not just the estimate), and confirm the gap before the next fixed anchor (footer, page edge) is still comfortable, not just non-negative. Confirmed in practice: adding a full new section to an already-tight page required trimming photo heights and margins across all existing sections plus two paragraphs — the page still landed with *more* footer clearance than before, because the compression was spread evenly rather than concentrated.
- **Check for new widows/orphans (§7) after any paragraph trim** — shortening copy shifts line breaks and can introduce a stray last word where none existed before; re-check every paragraph touched by the compression pass, not just the new one.

### Redistributing leftover space after removing an element
Removing a decorative element (a divider line, a spacer) from a top-aligned flow can leave the freed-up space stranded as dead space wherever the flow happens to end — not evenly redistributed. If the container has a fixed height greater than its content's natural height, don't hand-tune individual margins to compensate. **Make the container `flex; flex-direction:column; justify-content:center`** (or `space-between`) instead — it centers/distributes the whole content group within the available space in one change, and stays correct through future content edits.

### Adding new content to an already-tight fixed-height page
Before adding a photo/banner to an existing panel on a page with a hard height limit (a print page, a fixed-size ad), estimate its added height against the remaining budget to the next fixed anchor (a footer, a page edge) — don't just append it and hope. Confirmed getting this wrong: a ~2.4in photo banner added to an already-tall navy panel pushed the panel's bottom edge to within a hair of the footer's fixed position, only caught after the fact. Re-measure (top of §8) immediately after adding any new block to a page that doesn't grow to fit its content.

### Locally reducing an overlay/scrim in one spot only
A global gradient overlay (a cover photo's darkening scrim) can't be locally lightened by editing its own gradient stops — that changes the whole horizontal/vertical band, not just the one subject. Use `mask-image` with a `radial-gradient` on the scrim element instead: a lower alpha at the target's position, ramping back to full (opaque) alpha outside a soft radius, reduces the scrim's effective strength only in that spot while leaving the rest of the gradient's mood untouched.
```css
.scrim{
  background: linear-gradient(...); /* unchanged */
  -webkit-mask-image: radial-gradient(ellipse 24% 17% at 50% 45%, rgba(0,0,0,0.78) 0%, rgba(0,0,0,0.78) 45%, black 100%);
  mask-image: radial-gradient(ellipse 24% 17% at 50% 45%, rgba(0,0,0,0.78) 0%, rgba(0,0,0,0.78) 45%, black 100%);
}
```
Get the target's `at X% Y%` position from the actual crop math (the subject's anchor position relative to the crop box), not from eyeballing the rendered page.

### Scale mismatch when compositing multiple stock figures into one scene
When combining separately-sourced stock illustrations (an adult hero figure plus supporting kid/prop figures, say) into a single composition, don't eyeball relative pixel heights. Measure the reference figure's actual body height — head-to-toe, excluding any raised arms or held props that extend above the head — as a fraction of its total asset canvas height, then apply real-world proportion ratios to size everything else: a child figure next to an adult ≈ 55–60% of the adult's *body* height (not the adult's full asset height, which is inflated by raised arms/a held prop); a small accent potted plant next to a person ≈ 30–35% of the adult's body height. Skipping this measurement produced a first-draft composite where child figures and a "small" decorative plant both rendered as tall as the adult hero — caught only on user review, not before.

---

## 9. Platform / tool / service catalog

| Tool / Service | Use for | Reliability |
|---|---|---|
| Headless Chrome (`--headless=new`) | HTML → PNG / PDF | Most reliable pipeline — default for print and digital deliverables |
| Adobe Stock (`asset_search`, `asset_license_and_download_stock`) | Sourcing photos/illustrations | Reliable; `.ai` downloads need local Ghostscript (§6) |
| Adobe `document_render_vector` | Cloud AI→PNG | Unreliable — async jobs don't resolve, no polling. Use local Ghostscript. |
| Adobe `asset_inline_preview` | Preview a stock asset | Intermittent failures — fall back to `curl` + Read |
| Adobe `image_remove_background` | Cutout for real photos (headshots, product shots) | Reliable, clean hair/edge detail — see §6. Local files need the `asset_initialize_file_upload` → `curl` PUT → `asset_finalize_file_upload` chain, not the file picker. |
| Adobe Express (native template editing) | Client-editable deliverables | Reliable |
| Adobe Express (HTML import) | — | Avoid for complex logos/gradients (§4) |
| Adobe Fonts (`find_fonts`, `get_fontkit_embed_url`) | Typography | Reliable when the font exists there |
| jsDelivr Fontsource CDN (`cdn.jsdelivr.net/npm/@fontsource/<font>/files/...woff2`) | Self-hosting a font that's on neither Adobe Fonts nor Google Fonts (e.g. Open Sauce Sans) | Reliable — confirmed working 2026-08-25; download the `.woff2` files and base64-embed rather than linking live, same as any other font in the HTML→PDF pipeline |
| Adobe `image_crop_and_resize` (`fit:"reframe"`, subject/prompt `focus`) | Subject-aware crop to a target aspect | Reliable; treat its `requires_expand_chain` suggestion as a conservative flag, not a mandate — see §6 note on re-aspecting to a narrow panel before accepting a big generative expand |
| Ghostscript (Homebrew) | Rasterizing `.ai`/`.eps` | Reliable, no cloud dependency |
| Inkscape (local) | — | Needs Ghostscript backend for EPS/PS |
| Python PIL/Pillow + numpy | Cropping/resizing/masking/measurement | Core workhorse — §6, §8 |
| pypdf | Page count & physical dimensions | Reliable, run after every PDF generation |
| pdftoppm / pdftotext (poppler) | PDF → PNG for review; text extraction | Reliable |
| "Magic Folder" (existing local pipeline) | Batch photo prep — cutouts, palettes, crop-guides | Check with user before rebuilding manually (§6) |
| Upscayl CLI (`/Applications/Upscayl.app/Contents/Resources/bin/upscayl-bin`) | AI photo upscaling, local | Reliable, confirmed working 2026-08-15 (§6) — closes the DPI gap noted below |

---

## 10. Time cost — where the hours actually go

Not yet formally tracked, but the recurring pattern: iterative AI-driven quality-check loops (render → measure → adjust → re-render) eat disproportionate time on **small, high-constraint canvases** — tight margins leave little room to maneuver, so each fix needs several precise, position-dependent adjustments instead of one confident move. A spacious canvas absorbs the same category of fix in far fewer rounds.

**When stuck in repeated one-off nudges on the same element, stop and propose (or ask for) an explicit, ordered 2–4 step plan** to execute once, verifying each step before the next — this reliably breaks a stalled iteration loop that reactive nudging hasn't.

---

## 11. Reference libraries — DESIGN.md collections

External design-system references — real companies' colors/type/component patterns reverse-engineered into structured markdown by third parties, meant to inspire/scaffold, never copied 1:1 (trademark/derivative-look risk on client work).

**In use:** a local `DesignSystemLibrary/` holds three connected sources plus one on-demand service — brand systems, aesthetic/vibe skills, and the official format spec. Full catalog in `connected-libraries.md`.

**Blend, don't clone:** pull 2–3 references per project, not one — mixing across sources is fine (a brand `DESIGN.md` plus an aesthetic skill, say) — and synthesize a custom system from them. A single source used alone risks the finished piece reading as a copy of that one brand.

---

## 12. Composition & layout — translating vague feedback into fixes

- **"Not enough breathing room around the logo"** → two changes: increase the logo's size *and* the margin around it. Size alone doesn't fix it.
- **"Don't like the photo/illustration boxed in"** → avoid a hard-edged card on top of the background. Go full-bleed, or remove the illustration's own backdrop and place it on one full-width brand-motif blob/shape (check specs.md for the established color first).
- **"The text needs to be more prominent"** → more than a font-size bump: increase headline size *and* add a color-accent word/phrase for hierarchy (60/30/10 + accent-color).
- **"Make line two start with X"** → don't fight the container width; insert a manual `<br>` at the exact break point. When feedback names the desired break point for *several* headlines at once (one word per headline, e.g. "1st should start on Y, 2nd on Z..."), that's the same instruction repeated — hardcode a `<br>` at each named point rather than adjusting font-size/width and hoping the natural wrap lands there; on a fixed-width print/deck page this is always safe (no responsive breakpoint to break at another width).
- **Background-color choice affects foreground contrast, not just mood** — check any white/light foreground details (shoe soles, highlights) still read against a new background before committing to it.
- **"Enlarge the photo" is ambiguous** — could mean grow the photo *panel* (layout change) or zoom the subject within the *same* panel (crop change). These produce very different results — ask which is meant if intent isn't obvious from context.
- **A directional nudge ("move it down/right") on a tightly-cropped subject** — apply a small, testable shift, then verify with an actual before/after pixel measurement, not memory or the formula alone (§8, bleed note).
- **Repositioning a subject within a crop (e.g. "move it southeast") trades directly against how much scenery survives** — when the source photo's aspect is already close to the target canvas aspect, there's little or no horizontal slack to shift a subject sideways without first zooming in (using less of the source height), which crops away background/scenery. If the user's real ask is "give more headroom" *and* "keep the scenery," don't max out the zoom to grab every last pixel of repositioning room — solve the headroom problem first (it usually needs less zoom than expected) and use only the modest slack that remains for the sideways shift, checking the result against how much scenery got cut before presenting it.
- **Check accent-color contrast against the photo's actual tone independently of the primary text color** — getting the dark/light primary-text swap right (per §5's palette-matching note) doesn't guarantee the *accent* color also reads. Confirmed on a pale dusty-pink sky: navy primary text was fine, but the standard brand accent green blended in and needed a darker, more saturated shade plus a subtle shadow. Sample pixels behind the accent word specifically, not just behind the headline as a whole.
- **Run an actual WCAG contrast-ratio check on every accent color before using it as text/small-UI-element color — don't eyeball it.** Confirmed in practice: a brand palette can contain colors that look fine as a logo swatch but are barely legible as text — light/pastel accents (an olive, a tan, a light blue) measured as low as **~1.5:1 against white** (need 4.5:1 for body text, 3:1 for large text/graphical UI elements), while darker accents in the same palette cleared 6–13:1 easily. Formula: linearize sRGB, compute relative luminance `L = 0.2126R + 0.7152G + 0.0722B`, contrast `= (Lmax+0.05)/(Lmin+0.05)`. Quick recipe once a color fails: keep hue/saturation, reduce HSL lightness by 25–55% (tune per hue) until it clears 4.5:1, and reserve that darkened version for anywhere the color carries text or sits behind white text; the original bright value is still fine for a *large fill area* paired with dark or white text, just not for text/icon color on its own.
- **"The text is hard to read" on a dark-background document** — two separate failure modes, both common:
  - **Opacity too low:** semi-transparent text (`rgba(242,237,228,0.3)`) reads legibly on screen at full brightness but prints muddy. On dark backgrounds, floor is ~`0.75` for secondary text and `0.84+` for body copy — anything below that range should be treated as invisible until proven otherwise. Confirmed: a pitch brief running 9pt body at `0.7` opacity needed both a size bump (9→10pt) and an opacity lift (→0.84) before it read cleanly.
  - **Hardcoded dark hex values:** a secondary-text color like `#6B6158` or `#3D3530` that was chosen for a light background becomes invisible once the design switches to dark — hard to catch because the class name doesn't signal the color. Always use CSS custom properties (`var(--text-2)`, etc.) for secondary/muted text rather than inline hex values so the token swaps correctly with the theme.
- **"The gradient looks off"** — sample pixel values across the transition first. If smooth (no true banding) but still reads wrong, the fade is likely going fully-opaque-to-fully-transparent in too short a distance. Fix: hold a light, uniform tint across the rest of the subject instead of forcing a full clear in too little room.
- **A vague size/position complaint on a small-format piece** — check it against any large-format companion piece at literal matching pixel scale (stack/place side-by-side at native resolution) rather than judging each in isolation.
- **Visible frustration ("this has taken all day")** is a signal to stop iterative micro-adjustment and propose an explicit ordered plan (§10).
- **"Move the logo to [a different corner]"** — after moving it, re-check clearance/contrast at the NEW position against whatever's actually there in the photo. A corner that was safe before (or that's safe on a companion piece) isn't automatically safe here — a sleeve, a dark garment edge, or just less negative space can collide with the mark. Same applies whenever the underlying photo crop changes with the logo position held fixed — re-verify, don't assume the old clearance still holds.
- **Two-column split layouts have three required alignment checks** — confirmed as separate, easy-to-miss failures: (1) both panels' `padding-top` must be identical for content to align horizontally across the split — a 0.15in mismatch is enough to be obvious; (2) the text column must use `justify-content: flex-start`, not `center` — centering pushes content down to the vertical midpoint rather than starting at the top; (3) when the footer's right text sits over the dark panel, it needs explicit light color (e.g. `color: rgba(255,255,255,0.75)`) — it's invisible dark-on-dark otherwise. Check all three before shipping any split-layout slide.
- **Verify the specific changed slide with `pdftoppm` before sending the full PDF.** Rendering the full PDF and relying on the user to spot a layout error they have to scroll to find is slower and wastes credits when the feedback comes back. After any layout edit: `pdftoppm -png -f <page> -l <page>`, inspect, then rebuild the full PDF only when the target slide looks right.
- **"Remove [a divider/decorative element] and balance the space"** — don't hand-tune margins to compensate. Switch the container to `flex; justify-content:center` (§8) so the whole content group redistributes into the freed space in one change.
- **"The blob/background shape looks strange on one side"** → an accent blob that doesn't span the full canvas width will show an abrupt, unnatural vertical edge where the container clips it. Make it span edge-to-edge (both left and right canvas edges) with only the top boundary drawn as the organic wave, rather than a closed or asymmetric shape floating off to one side.
- **"Make the logo/mark N× as large" on a badge with an icon inside a colored container** — ambiguous whether N× applies to the whole badge (container + icon scaled together) or just the icon graphic within a proportionally-adjusted container. Literally scaling the whole badge produced a wildly oversized result that swallowed part of the photo's subject. If context doesn't make it obvious, ask — don't guess and iterate blind, since the two readings diverge fast at anything beyond ~1.5×.
- **A vague "too big" / "too small" follow-up after a prior specific resize instruction ("I told you the wrong number I guess")** — don't just apply a fresh guess from scratch. Anchor to the last size that was confirmed too big/small in the *opposite* direction and split the difference, since the user is calibrating relative to what they already saw, not asking for an arbitrary new value.
- **"I don't understand this chart" / a data visualization feels unclear** → a new chart type, even a well-executed one, isn't automatically the right fix for "make this data more visual." If the client's existing materials already establish a component for presenting stats (a stat-card, say), reuse that SAME component — recolored for context if needed — rather than introducing an unfamiliar chart format. Confirmed: a bar-chart comparison was rejected in favor of reusing the existing white stat-card's shape just recolored for a dark panel — consistency of the visual language read clearer to the client than a more sophisticated but novel treatment. Technique: scope the existing class inside a parent selector (`.dark-panel .stat-card{...}`) to override only color values rather than rebuilding the component from scratch.
- **"Spacing feels inconsistent" on a repeated component that has visual variants** (a stat card whose icon is sometimes a number, sometimes an arrow, sometimes a plain dot) — the reported inconsistency is rarely padding/margin drift between copies; it's usually that each variant's icon sits in a *differently sized* box (a two-digit number naturally needs more width than a 10px dot), so the visual gap before the label text ends up different per variant even though the CSS gap value is identical. Fix: give every variant's icon the exact same fixed-size, centered slot (e.g. one `.icon-slot { width: Xin; display:flex; align-items:center; justify-content:center }` wrapping whatever glyph goes inside), so the icon-to-text gap is visually identical regardless of which variant is rendered.

---

## 13. Signature design concepts — house style

Patterns worth defaulting to across projects, independent of any one client's brand — what makes a piece recognizably intentional even when the palette changes. Different from process discipline (measuring, orphan-checking, asset workflow) elsewhere in this doc: those are *how* to work, this is *what* the work looks like. Add a new pattern here only once it's proven itself more than once — a single clever solution isn't a signature yet, a repeated instinct is.

**Precedence: explicit client spec > signature default > generic convention.** These are the baseline to reach for automatically — but a client's documented brand guide (specs.md, a formal style guide) wins whenever it explicitly specifies something that collides with one of these (e.g. a style guide that mandates centered headlines). Same hierarchy §2/§5 already use for colors and fonts: never override a documented client requirement with a default. Where the guide is silent, which is most of the time since these operate at a structural/craft level brand guides rarely cover, apply the signature without checking in first.

### Progressive disclosure by default
Every surface starts simple and rewards attention — never the reverse (a dashboard or page dumping full density up front). Default state shows the minimum: a headline number, one label, a calm, uncluttered surface. Interaction — hover, click, scroll — is what peels back the next layer of data or function, then the next, then the next: an oasis after oasis effect instead of a single flat reveal.
- Term: this is **progressive disclosure**, a standard UX principle (Nielsen Norman Group) — surface only what's needed for the task at hand, defer detail until it's asked for.
- Not just hover states: a stat-card showing one number that expands to a breakdown on click, a nav that only reveals sub-items once its parent is active, a report whose executive summary expands into supporting detail — same principle, different trigger.
- Design test for any new screen: cover everything but the default state — does it still read as calm and complete on its own? If the default state feels bare without the hidden layers, disclosure isn't truly progressive, it's just hidden.

### Two-row asymmetric growth (not single-row wrap-to-width)
**Only applies once a set is large enough to need a second row in the first place.** A small set that comfortably fits across one full-width row at a normal, readable size should just stay a single row — no orphan risk exists yet, so there's nothing for this pattern to solve, and forcing a two-row split on 2–4 boxes is manufactured complexity, not signature. Trigger it functionally, not by a fixed count: the moment a set would wrap into a second row on its own (given the container width and a sane minimum box size), that's the exact moment orphan risk appears, and that's when this pattern kicks in instead of a naive wrap. "Roughly 5 or more items" is a reasonable mental shortcut for when that point tends to arrive, not a hard cutoff — a few very wide boxes can hit it sooner, many slim ones later.

Default grid/flex behavior fills one row edge-to-edge until full, *then* wraps a new row below — every row equal-weight, growth is single-file. Once the pattern is triggered, we do the opposite: **two rows grow together as a pair**, with the **top row carrying more width by default** (a wider, more dominant band up top; a narrower, secondary band below), and both extend left-to-right in lockstep as content is added — never row 1 filling to completion before row 2 starts.
- Mental model: add one "column pair" at a time (a wide top cell + a narrower bottom cell), not one item at a time into a single flowing row.
- Starting ratio to refine from, not a fixed law: top row ≈ 60–65% of the shared width/weight, bottom row ≈ 35–40%.
- Implementation is project-specific (paired CSS grid rows, two flex rows fed from the same indexed data, etc.) — the point is the *row relationship*, not one specific markup pattern.
- **Guard against the default failure mode: boxes within one row must sit left and right of each other, spanning the row's full width — never stacked vertically.** Plain block-level elements stack vertically by default (each takes its own line), so without an explicit horizontal layout (`display:flex` in a row direction, or a CSS grid with column tracks) a row can silently collapse into a single vertical column instead of extending sideways. The only vertical relationship in this whole pattern is the top row sitting above the bottom row — everything *within* a row stays horizontal. Check this visually on every implementation, not just in the markup: render it and confirm boxes are actually extending left-to-right, not falling into a stack.

### Negative space, balanced to the limit the content allows
Balance the negative space on every piece as close to fully as the actual content permits — this is a standing goal on every project, not a fix reached for only when feedback calls it out. **Balanced does not mean symmetric or evenly distributed** — it means no region reads as accidentally cramped or accidentally empty relative to what it's holding; deliberate asymmetry (left-weighted composition below, asymmetric margins on a trailing-extension subject, §8) is still balanced when the remaining space is distributed with intent. The imbalance to eliminate is the *unintentional* kind: a gap that's there because nothing was done about it, not because it was chosen.
- Practical check: after any layout is "done," run the §8 measuring pass specifically on gaps/margins, not just element positions — sum the negative space per region and ask whether the distribution reads as chosen or leftover.
- When content genuinely can't fill a space evenly (a short headline over a tall panel, a photo that doesn't match its frame's aspect ratio), don't force artificial padding-balance — resize the element, adjust the container, or use the two-row/left-weighted patterns above to absorb the imbalance structurally, rather than eyeballing extra whitespace into place.
- This is the umbrella principle behind several existing fixes already logged individually: "not enough breathing room" (§12), flex `justify-content:center` to redistribute freed space after removing an element (§8), and re-measuring a joint space budget when two elements grow at once (§8) — all of those are negative-space balancing in service of this same goal.

### Left-weighted composition
We consistently place more visual mass toward the left than the centered/right-balanced symmetry most sites and decks default to. This is a deliberate asymmetry, not an accident of content — once established for a project, hold it consistently rather than drifting back to centered balance under pressure to "even things out." Same instinct as the asymmetric-margins call for a cropped subject with a trailing extension (§8) — applied at the scale of a full composition instead of one image.

### Avoid center-aligned text as the default
Default to left-aligned text (running body copy, headlines, labels, captions) rather than reaching for center alignment as the safe/neutral choice — it isn't neutral, it's the single most common alignment in generic design and works directly against left-weighted composition above. Left alignment also gives multi-line text a consistent starting edge, which is easier to scan than the ragged, shifting left edge center-alignment produces.
- Genuine exceptions exist and should be used without hesitation when the content calls for them: a single short standalone line (a one-line stat + label, a logo lockup), formal/ceremonial pieces where centered symmetry is the convention (certificates, formal invitations), or a deliberately symmetric hero moment where the composition itself is centered, not just the text.
- The test: would this still need centering if the layout around it were left-weighted? If the surrounding composition is already asymmetric, centered text inside it usually reads as a mismatch, not a rest point.

### Short, discrete text is rooted at its companion content, extending away from it
Magazine-column pattern this is drawn from: a two-column spread, left column holding a header + body copy, right column holding a tall vertical image. The **header** right-aligns (flush right, ragged left) against the column boundary nearest the image. The flush edge is the root — it sits planted against the image, touching it — and the text itself extends *away* from that edge, out into the column, like it grew out of the image and reached leftward. It is not reaching toward the image; the image is where it starts, not where it's headed. The **body copy underneath stays left-aligned**, because once the reader is into running text, a normal reading experience (consistent left edge, easy scanning) matters more than the directional effect.
- Not limited to headers — the same treatment applies to any short, discrete text element with a companion piece of content to root against: pull-quotes, review snippets, captions, short descriptive blurbs, stat labels, credits.
- The line that matters is **length/purpose, not element type**: a short, self-contained piece of text (a line or two, meant to be taken in as a unit) can carry this treatment; a large body of running text (multiple paragraphs, meant to be read straight through) should not — legibility wins there, so it stays left-aligned regardless of what's beside it.
- Within one composite block (e.g. header + paragraph together), only the short/discrete piece takes the treatment — a following long-form paragraph still reverts to left-aligned even if the short piece above it is right-aligned.
- Get the direction right: the aligned (flush) edge is always the one touching the companion content — that's the root. The ragged edge is always the one extending away, into open space. If the ragged edge ends up touching the content instead, the effect reads backwards — like the text is drifting away from its anchor rather than growing out of it.
- This isn't a contradiction of "avoid center-aligned text" above — it's still a left/right choice, never a centered one. That rule says *don't default to center*; this rule says *which of left/right to pick, and when a piece of text is short/discrete enough to earn the treatment at all*.
- Mirror case: when the companion content sits to the *left* instead, the short text flips to left-aligned — its flush-left edge now roots against the content on its left, and it extends rightward away from it.

### Zero strays at every scale, not just the sentence
§7 covers word-level orphans/widows in running text. The same intolerance holds a level up: a grid or row that ends with exactly one block stranded alone — visually orphaned from its row-mates — is the same defect at a larger scale, and it's part of what makes a layout look noticeably *off* even when a viewer can't say why. When a set of growing blocks would leave a single item alone on a new row, treat it the same as a widowed word: adjust the count, adjust the two-row growth ratio above, or fold the last item back into an existing row — never let a block sit by itself.

### Alliterative headers and copy
Not a layout rule, but the same "recognizably ours" instinct applied to words — favor alliterative phrasing in headers, section titles, and callouts when a natural option exists ("Progressive disclosure," "Zero strays," "Left-weighted"). Don't force it where it would twist meaning or read as gimmicky; it's a tiebreaker between equally-good phrasings, not a constraint that overrides clarity.

*Open list — add the next repeated instinct as it surfaces, don't force one prematurely.*

---

## 14. Vector logo reuse & multi-instance lockup pages

Recurring pattern on brand-guide/lockup pages: the same logo file needs to appear many times on one page (on-white, on-dark, on-blue swatches; a Don'ts grid) — several pitfalls only surface at that point.

**Strip `<defs>/<style class="stN">`, set fill as a presentation attribute on the root `<svg>` instead.** Illustrator-exported SVGs define fill via a CSS class (`.st0{fill:...}`) — reusing that same file twice on one page makes the *second* instance's class silently override the first's color (browsers keep one definition per class name, even across separately embedded `<svg>` documents in the same page). [[feedback_inline_svg_reuse]] already flagged the collision; the fix that scales cleanly to many instances: regex out the `<defs>` block, strip `class="stN"` from every path, and set `fill="#hex"` directly on the root `<svg>` tag — `fill` is an inherited SVG presentation attribute, so every child path inherits it with zero per-path edits. Also strip any `id="Layer_1"`-style id Illustrator adds; duplicate ids across instances don't break rendering but are worth cleaning up.

**Flex swatch grids: don't hardcode the logo's pixel/inch width inside a `flex:1` cell.** A fixed-width child forces the whole row to overflow past its container once enough swatches are added — invisible with 1–2 items, breaks silently at 3+. Fix: add `min-width:0` to the flex child (flex items default to a content-based minimum width that ignores `flex:1` otherwise) and size the logo itself at `width:100%` of its swatch, not a fixed inch value, so it scales to whatever width the flex distribution actually gives it.

**A "don't crop the logo" demo needs `overflow:hidden` on a non-flex ancestor.** `display:flex` gives children `flex-shrink:1` by default — a fixed-width logo centered inside a flex clipping box just shrinks to fit instead of overflowing to get visibly clipped, silently defeating the demo. Use a plain block container (`position:relative; overflow:hidden`) with the logo absolutely positioned inside at its full natural size instead.
