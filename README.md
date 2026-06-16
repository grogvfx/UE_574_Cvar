# Unreal Render — CVar Manager (UE 5.7.4)

A self-contained, offline web tool that turns a few scenario choices into a curated,
memory-aware set of **rendering console variables (cvars)** for Unreal Engine 5.7.4 —
then exports them as console commands, a `DefaultEngine.ini` block, or a ready-to-run
**Movie Render Graph Python script**.

Built and tuned for fast, clean, flicker-free renders on **NVIDIA A40-16Q (16 GB)** cards,
but every value is exposed and adjustable.

> Every cvar name and default in the tool is taken from a real `r.*` dump of **UE 5.7.4**
> (`cvars.csv`, 9,276 cvars). Nothing is invented — if a cvar isn't in that dump, the tool doesn't use it.

---

## Files

| File | What it is |
|------|------------|
| `render-cvar-manager.html` | **The tool.** Open this in a browser. |
| `cvars-data.js` | Generated dataset (the full 5.7.4 dump as JSON). Powers the *Browse* tab and *Pin any cvar* autocomplete. Must sit next to the HTML. |
| `cvars.csv` | The source UE 5.7.4 cvar dump (`NAME,VALUE,SETBY`). Used to regenerate `cvars-data.js`. |
| `ue_render_presets.json` | Example of an exported presets file (created by the tool's *Export all*). |
| `render_cvars_*.py` | Example of an exported Movie Render Graph script (created by the tool's *Export .py*). |

---

## Opening the tool

Just **double-click `render-cvar-manager.html`** — no install, no server, works offline.

The *Build Preset* tab works from a plain `file://` open. The *Browse CVars* tab and the
*Pin any cvar* autocomplete load `cvars-data.js`; most browsers allow that for a sibling file,
but a few (strict Chrome configs) block local file loads. If the Browse tab says the data
didn't load, serve the folder instead:

```bash
# from inside this folder
python -m http.server 8000
# then open http://localhost:8000/render-cvar-manager.html
```

---

## Build Preset tab — workflow

Set up your shot on the left; the recommended cvars, a VRAM estimate, and warnings update live on the right.

1. **Render systems** — toggle what's actually in your render: Sky Atmosphere, Volumetric
   Clouds/Fog, Nanite, Virtual Shadow Maps, Distance Fields, Reflections, Ray-Traced Shadows,
   Ambient Occlusion, Translucency, Subsurface Scattering, Bloom, DoF, Motion Blur, Lens Flare,
   Hair/Groom, Auto Exposure.
2. **Global illumination** — *No GI* / *Lumen SW* (software Global-Distance-Field / "voxel" path)
   / *Lumen HW-RT* (uses the A40 RT cores; needs Ray Tracing enabled in Project Settings + restart).
3. **Scene profile** — three master sliders: **Scene scale** (product ⟶ open world),
   **Speed ⟷ Quality**, and **VRAM budget** (default 14000 MB = leave ~2 GB headroom on a 16 GB card).
4. **Shadow / Lumen / Voxelization tuning** — fine sliders that each drive *several* related cvars,
   with a live readout of the resulting values:
   - *Shadow:* Close-up detail, Soft-shadow quality, Large-scene range.
   - *Lumen:* Final gather, Reflections, Temporal stability (anti-flicker).
   - *Voxelization:* Voxel detail, Voxel range (Global Distance Field).
5. **Content in shot** — Static, Crowd, Characters, Niagara, Foliage. Each adjusts the recommendations
   (e.g. crowds enlarge the VSM page pool and enable the skin cache).
6. **Output / AA** — TSR / TAA / None, screen percentage (keep **100** for final renders), and a
   **Cinematic / MRQ** switch that adds convergence warm-up frames, fully-loaded textures, and deeper
   temporal accumulation.
7. **Advanced control** — per-system override dropdowns (Nanite, VSM, Lumen, Textures, Translucency).
   `Auto` follows the scenario; pick a value to force it. Pool overrides (Nanite/VSM/Textures) are
   locked out of the auto memory-pressure valve.
8. **Pin any cvar** — type *any* cvar name from the dump (autocomplete), give it a value and a scope,
   and it's appended to the output. Unknown names are flagged.
9. **Presets & notes** — save/load/delete named presets in browser local storage, attach field notes
   ("tested — VSM still flickered, raised pages to 24k"), and **Export all / Import** as JSON to move
   presets between machines.

### VRAM estimate & memory pressure
The bar is a **rough** estimate (base + textures + Nanite + VSM + fog + Lumen + DF + clouds).
If it exceeds your budget, the tool auto-scales the texture/Nanite/VSM pools down to fit (the
"global pressure valve") and tells you. Always confirm real usage in-engine with `stat GPU`.

---

## Output — Project vs Runtime split (important)

Recommendations are split into two scopes, because they go in **different places**:

- **Project** (`⚙` badge) — read at engine startup, **cannot** be changed at render time
  (e.g. `r.RayTracing`, `r.DynamicGlobalIlluminationMethod`, `r.DistanceFields`, `r.Lumen.TraceMeshSDFs`).
  Set these in **Project Settings / `DefaultEngine.ini`** and restart the editor.
- **Runtime / render-graph** (`▶` badge) — safe to apply live, via the console or the Movie Render
  Graph / Queue job.

### Export buttons
- **Copy commands** — `cvar value` lines, grouped by scope (paste into the console or an exec file).
- **Copy .ini** — a `[/Script/Engine.RendererSettings]` + `[ConsoleVariables]` block for `DefaultEngine.ini`.
- **Download .txt** — the console commands as a file.
- **Export .py (Render Graph)** / **Copy .py** — the Python script (below).
- **Save + .py** (Presets card) — saves the preset *and* downloads its script in one click.

---

## The Movie Render Graph Python script

`Export .py` generates a `unreal.MovieGraphScriptBase` subclass that **applies your runtime cvars on
job start and cleanly restores the editor's previous values on job finish** — so you never paste cvars
into a render by hand again.

What it does:
- Builds a `RENDER_CVARS` dict from your preset's **runtime** cvars (project cvars are listed as
  comments only — they can't be set at render time).
- `on_job_start`: captures each cvar's current value, then applies yours via `execute_console_command`.
- `on_job_finished`: restores every captured value.
- The class name includes your preset name (e.g. `RenderCVarOverrides_MyHeroShot`) so multiple presets
  don't collide.

### How to use it
1. Build your preset, name it, click **Export .py** (or **Save + .py**).
2. Drop the `.py` into your project's `Content/Python/` folder.
3. In your **Movie Render Graph**, add an **Execute Script** node (or set it on the graph's script
   settings) and point its **Script Class** at the generated class name.
4. Render — cvars apply on job start, revert on finish.

> **Headless note:** the script uses `unreal.EditorLevelLibrary.get_editor_world()`, which targets the
> in-editor world (the usual MRG case). If you render headless via the command line (`-game`), that world
> accessor differs — ask for a headless-safe variant if you need one.

The generated script follows the same `MovieGraphScriptBase` override pattern
(`on_job_start` / `on_job_finished`) as a hand-written render-graph callback class.

---

## Browse CVars tab

Search and filter the full 5.7.4 dump:
- Free-text search by name.
- Category quick-filters (Lighting & GI, Shadows, Atmosphere & Fog, Nanite & Geometry, Post Process,
  Anti-aliasing, Textures & Streaming, Reflections, Materials, Hair, Animation, FX/Niagara, Physics).
- Filter by `SetBy` source.
- Shows each cvar's 5.7.4 default and, where the tool curates it, a short description.

---

## Regenerating `cvars-data.js` from a new dump

If you get a fresh cvar dump (e.g. from a different engine version), replace `cvars.csv`
(format: `NAME,VALUE,SETBY`) and regenerate:

```bash
python - <<'PY'
import csv, json
rows=[]
with open('cvars.csv', newline='', encoding='utf-8', errors='replace') as f:
    r=csv.reader(f); next(r)
    for row in r:
        if not row: continue
        rows.append({"n":row[0],
                     "v":(','.join(row[1:-1]) if len(row)>=3 else (row[1] if len(row)==2 else '')),
                     "s":row[-1] if len(row)>=2 else ''})
with open('cvars-data.js','w',encoding='utf-8') as o:
    o.write("window.CVARS="); json.dump(rows,o,separators=(',',':'),ensure_ascii=False); o.write(";")
print("wrote", len(rows), "cvars")
PY
```

> The curated recommendation logic and defaults inside the HTML are hand-tuned for 5.7.4. A new dump
> updates the *Browse* tab and pin autocomplete, but the recommended values may need review for other versions.

---

## Caveats

- The recommendations are **informed starting points**, not guarantees — every project, scene, and
  light rig differs. Validate with test renders.
- VRAM numbers are **rough**. Trust `stat GPU`, `r.Nanite.Streaming.Debug 1`, and the VSM page-pool
  overflow messages in the log over the estimate.
- This dump has **no DLSS/NGX cvars** (plugin not installed), so the AA path here is TSR/TAA only.
- Presets and notes live in your browser's local storage for this file — use *Export all* to back them up.
