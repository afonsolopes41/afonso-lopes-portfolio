# afonsolopes.dev

Personal site and portfolio. A dark, ASCII-rendered page whose opening
sequence is a 3D dragonfly taking off from a branch — scrubbed frame by
frame by the scroll wheel, rendered entirely as text characters through a
WebGL post-process pass.

No framework, no build step, no dependencies to install. Open `index.html`
over HTTP and it runs.

**Repository:** <https://github.com/afonsolopes41/afonso-lopes-portfolio>  
**Live:** _not deployed yet — see [Deployment](#deployment)._

---

## Contents

- [What it does](#what-it-does)
- [Running it locally](#running-it-locally)
- [Architecture](#architecture)
- [The parts that are easy to break](#the-parts-that-are-easy-to-break)
- [Editing the content](#editing-the-content)
- [Tuning hooks](#tuning-hooks)
- [Analytics](#analytics)
- [Deployment](#deployment)
- [Browser support](#browser-support)
- [Credits and licensing](#credits-and-licensing)

---

## What it does

| | |
|---|---|
| **Hero** | A glTF dragonfly, lit by a two-rig setup, rendered to a texture and re-drawn as ASCII glyphs in a fragment shader. The scroll wheel scrubs the animation clip directly, so scrolling back runs it backwards. |
| **Scroll** | Transform-based smooth scrolling. The document scrolls natively; the content is drawn at an eased position that lags behind it. That lag is the weight. |
| **Flight budget** | The whole opening runs over exactly **26 wheel notches**, at which point section 02 sits 25% from the top of the screen. The page layout is *derived* from that number, not the other way round. |
| **Break-up** | From notch 19 the character field disperses into columns and clears. |
| **Pointer** | Characters near the cursor swap for a set of louder glyphs in the accent orange, driven by movement rather than presence. |
| **Second view** | A dragon over the summer-school section, sharing the same glyph atlas and shader through a separate renderer. Always turning; each scroll notch kicks it, and scrolling up turns it the other way. |
| **Cursor** | Twenty spring chains of fifty nodes, drawn as additive quadratic ribbons. |
| **Gallery** | 35 photos on a seamless drifting rail; clicking one opens a lightbox with keyboard arrows, swipe, and neighbour preloading. Thumbnails total 3 MB and are lazy; the full versions (10 MB) load only on open. |
| **Sections** | 01 About · 02 Trajectory · 03 Projects · 04 Future AIoT · 05 Signal · 06 Domains · 07 Contact. Anything added **above** section 02 comes out of the flight budget; everything else is free. |
| **Boot / fault** | A real progress readout over the ~1.9 MB of runtime and geometry, and a designed failure screen for when something does not arrive. |

---

## Running it locally

It must be served over HTTP. Opening the file directly with `file://` will
fail — the glTF loader and its Draco worker are both blocked by the
origin rules.

```bash
python -m http.server 8000
```

Then open <http://localhost:8000>.

To reach it from a phone on the same network:

```bash
python -m http.server 8000 --bind 0.0.0.0
```

…and browse to `http://<your-lan-ip>:8000`.

---

## Architecture

### Why one file

`index.html` is a single ~280 KB document containing the markup, the
stylesheet and every script. That is a deliberate choice, not an
accident:

- **There is no build step to break.** The file that is edited is the
  file that ships. No bundler, no transpiler, no lockfile, no CI.
- **One request.** Everything except the two models and the three.js
  runtime arrives in the first response.
- **The tuning is inseparable from the markup.** Nearly every constant in
  here was arrived at by measurement against reference frames, and the
  comments explaining *why* a number is what it is are worth more next to
  the number than in a separate document.

The trade is real: no module boundaries, no tree-shaking, no unit tests
around the internals. If this grows much further, splitting the scripts
out and adding a bundler is the right next move. It is not there yet.

### The modules inside it

Each is a self-contained IIFE. A throw in one cannot take down the
others.

| Module | Responsibility |
|---|---|
| **Boot** | First script in the body, ahead of the CDN tags so it can watch them load. Tracks real progress, reveals the page, and owns the failure screen. |
| **Renderer** | three.js scene, the two lighting rigs, the model load, the framing solver, the wing animation, the ASCII post-process pass, and the scroll→clip scrub. The largest module by far. |
| **Dragon view** | A second renderer with its own scene and render target, sharing the first one's glyph atlas and shader source. |
| **Portfolio layer** | Everything that is not 3D: the nav, the menu sheet, scroll reveals, the photo rail, the live link graph, the contact board. |
| **Cursor** | The ribbon trail, on its own canvas overlay. |

### The ASCII pipeline

```
scene ──► WebGLRenderTarget ──► fragment shader ──► screen
                                      │
                                glyph atlas
                        (baked 1:1 with the device cell)
```

The atlas is a horizontal strip of glyphs baked at exactly the size they
are drawn at. This matters more than it looks: an earlier version baked
at 64 px and displayed at 6 px, and the 10× minification silently ate the
thin strokes — the ring of an `o`, the stem of a `/` — so pushing *more*
cells toward the bright end of the ramp produced *less* ink on screen.

---

## The parts that are easy to break

Read this before changing layout.

### 1. The 26-notch budget

`layoutFlight()` sizes the `#flight` spacer so that section 02 lands
exactly 26 wheel notches (2600 px) down. It does that by measuring
everything above section 02 and giving the flight whatever is left over.

**Anything added above section 02 is spent out of that budget.** The hero
alone is `100vh`, so on a tall window the budget runs out. A
`@media (min-height: 1100px)` block tightens the spacing on tall screens
for exactly this reason. Verified at 0 px error from 620 px to 1440 px of
viewport height; above roughly 1450 px it degrades gracefully — the
animation simply takes a notch or two longer.

If you add content above section 02, re-check the error:

```js
document.querySelector('#sectionWriting').offsetTop - (26 * 100 + innerHeight * 0.25)
```

It should be 0 (±1 for sub-pixel rounding).

### 2. `#scroller` is `position: fixed`

The page content is translated every frame rather than scrolled. Two
consequences:

- Anything that must stay put — the nav, the menu sheet, the overlays —
  has to live **outside** `#scroller`, or it will ride the transform.
- `IntersectionObserver` is unreliable against a transformed ancestor, so
  scroll reveals measure rects on a frame instead.

### 3. Non-indexed geometry needs an attribute called `position`

three.js bails out of `renderBufferDirect` for any non-indexed geometry
with no attribute by that exact name — silently, with no error and no
draw call. Naming a fullscreen triangle's attribute `aPos` produces a
blank screen and a clean console.

### 4. `Box3.setFromObject` walks descendants

The wings are children of the thorax mesh, so a "body box" measured that
way came out 5.51 units where the thorax's own geometry is 0.77. The
framing solver uses each mesh's own `geometry.boundingBox` transformed by
its world matrix.

### 5. Mirrored meshes have inverted normals

`THREE.FrontSide` culls them away completely. Both models use
`DoubleSide`.

---

## Editing the content

| What | Where |
|---|---|
| The hero word | `var P_TEXT` — also update the `aria-label` and `.ptext-sr` span |
| Chengdu photos | 35 photos in `photos/`, each at two sizes: `chengdu-NN.jpg` (760px, the rail) and `chengdu-NN-full.jpg` (1600px, the lightbox). Order and captions live in the `SHOTS` array. See `photos/README.txt`. |
| Portrait | `photos/afonso.jpg` |
| Age | `data-age="YYYY-MM-DD"` — computed at load, never typed |
| Email | `var ADDRESS`, plus the `mailto:` hrefs |
| Colours | `:root` — `--sig` orange, `--sig2` red, `--sig3` indigo |
| Projects | The `.projlist` block in section 03 |
| Flight length | `FLIGHT_NOTCHES`, `GLITCH_START_NOTCH`, `NOTCH_PX` |

---

## Tuning hooks

All dev surfaces are gated behind `?tune` in the query string and are
`null` without it, so nothing is exposed in production.

| Hook | For |
|---|---|
| `window.__ascii` | Lighting rigs, shader uniforms, framing, single-frame stepping, the dragon view |
| `window.__site` | Menu, reveals, photo rail, link graph |
| `window.__cursor` | Ribbon state, single-frame stepping |
| `window.__boot.debug` | Boot bar and log state |

Two more query flags:

- `?boot=4000` — adds a synthetic four-second task so the loading screen
  can actually be looked at. It is otherwise finished before it is shown.
- `?fault=CODE` — renders the failure screen on demand.

---

## Analytics

Cloudflare Web Analytics, sitting commented out at the bottom of
`index.html` until it has a token.

It is there instead of Google Analytics for one concrete reason: GA4
sets cookies, and cookies mean a consent banner under GDPR. A cookie
wall is a poor first thing to show somebody who followed a link from
LinkedIn. Cloudflare's beacon is cookieless and does not fingerprint, so
no banner is required.

To turn it on:

1. <https://dash.cloudflare.com> → Web Analytics → **Add a site**
2. Paste the deployed hostname
3. Copy the token into `data-cf-beacon` and delete the two `<!--` `-->`
   markers around the tag

It is left commented rather than sitting there with a placeholder
because an unconfigured beacon still fires, still fails, and leaves a
permanent `ERR_CONNECTION_REFUSED` in the console of the live site.

---

## Deployment

Static hosting, no server needed. The repository root is the web root.

**GitHub Pages:** Settings → Pages → deploy from `main` / root.
**Netlify / Cloudflare Pages:** connect the repository, leave the build
command empty, publish directory `/`.

`404.html` is picked up automatically by all three.

Check after deploying:

- The site is served over **HTTPS** — the clipboard API needs a secure
  context, and there is a fallback but the real path is better.
- `og:image` is set, or the link renders as a bare grey card when shared.

---

## Browser support

Chrome, Edge, Firefox and Safari, current versions. Requires WebGL.

- **`prefers-reduced-motion`** is honoured: the cursor ribbons do not
  start, and the reveals and boot animation are disabled.
- **Touch:** the cursor follows the finger. Its listeners are passive and
  never call `preventDefault`, so scrolling is untouched.
- **Hidden tabs:** every loop stops.

Known gaps, honestly: the menu sheet does not trap keyboard focus, there
is no skip link, and the decorative micro-labels (`SEC—01`) sit at about
2.2:1 contrast. Everything carrying actual information clears WCAG AA at
4.9:1 or better.

---

## A note on the photo "protection"

The gallery blocks the context menu over photos, blocks dragging, makes
the images unselectable, and lays a transparent shield over each one so
"save image as" resolves against an empty div. Verified: a click over a
photo lands on the shield, not the `<img>`.

**None of that protects the photos, and it cannot.** A browser has to
receive the bytes to draw the picture, so anyone looking at a photo
already has it — the network tab, the disk cache and the print-screen
key are all outside anything a page can reach. What the measures buy is
that the casual routes do not land on an image, which covers people who
would have grabbed one without thinking. Against anyone who opens
devtools they do nothing.

The two measures that do survive extraction are a visible watermark and
shipping a resolution not worth having. The second is in place: 1600px
on the long edge is fine on screen and useless for print.

---

## Credits and licensing

**This design is derivative.** The layout, the ASCII treatment and the
scroll-driven opening are modelled closely on
[dragonfly.xyz](https://www.dragonfly.xyz/). The 3D models,
the typography and all copy are this site's own, but the visual language
is theirs and it would be dishonest to present it otherwise.

Also used:

- [three.js](https://threejs.org/) r128 — MIT
- [React Bits](https://reactbits.dev/) — `ParticleText` and `SwarmCursor`
  were ported from React to vanilla JS
- The cursor ribbon effect, ported from a widely circulated
  `useCanvasCursor` hook

The **code** in this repository is MIT licensed (see `LICENSE`). The
**photographs** are not — they are personal and are not licensed for
reuse.
