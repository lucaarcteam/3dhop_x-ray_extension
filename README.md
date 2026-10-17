# 3DHOP X-Ray Extension

**Version 2.1** — real-time x-ray opacity slider for 3DHOP.

[![Release](https://img.shields.io/github/v/release/lucaarcteam/3dhop_x-ray_extension?sort=semver&label=latest%20release&color=1785c8)](https://github.com/lucaarcteam/3dhop_x-ray_extension/releases)

An **x-ray view** extension for [3DHOP](https://www.3dhop.net) (3D Heritage Online Presenter).

It loads two aligned meshes of the same subject — an outer *container* surface and an
inner *skeleton* — and lets the user switch the outer surface to a see-through
"x-ray" state so the inner structure becomes visible, without moving the camera.

The demo subject is a **left hand** — the same hand as Roentgen's first radiograph:
the outer skin shell plus the bones (and two rings) inside.

```
      normal (x-ray OFF)                 x-ray ON
   ┌────────────────────┐          ┌────────────────────┐
   │   opaque hand      │          │  faint hand shell  │
   │  (skeleton hidden) │   ──►    │  skeleton visible  │
   └────────────────────┘          └────────────────────┘
```

## Live demo

<p><a href="https://lucaarcteam.github.io/3dhop_x-ray_extension/x-ray.html" target="_blank" rel="noopener">🖐️ <strong>Open the live hand demo</strong></a> &nbsp;<em>(opens in a new window)</em></p>

The demo runs on **GitHub Pages** (static files, no server-side code): orbit / zoom /
pan with the 3DHOP trackball, toggle x-ray from the toolbar, drag the intensity
slider for the real-time transparency, and measure distances in millimetres with the
measure tool.

## Releases

- **v2.1** *(latest)* — opacity slider (fully opaque ↔ fully transparent, classic
  alpha compositing) and corrected millimetre calibration.
  https://github.com/lucaarcteam/3dhop_x-ray_extension/releases/tag/v2.1
- **v2.0** — real-time transparency slider, measure tool in mm, Röntgen
  demo subject (left hand with rings).
  https://github.com/lucaarcteam/3dhop_x-ray_extension/releases/tag/v2.0
- **v1.0** — original x-ray view between two models (Luca Bezzi / Arc-Team).
  https://github.com/lucaarcteam/3dhop_x-ray_extension/releases/tag/v1.0

Every release ships its own **Source code (zip / tar.gz)** archive.

## Features

- Toggle **x-ray** on/off from the toolbar.
- **Opacity slider**: sets the outer shell opacity in real time — **100% = fully
  opaque**, **0% = fully transparent** (only the skeleton), smooth fade in
  between (default 90%).
- Standard 3DHOP tools: measurements (in mm), orbit / zoom / pan trackball, plane
  sections (clipping) and fullscreen.
- Runs entirely in the browser; no server-side code. Just static files.

## Running it

3DHOP loads the meshes with `XMLHttpRequest`, so the page must be served over
HTTP — opening the file with `file://` will not work. Any static server is fine:

```bash
# from the project folder
python3 -m http.server 8000
# then open http://localhost:8000/x-ray.html
```

## Project structure

```
3dhop_x-ray_extension/
├── x-ray.html              # the demo page (UI + scene setup + x-ray logic)
├── js/                     # 3DHOP core (unmodified upstream) + helpers
│   ├── presenter.js        # 3DHOP engine
│   ├── nexus.js, ply.js    # model loaders
│   ├── spidergl.js, jquery.js, trackball_*.js
│   └── init.js             # 3DHOP UI helpers (toolbar, sections, fullscreen)
├── skins/dark/             # toolbar icons (incl. x-ray.png / x-ray_on.png)
├── stylesheet/             # 3DHOP css
└── models/
    ├── hand_skin.ply       # outer container  (skin)
    ├── hand_bones.ply      # inner structure  (skeleton)
    └── hand_rings.ply      # rings, always opaque (Roentgen homage)
```

The x-ray logic lives entirely in `x-ray.html` — the 3DHOP core (`js/presenter.js`)
is **stock upstream code**, never edited on disk; the only tweak is a runtime
`gl.blendFunc` rewrite in the page (explained below). Transparency itself uses the
native `Presenter.setInstanceTransparencyByName()` API.

## How it works

Three instances share the scene:

- `external_model` — the skin shell (starts opaque);
- `internal_model` — the bones (always present, but hidden inside the shell);
- `ring_model` — the rings, always opaque (they stay visible in the x-ray view).

Transparency is applied with the native
`Presenter.setInstanceTransparencyByName('external_model', true, redraw, alpha)`
API, where `alpha` is the shell **opacity** (1 = opaque, 0 = invisible).

Stock 3DHOP renders transparent instances **additively**
(`gl.blendFunc(SRC_ALPHA, ONE)`): the object behind is never attenuated, so the
shell could never look truly opaque and there was a hard visual jump when
switching back to the normal (opaque) pass. `x-ray.html` rewrites **only** that
one blend call to standard alpha compositing (`SRC_ALPHA, ONE_MINUS_SRC_ALPHA`),
so the opacity fades smoothly and 0% really is invisible. The overlays used by
the measure / picked-point tools also call `(SRC_ALPHA, ONE)`, but with
`depthFunc GREATER`; they are detected via the depth function and left additive.
The shared 3DHOP core files are never edited.

### Real-time opacity slider

The **X-Ray intensity slider** (bottom centre) maps linearly onto the shell
opacity in real time:

- **100% = fully opaque** — normal, depth-tested render (skeleton hidden);
- **0% = fully transparent** — only the skeleton (and the rings) show;
- in between: a smooth fade (`alpha = pct / 100`).

The default is **90%** (a slightly transparent hand). The toolbar **x-ray**
button stays as an on/off shortcut: off → 100% opaque, on → the last value below
100% (default 90%). This is the same tool used in the Similaun-mummy project.

> Implementation detail: the 3DHOP API treats `alpha = 0` as "not provided", so
> the slider clamps the value to a tiny `XRAY_ALPHA_MIN = 0.0001` (visually
> indistinguishable from 0).

### Measure tool (mm)

The standard 3DHOP measure tool is wired to `presenter._onEndMeasurement`.
3DHOP reports measurements in raw (model) units and the models are normalised
(max dimension = 1.0) to the source hand's real size, so the raw measure is
converted to millimetres via `MEASURE_MM_PER_UNIT = 149.27` (the source hand's
max dimension along its longest axis, in mm). The hand is a free reconstruction
inspired by the historic plate, so its absolute metric values are indicative.

## Default subject: a nod to Roentgen

The demo hand wears two rings on the ring finger — a small homage to the **first
medical radiograph** (*Hand mit Ringen*, W. C. Roentgen, 1895), which showed his
wife's left-hand bones and her rings:

- a **thin flat band** (wedding band) on the proximal side;
- a **solitaire** on the distal side — its band swells towards a raised bezel
  (*castone*) that holds a small gem on the back of the hand.

The rings are kept opaque, so they appear both on the solid hand and in the x-ray
view, exactly like on the historic plate.

## Models and credits

The two demo models are derived from the **Z-Anatomy** open 3D atlas of human
anatomy (same coordinate space for skin and bones, so they line up):

- **Z-Anatomy** — © Gauthier Kervyn and contributors, licensed
  **CC BY-SA 4.0** (<https://www.z-anatomy.com>).
- Z-Anatomy in turn builds on **BodyParts3D / Anatomography**
  (DBCLS, **CC BY-SA 2.1 JP**, <https://lifesciencedb.jp/bp3d/>).

The PLY exports in `models/` are therefore provided under **CC BY-SA 4.0**.
If you replace them with your own meshes, keep the two files aligned in the same
coordinate system.

## License

- **Code** (this extension and the bundled 3DHOP): **GPL-3.0-or-later** — see `LICENSE`.
- **Models** in `models/`: **CC BY-SA 4.0** (see credits above).

## Roadmap

- ~~**v2.0** — real-time transparency slider~~ *(done)*.
- ~~**v2.1** — opacity slider (opaque ↔ transparent) + measure calibration fix~~
  *(done)*.
- Optional polish: localise the remaining toolbar tooltips to Italian.

## Version notes

**v2.1** (this revision):

- **opacity slider**: 100% = fully opaque, 0% = fully transparent, dragged live —
  the shell is composited with standard alpha blending (a runtime `gl.blendFunc`
  rewrite in the page) instead of the stock additive "ghost", so the opacity fades
  smoothly (the same tool as the Similaun-mummy project);
- **measurement fix**: `MEASURE_MM_PER_UNIT` overestimated every measurement by
  ~20% (it wrongly divided the raw model measure by the viewer's scene-radius
  scaling); it is now `149.27` (raw model unit → mm).

**v2.0**:

- added the **X-Ray intensity slider** (bottom centre, always active): dragging it
  enables x-ray and updates the shell's transparency in real time;
- fixed slider dragging against `#3dhop`'s `mousedown` preventDefault;
- added the standard 3DHOP **measure tool** with millimetre calibration and a
  localised "Lunghezza misurata" box (the stock `measure`/`measure_on` icons and
  `measureSwitch` from `init.js` were already present);
- new background featuring Roentgen's first radiograph behind the 3D hand.

**v1.0** — cleaned up and made runnable:

- replaced the placeholder meshes with aligned real models (skin + skeleton) and
  added the Roentgen rings;
- restored the original transparency value (`XRAY_ALPHA = 0.9`);
- kept the author's own x-ray toolbar icon and added a matching "on" state (amber
  highlight, coherent with the 3DHOP dark skin);
- restored the classic full-window 3DHOP viewer area;
- fixed the toolbar action bug (`x-ray` no longer also triggered fullscreen);
- added a real `xraySwitch()` toggle (state reflected in the toolbar);
- removed the extra always-visible transparent instance (single container instance,
  transparency toggled directly) — no more ghosting/z-fighting;
- documented the project, fixed the folder layout, added `.gitignore`.

## Authorship

Version **1.0** of this extension was developed by **Luca Bezzi**
([Arc-Team](https://www.arc-team.com/)) as part of the **3D documentation project of the
Similaun (Ötzi) mummy** and its associated Copper Age equipment — the context that
originally produced the x-ray view application. Reference:

> Bezzi, L.; Bezzi, A.; Gietl, R.; Moraes, C.; Vallazza, E.; Guareschi, E.E.; Tauber, M.;
> Peschel, O.; Pernter, P.; Putzer, A. *Preserving the Past: The 3D Documentation of Ötzi,
> the Iceman Mummy, and Its Archaeological Context.* **Heritage** 2026, *9*(9), 339.
> <https://www.mdpi.com/2571-9408/9/9/339>

The **v2.x** releases — the real-time transparency slider, the measure tool and the
Röntgen-inspired demo subject — are the implementation and evolution of that first
application.

## Credits

3DHOP is developed by the Visual Computing Lab, ISTI-CNR (<https://www.3dhop.net>).
