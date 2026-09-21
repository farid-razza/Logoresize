# Logo Squaring Service

Turns any logo into a **square PNG** that Google's Performance Max logo slot will accept.

The logo itself is never touched. It is placed, unchanged, on a square background — the
square is the empty space around it.

```
  before                        after
  ┌──────────────────────┐      ┌──────────────┐
  │  RIVERSIDE PLUMBING  │  ->  │              │
  └──────────────────────┘      │  RIVERSIDE   │
    1200 x 300                  │  PLUMBING    │
                                │              │
                                └──────────────┘
                                  1200 x 1200
```

Implements the **Logo Squaring Rules, 19 September 2026**.

---

## Quick start

```bash
python -m venv .venv
.venv\Scripts\activate            # Windows
pip install -r requirements.txt
copy .env.example .env            # optional - the defaults work as-is
python app.py                     # http://127.0.0.1:8000/docs
```

There is no API key to configure. Everything runs locally with Pillow.

---

## The rule

```
INPUT   W = logo width in pixels
        H = logo height in pixels

STEP 1  If W and H cannot be read, STOP. Reason: unreadable.
STEP 2  L = the LARGER of W and H
STEP 3  EDGE = L;  if EDGE < 128 then 128;  if EDGE > 1200 then 1200
STEP 4  Make a canvas EDGE wide and EDGE tall, in the background colour.
STEP 5  Scale the logo by min(EDGE/W, EDGE/H), but NEVER above 1.0.
        Centre it on the canvas.

OUTPUT  A square PNG, EDGE x EDGE.
```

**The output size is different for every logo.** It is not fixed. A 600 x 200 logo becomes
a 600 x 600 square, *not* 1200 x 1200 — stretching 600 px of detail across 1200 px would
make the logo look worse, and Google is happy with 600 x 600.

`min(..., 1.0)` in STEP 5 is the whole safety rule: **a logo is never enlarged**, because
enlarging blurs it.

### Worked examples

| Logo that arrives | Square we make | What is inside it |
|---|---|---|
| 1000 x 1000 | 1000 x 1000 | unchanged |
| 2000 x 2000 | 1200 x 1200 | shrunk to 1200 x 1200 |
| 1200 x 300 | 1200 x 1200 | 1200 x 300, centred |
| 600 x 200 | 600 x 600 | 600 x 200, untouched |
| 4000 x 500 | 1200 x 1200 | shrunk to 1200 x 150 |
| 100 x 50 | 128 x 128 | 100 x 50, untouched |
| 40 x 20 | 128 x 128 | 40 x 20, untouched |
| 120 x 300 | 300 x 300 | 120 x 300, centred |
| corrupt file | none | refused, unreadable |

---

## The five checks

They run in this order. Each is cheaper than the next, so a logo that will be refused is
refused before any effort is spent on it.

| | Check | What happens |
|---|---|---|
| 1 | Can the file be read at all? | If not, refused. **This is the only refusal.** |
| 2 | Is it already square? | To within 5%, not to the pixel |
| 3 | How big is the square? | `clamp(max(W,H), 128, 1200)` |
| 4 | Which background colour? | Filename → pixels → white |
| 5 | Make it | Pad. Never crop, never stretch, never enlarge |

**No logo is ever refused for being small.** Google's 128 px minimum applies to the
*image*, not to the logo inside it — a 40 x 20 logo is centred on a 128 x 128 square.

Square, horizontal and vertical logos all run the **same code path**. There is one
procedure, not three.

---

## Pad, never crop, never stretch

Three ways to make a wide image square. Only one is acceptable.

| | What it does | Result on a wordmark |
|---|---|---|
| Crop | Cuts off what doesn't fit | "Riverside Plumbing" becomes "ersid". Unusable. |
| Stretch | Squashes it to fill the square | Short and fat. Unusable. |
| **Pad** | Puts it unchanged on a bigger square | Exactly as the business drew it. **Correct.** |

One scale factor is applied to both axes, so the artwork cannot be squashed. The canvas is
never smaller than the scaled artwork, so nothing can be cut off.

---

## The background colour

The empty space has to be some colour, and it is not always white. A white logo on a white
square is a blank square.

Decided per logo, in this order:

1. **What the file says about itself** — `logo-on-dark`, `logo-white`, `logo-reverse`.
2. **What the pixels say** — and this splits in two:
   - a **transparent** logo is ink and nothing else, so it gets the *opposite* plate or it
     disappears;
   - an **opaque** logo already sits on a background its designer chose, so the padding
     *continues* that background. (A dark wordmark on its own white rectangle, padded with
     near-black, would become a white rectangle floating on black.)
3. **White**, when neither gives an answer.

Only two colours are ever used: `#ffffff` and `#0d0d0d`. There is no brand-colour matching
— guessing a brand's colour and getting it slightly wrong looks worse than an honest white
square.

**Transparency is always flattened** onto the chosen colour. Left in place, the finished
square would show whatever sits behind it in the advert instead of the colour we picked.

---

## The API

### `POST /api/v1/logo/square`

The main endpoint. Give it a URL; it works out the size and colour itself.

```bash
curl -X POST http://127.0.0.1:8000/api/v1/logo/square \
     -H "Content-Type: application/json" \
     -d '{"logo_url":"https://cdn.site/logo.png"}'
```

