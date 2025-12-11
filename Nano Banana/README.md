# README

# README

This folder defines the JSON contracts that Nano Banana uses for architectural renders a **single room at a time**. The JSON is an LLM-facing scene model: Nano Banana reads and writes it, and the render engine uses it to create images.

It is intentionally **per-room**, not a whole-house schema.

---

## 0. Pipeline (per room)

For each room you want to work with, the intended flow is:

1. Extract a room model (once per room)

   * Use the prompt in [Image → JSON Extractor](./Image%20-%3E%20JSON%20Extractor.md).
   * Inputs: one floor plan image + one or more interior reference images of the *same* room.
   * Output: a single JSON object describing the room geometry, walls, views, and elements (`elems`) with clear-out flags (`rm`/`repl`).

2. Optionally add canonical 3D-like views

   * Use the prompt in [3D Modelling View Creator](./3D%20Modeling%20View%20Creator.md).
   * Inputs: the JSON from step 1 + a **focus area** description and a `focus_key`.
   * Output: the *same* JSON, but with extra orbit-style camera views and matching `render.outs` entries. Only `views` and `render.outs` are changed.

3. Use the JSON as the contract for design / clear-out prompts

   * Downstream prompts (for Nano Banana) take this JSON as input.
   * You update `elems` (e.g. remove or add furniture) and maybe add more `views` / `render.outs`.
   * The render engine does not guess the room; it trusts this JSON as the source of truth.

This README is **explanatory only**. For exact field-by-field rules, always refer to the two prompt specs:

* [Image → JSON Extractor](./Image%20-%3E%20JSON%20Extractor.md) – canonical single-room scene extractor.
* [3D Modelling View Creator](./3D%20Modeling%20View%20Creator.md) – canonical orbit-view generator around a focus area.

---

## Image → JSON Extractor

This extractor prompt converts a **single-room floor plan + interior photos** into a compact JSON description that Nano Banana can use as a geometric + semantic scene model.

The extractor itself is LLM-based (e.g. Nano Banana in “text+vision” mode). The resulting JSON is what you feed into downstream render / design prompts.

---

### 1. High-level structure

The JSON has six main parts:

1.1 `legend` – mini dictionary of category codes and flags.  
1.2 `proj` – project metadata.  
1.3 `media` – input image filenames.  
1.4 `space` – room geometry (footprint and walls).  
1.5 `views` – camera positions for each reference image (and later, render views).  
1.6 `elems` – all room elements (floor, walls, windows, furniture, appliances, clutter).  
1.7 `render` – requested outputs + simple keep/remove rules by category.

Very compact, but enough for coherent geometry and repeated render passes.

---

#### 1.1 `legend`

```json
"legend": {
  "cat": {
    "arch": "architecture surfaces (floor, wall, ceiling)",
    "open": "openings (window, door, arch)",
    "fix":  "fixed elements (ceiling fan, AC, built-in light)",
    "furn": "furniture",
    "appl": "appliances",
    "dec":  "decor",
    "grp":  "group of small items"
  },
  "pos_rel": {
    "on": "on one wall",
    "between": "between two walls or at a corner",
    "floor": "free-standing on floor",
    "ceil": "attached mainly to ceiling"
  },
  "vis": {
    "f": "fully visible",
    "p": "partially visible",
    "o": "mostly occluded"
  }
}
```

Purpose:

- Makes the JSON self-describing.  
- Safe, compact explanation of short codes for any model or human reading it.

---

#### 1.2 `proj`

```json
"proj": {
  "name": "Room_Renovation_Clearout"
}
```

Just a human label for the current room / scenario. Useful for logging or multi-room batches, but not strictly required for geometry.

---

#### 1.3 `media`

```json
"media": {
  "floor": "floorplan.jpg",
  "refs": [
    { "id": "v1", "file": "view_1.jpg" },
    { "id": "v2", "file": "view_2.jpg" },
    { "id": "v3", "file": "view_3.jpg" }
  ]
}
```

- `floor`: floor plan file name.  
- `refs`: interior reference images (each gets an id used later under `views`).

The extractor LLM sees these images directly; the JSON only stores filenames + ids so later steps can refer to them.

---

#### 1.4 `space` – geometry and walls

```json
"space": {
  "geom": {
    "pts": [[0,0],[1,0],[1,0.6],[0.4,1],[0,1]],
    "H": 2.6,
    "walls": [
      { "id": "w1", "seq": 1, "p0": 0, "p1": 1, "label": "entrance_wall" },
      { "id": "w2", "seq": 2, "p0": 1, "p1": 2, "label": "window_wall_long" },
      { "id": "w3", "seq": 3, "p0": 2, "p1": 3, "label": "back_wall" },
      { "id": "w4", "seq": 4, "p0": 3, "p1": 4, "label": "short_wall" },
      { "id": "w5", "seq": 5, "p0": 4, "p1": 0, "label": "stair_wall" }
    ]
  }
}
```

- `pts`: polygon vertices of the room footprint, **normalised** to [0,1] x [0,1].  
  - Index i in this array is used as `p0` / `p1` in walls.  
- `H`: room height.  
- `walls`: each wall is a segment between two vertices (`p0`→`p1`) and has:
  - `id`: stable identifier (`"w1"`, `"w2"`, …).  
  - `seq`: perimeter order index (1-based).  
  - `label`: short human description.

This gives Nano Banana a reusable, explicit floor shape + wall segmentation, rather than only implicit geometry in text.

---

#### 1.5 `views` – cameras for each reference / render

```json
"views": [
  {
    "id": "v1",
    "ref": "v1",
    "cam": {
      "rel": "corner",
      "w1": "w1",
      "w2": "w2",
      "xy": [0.05, 0.05],
      "h": 1.6
    }
  },
  {
    "id": "v2",
    "ref": "v2",
    "cam": {
      "rel": "wall",
      "w1": "w2",
      "w2": null,
      "xy": [0.85, 0.20],
      "h": 1.6
    }
  }
]
```

- `id`: view id used elsewhere (`render.outs[*].from`).  
- `ref`: which `media.refs[*].id` this view corresponds to (for reference images).  
- `cam`:
  - `rel`: `"corner"`, `"wall"`, or `"free"` – qualitative camera placement.  
  - `w1`, `w2`: which wall(s) the camera relates to.  
  - `xy`: camera floor position in the same normalised coordinates as `space.geom.pts`.  
  - `h`: camera height (m).

For reference images, the extractor fills these. For synthetic render views, you or a design prompt can add more entries with `ref: null`.

---

#### 1.6 `elems` – all elements in the room

Each entry describes one element or a group of elements.

Example:

```json
"elems": [
  {
    "id": "floor_1",
    "cat": "arch",
    "type": "floor",
    "pos": {
      "rel": "floor",
      "w1": null,
      "w2": null,
      "xy": [0.5, 0.5],
      "h": 0,
      "sz": [null, null, null]
    },
    "views": [],
    "d": "tile_terracotta_30x30_matt_black_grout",
    "rm": false,
    "repl": null
  },
  {
    "id": "win_1",
    "cat": "open",
    "type": "window",
    "pos": {
      "rel": "on",
      "w1": "w2",
      "w2": null,
      "xy": [0.8, 0.1],
      "h": 1.0,
      "sz": [1.6, 1.2, 0.2]
    },
    "views": [
      { "v": "v1", "bb": [0.55, 0.80, 0.30, 0.65], "vis": "f" }
    ],
    "d": "window_casement_black_aluminium_clear_glass",
    "rm": false,
    "repl": null
  }
]
```

Key fields:

- `id`: stable element id.  
- `cat`: high-level category (matches `legend.cat`).  
- `type`: specific element type.  
- `pos`:
  - `rel`: relationship to walls/floor/ceiling (`on`, `between`, `floor`, `ceil`).  
  - `w1`, `w2`: associated wall ids.  
  - `xy`: floor position in the normalised footprint.  
  - `h`: vertical position (m).  
  - `sz`: approximate `[width, height, depth]` (m) or `[null,…]` if unknown.

- `views`: for each reference view this element appears in:
  - `v`: view id (`"v1"`, `"v2"`, …).  
  - `bb`: bounding box in normalised image coordinates `[xmin, xmax, ymin, ymax]` with x,y in [0,1].  
  - `vis`: `"f"`, `"p"`, `"o"` (fully / partial / occluded).

- `d`: short compressed description for materials / colour / shape.  
- `rm`: whether this element should be removed in a “clear-out” render.  
- `repl`: what to show instead if removed (`"plain_wall"`, `"extend_floor_and_walls"`, etc.).

---

#### 1.7 `render` – outputs and keep/remove rules