```jsonc
{
  "status": "ok",
  "request_sent": { "size": 600, "background": "#ffffff" },
  "source": {
    "source_width": 600, "source_height": 200,
    "aspect_type": "horizontal", "source_transparent": true,
    "edge": 600, "scale_factor": 1.0, "enlarged": false,
    "artwork_size": [600, 200],
    "background_hex": "#ffffff",
    "background_reason": "transparent source, ink luminance 40 of 255; contrasting plate"
  },
  "output": { "width": 600, "height": 600, "format": "PNG",
              "bytes": 12043, "within_5mb": true },
  "provenance": { "machine_made": true, "business_upload": false,
                  "copy_of": "https://cdn.site/logo.png" },
  "image_base64": "..."
}
```

`size` and `background` are optional. Send them to override the arithmetic:

```jsonc
{ "logo_url": "https://cdn.site/logo.png", "size": 512, "background": "#0d0d0d" }
```

### `POST /api/v1/logo/resize`

Same thing, but accepts a **file upload** (`file`) as well as `image_url`. Add
`return_image=true` to get the PNG streamed back instead of JSON.

```bash
curl -X POST http://127.0.0.1:8000/api/v1/logo/resize \
     -F "file=@logo.png" -F "return_image=true" -o square.png
```

### `POST /api/v1/logo/batch`

Up to 200 URLs or local filenames in one call.

```bash
curl -X POST http://127.0.0.1:8000/api/v1/logo/batch \
     -H "Content-Type: application/json" \
     -d '{"items":["https://cdn.site/a.png","acme.png"]}'
```

### `GET /api/v1/logo/health`

Config and readiness.

---

## Provenance

Every square is labelled as machine-made and as a copy of the original, so the platform
never files it as the business's own upload. The label is in three places:

- the JSON response, under `provenance`
- **inside the PNG itself**, as text chunks — so it survives the file being saved anywhere
- response headers on `return_image=true`: `X-Logo-Machine-Made`, `X-Logo-Copy-Of`

> **Needs confirming:** the rules say *what* the label must convey but do not name the
> fields. `machine_made`, `business_upload` and `copy_of` are this service's names and
> should be checked against the platform's asset library.

---

## What is rejected

| Check | Rejected as |
|---|---|
| The file cannot be read | `unreadable` (HTTP 422) |
| Width does not equal height | `not_square` |
| Size outside 128–1200, or not the size asked for, or over 5 MB | `out_of_range` |

The last two are a guard on our own output — they cannot fire for a logo squared here,
which is the point. They exist so a bug shows up as an error rather than as a bad logo.

---

## Checking it works

```bash
curl -X POST http://127.0.0.1:8000/api/v1/logo/resize      -F "file=@some-wide-logo.png" -F "return_image=true" -o square.png
```

A 600 x 200 logo must come back as a 600 x 600 PNG with the artwork untouched at
600 x 200 in the middle. Anything else is a bug.

Before it was handed over, the service was run against 93 real logos pulled from a
customer sheet — 82 horizontal, 9 square, 2 vertical, aspect ratios from 1.08:1 to
12.9:1. The test harness that produced those numbers is not part of this repo; it lives
in the project history if it is needed again.

### Result on the 93-logo test set

| | |
|---|---|
| Squared correctly | 93 of 93 |
| Logo identical to the source artwork | **0 of 255 difference on all 93** |
| Logos enlarged | **0** |
| Proportions changed | **0** (whole canvas matches an independent rebuild, 0 of 255) |
| Different output sizes produced | 71, from 128 to 1200 |
| Transparent sources flattened | 72 of 93 |
| Time per logo | ~2 ms |

---

## Configuration (.env)

All optional — the defaults are the rules.

| Key | Default | Purpose |
|---|---|---|
| `EDGE_MIN` | `128` | Google's floor for the image |
| `EDGE_MAX` | `1200` | Google's recommended size; nothing above helps |
| `SQUARE_TOLERANCE` | `0.05` | how close to square counts as square |
| `BG_LIGHT` | `#ffffff` | the light plate |
| `BG_DARK` | `#0d0d0d` | the dark plate |
| `MAX_OUTPUT_MB` | `5` | Google's file limit |
| `MAX_UPLOAD_MB` | `25` | largest upload accepted |
| `TRIM_BORDER` | `false` | keep `false` — trimming changes the logo's framing |
| `SAVE_OUTPUT` | `true` | write each square to `data/output/` |

---

## Layout

```
app.py           FastAPI entrypoint
router.py        all routes
controller.py    request validation + response shaping
service.py       the five checks and the squaring itself
config.py        .env-driven settings
```

Five files. `data/input/` and `data/output/` are created automatically on first run.

The squaring lives in four small functions in `service.py`:

| Function | Does |
|---|---|
| `compute_edge(w, h)` | STEP 2–3: the square's size |
| `choose_background(img, name)` | STEP 4: which plate, and why |
| `artwork_scale(w, h, edge)` | STEP 5: the scale, capped at 1.0 |
| `square_canvas(img, edge, hex)` | STEP 4–5: pad, centre, flatten |

---


**Publishing consent.** The business is meant to be shown both versions and asked before a
campaign publishes. That gate is not built here; the rules call it the next piece of work.