```json
```json
"render": {
  "outs": [
    {
      "id": "r1",
      "from": "v1",
      "lens": { "t": "wide", "f": 18, "fov": 90 }
    },
    {
      "id": "r2",
      "from": "v2",
      "lens": { "t": "wide", "f": 18, "fov": 90 }
    }
  ]
}
```

* `outs`: which views to render.

  * `id`: render id.
  * `from`: which `views[*].id` to use as camera.
  * `lens`: simple lens spec.

Clear-out behaviour is controlled only by per-element flags in `elems`:

* `elems[*].rm`:

  * `true`  = remove this element in a clear-out / redesign render.
  * `false` = keep this element.

* `elems[*].repl`:

  * If `rm=true`: short description of what should appear instead (e.g. `"extend_floor_and_walls"`, `"empty_floor"`).
  * If `rm=false`: `null`.

There are **no** category-level clear-out rules in `render`.


You can override `rules` in later design stages (e.g. keep furniture, change only decor).

---

### 2. Usage

#### 2.1 Step 1 – Extraction (this repo file)

1. In a Nano Banana (or other LLM) chat:
   - Paste the full text from `Image -> JSON Extractor.md`.  
   - Attach:
     - One floor plan image of a single room.  
     - 1–4 interior reference photos of that same room.

2. Ask the model to run the extractor once:
   - It should output ONLY the JSON object described above (no explanations).

3. Save the JSON output and the image set together.

This JSON is now your **room model**.

---

#### 2.2 Step 2 – Design / clear-out / pseudo-3D work

In a new Nano Banana chat:

1. Provide:
   - The room JSON from step 1.  
   - Any subset of the original reference images you want to stay aligned with.

2. Ask for changes in terms of this model, for example:

   - “Add a kitchen front along wall w2, under window win_1, about 2 m long.”  
   - “Remove all elems with cat in rm_cat and render r1 and r2 again.”  
   - “Create two new views orbiting around the kitchen area with ids v_kitchen_left, v_kitchen_right and matching renders r_kitchen_left/right.”

3. The design prompt updates:
   - `elems` (adding new cabinets, appliances, decor, etc.).  
   - `views` and `render` (new cameras, new outputs).  

4. You then use the updated JSON + images as the input for Nano Banana’s actual rendering pipeline.

---

### 3. Design principles

- **Geometry first** – `space.geom` + `xy` + `H` ensure the room shape is explicit and reusable.  
- **Categories for bulk ops** – `cat`, `keep_cat`, `rm_cat` and `rm`/`repl` give you simple clear-out and redesign rules.  
- **Compact but English-like** – field values like `furn`, `appl`, `tile_terracotta_30x30_matt_black_grout` remain readable for humans and LLMs while staying token-efficient.  

This setup lets you use the same JSON both as:

- a compact intermediate representation for Nano Banana; and  
- a stable “contract” between extraction, design, and rendering stages.

<br><br><br><br><br>
  
## 3D Modelling View Creator

This spec takes an existing **single-room** JSON (from the Image → JSON Extractor) and adds a small family of pseudo-3D “orbit” views around a chosen focus area.

### What it does

* You choose a focus area (for example “front of the kitchen run”, “TV wall”, or a specific element / wall) and a short `focus_key` (e.g. `kit1`, `tv`, `desk`).

* The LLM computes a `focus_xy` point inside the room and places up to four cameras on a small arc around it with fixed naming:

  * `v_<focus_key>_front`
  * `v_<focus_key>_left`
  * `v_<focus_key>_right`
  * `v_<focus_key>_over` (optional overhead / high angle)

* For each view it also adds a matching render output in `render.outs`:

  * `r_<focus_key>_front`
  * `r_<focus_key>_left`
  * `r_<focus_key>_right`
  * `r_<focus_key>_over` (if used)

* It **never** changes walls, elements, or other pre-existing views / renders. It only adds or updates entries for the given `focus_key`.

### Inputs

To use this spec you need:

* The full room JSON produced by the Image → JSON Extractor (single-room only).
* A text instruction that describes:

  * The `focus_key` you want to use.
  * How to locate the focus:

    * element id(s), or
    * wall id(s), or
    * an explicit normalised coordinate `[x, y]` in [0,1]×[0,1], or
    * a short description like “middle of the south wall” or “centre of the TV wall”.

### Outputs

The result is the **same JSON object**, but with:

* New or updated `views` whose ids start with `v_<focus_key>_...`.
* New or updated `render.outs` whose ids start with `r_<focus_key>_...`.

Nothing else in the JSON should be changed unless you explicitly ask for it in your instruction.

### Prompting pattern (how to run it)

For a given room:

1. Paste the current room JSON.
2. Paste the full text from [3D Modelling View Creator](./3D%20Modeling%20View%20Creator.md).
3. Add an instruction such as:

   * “Using `focus_key = "kit1"`, add orbit views around the main kitchen wall and update `views` and `render.outs` accordingly. Return the full updated JSON only.”

If you need several focus areas (e.g. kitchen vs TV vs desk), call the spec multiple times with different `focus_key` values. Each call only touches the `v_<focus_key>_*` and `r_<focus_key>_*` entries for its own key.

For the exact geometric rules (how `focus_xy` is chosen, radii, angles, heights), see the 3D spec file itself.

- Geometry-aware – everything is grounded in `space.geom.pts` and the normalised `xy` frame.
- Token-efficient – the runtime JSON remains compact; heavy explanation lives in these .md files, not inside the JSON.
