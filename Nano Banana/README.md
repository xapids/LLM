# README


## Table of Contents

[Image → JSON Extractor](https://github.com/xapids/LLM/tree/main/Nano%20Banana#image--json-extractor)
- [1. High‑level structure](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#1-high-level-structure)
  - [1.1 legend](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#11-legend)
  - [1.2 proj](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#12-proj)
  - [1.3 media](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#13-media)
  - [1.4 space](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#14-space--geometry-and-walls)
  - [1.5 views](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#15-views--cameras-for-each-reference--render)
  - [1.6 elems](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#16-elems--all-elements-in-the-room)
  - [1.7 render](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#17-render--outputs-and-keepremove-rules)
- [2. Usage](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#2-usage)
   - [2.1 Step 1 – Extraction (this repo file)](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#21-step-1--extraction-this-repo-file)
   - [2.2 Step 2 – Design / clear‑out / pseudo‑3D work](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#22-step-2--design--clear-out--pseudo-3d-work)
- [3. Design principles](https://github.com/xapids/LLM/blob/main/Nano%20Banana/README.md#3-design-principles)

[3D Modelling View Selection](https://github.com/xapids/LLM/tree/main/Nano%20Banana#3d-modelling-view-selection)

---

## [Image → JSON Extractor](./Image%20-%3E%20JSON%20Extractor.md)

This extractor prompt converts a **single-room floor plan + interior photos** into a compact JSON description that Nano Banana can use as a geometric + semantic scene model.

The extractor itself is LLM-based (e.g. Nano Banana in “text+vision” mode). The resulting JSON is what you feed into downstream render / design prompts.

---

### 1. High-level structureasdasdasd

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

### 1.1 `legend`

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

### 1.2 `proj`

```json
"proj": {
  "name": "Room_Renovation_Clearout"
}
```

Just a human label for the current room / scenario. Useful for logging or multi-room batches, but not strictly required for geometry.

---

### 1.3 `media`

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

### 1.4 `space` – geometry and walls

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

### 1.5 `views` – cameras for each reference / render

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

### 1.6 `elems` – all elements in the room

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

### 1.7 `render` – outputs and keep/remove rules

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
  ],
  "rules": {
    "keep_cat": ["arch", "open", "fix"],
    "rm_cat": ["furn", "appl", "dec", "grp"]
  }
}
```

- `outs`: which views to render.  
  - `id`: render id.  
  - `from`: which `views[*].id` to use as camera.  
  - `lens`: simple lens spec.

- `rules`: simple category-based clear-out logic.
  - `keep_cat`: categories to retain.  
  - `rm_cat`: categories to remove (or replace using each element’s `rm`/`repl`).

You can override `rules` in later design stages (e.g. keep furniture, change only decor).

---

### 2. Usage

### 2.1 Step 1 – Extraction (this repo file)

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

### 2.2 Step 2 – Design / clear-out / pseudo-3D work

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


## 3D Modelling View Selectionasdasd

This spec defines how to add a **canonical set of pseudo-3D “orbit” views** to an existing room JSON (as produced by the *Image → JSON Extractor*), so that you can iteratively design around a specific focus area (e.g. a kitchen front, TV wall, desk zone).

You use the extractor once to build a room model, then you use this view-selection spec to generate 3D-like camera angles around a chosen focus area.

The JSON contract stays the same; you only add or update a small family of views and renders.

---

## 1. Inputs and outputs

### 1.1 Input

- A **single room JSON** that already follows the *Image → JSON Extractor* schema:
  - `space.geom.pts`, `space.geom.H`, `space.geom.walls`
  - Existing `views` and `render`
  - Existing `elems` (optional but typical)

- A **focus request** from the user, in one of these forms:
  - By element id(s), e.g. `["kit_run_1"]`, or `"focus on win_2"`
  - By wall id(s), e.g. `"the run along w2"`
  - By approximate area description, e.g. `"centre of the south wall"` or `"corner where w2 meets w3"`
  - Optionally by explicit `xy` (0–1), e.g. `"focus near [0.75, 0.25]"`

### 1.2 Output

- The **same JSON object**, but with:
  - Additional or updated `views` entries for the pseudo-3D orbit views.
  - Additional or updated `render.outs` entries referencing these new views.

Nothing else is changed unless explicitly requested.

---

## 2. Focus definition

First, define a normalised focus point inside the room.

### 2.1 Compute `focus_xy`

Use the following priority:

1. **Explicit xy**  
   - If the user gives a coordinate `[x, y]` in [0,1]×[0,1], use that directly.

2. **Element id(s)**  
   - If user references one element id:
     - Let `focus_xy = elems[id].pos.xy`.
   - If several elements are referenced:
     - Take the average of their `pos.xy`.

3. **Wall id(s)**  
   - If one wall id is referenced:
     - Project the mid-point of that wall segment in `space.geom.pts` into xy and use it.
   - If a corner between two walls is requested:
     - Use the common vertex between those walls.

4. **Descriptive text only**  
   - Infer the closest wall(s) and approximate position from the text (e.g. “middle of south wall” ⇒ mid-point of that wall).
   - Clamp `focus_xy` to [0,1]×[0,1].

### 2.2 Focus height

- Let `focus_h = min( max(1.0, H * 0.4), 1.4 )` where `H = space.geom.H`.
  - If `H` is not available, default `focus_h = 1.1`.

The height is a conceptual “look-at” level; it is not stored as a field, but the render engine should aim the cameras at this height above the focus point.

---

## 3. Canonical pseudo-3D views

We define up to **four** standard views around the focus point:

- `v_focus_front`
- `v_focus_left`
- `v_focus_right`
- `v_focus_over` (optional, overhead / high angle)

You may drop `v_focus_over` if the user only wants three views.

All views **look at** the same `focus_xy` region; their cameras are placed around it in an arc.

### 3.1 Common values

- Base radius `r` (distance from focus to camera on the floor plane):
  - Let `r = 0.25` in normalised units (25 % of the smaller room dimension).
  - If that would place the camera outside the footprint by a large margin, reduce to `r = 0.18`.

- Camera height:
  - For `v_focus_front` / `v_focus_left` / `v_focus_right`: `cam.h = 1.5` (or clamp to ≤ H−0.2).
  - For `v_focus_over`: `cam.h = min(H * 0.9, 2.4)`; if `H` is missing, use `2.2`.

- All views use the same lens by default:
  - `lens = { "t": "wide", "f": 18, "fov": 90 }`

### 3.2 Angle convention

Define a local frame around `focus_xy`:

- Use the room’s long axis if obvious from `pts` (bounding box).
- Otherwise treat **+x** direction as “front” and **+y** as “left” in the local orbit.

Angles in degrees:

- Front:   `θ_front = 0°`
- Left:    `θ_left  = +50°`
- Right:   `θ_right = −50°`

For a given angle `θ` (in radians) and radius `r`:

```text
cam_x = focus_x - r * cos(θ)
cam_y = focus_y - r * sin(θ)
```

Clamp `(cam_x, cam_y)` back into [0,1]×[0,1] if needed, keeping it close to the footprint boundary.

### 3.3 View definitions

Each view becomes a `views` entry.

Example `v_focus_front`:

```json
{
  "id": "v_focus_front",
  "ref": null,
  "cam": {
    "rel": "free",
    "w1": null,
    "w2": null,
    "xy": [cam_x, cam_y],
    "h": 1.5
  }
}
```

Apply the same pattern for:

- `v_focus_left` (θ_left)
- `v_focus_right` (θ_right)
- `v_focus_over` (if used):
  - Place `cam.xy` almost on top of focus:
    - `cam.xy = [focus_x, focus_y + 0.08]` (slight offset).
  - `cam.h` high (see 3.1).
  - `cam.rel = "free"`.

**Orientation / look-at**  
The JSON does not store an explicit look-at vector; by convention, the render engine interprets any view whose id starts with `"v_focus_"` as:

- “Camera looks at `(focus_xy, focus_h)`.”

---

## 4. Render outputs for pseudo-3D views

Extend `render.outs` with matching entries:

```json
"render": {
  "outs": [
    {
      "id": "r_focus_front",
      "from": "v_focus_front",
      "lens": { "t": "wide", "f": 18, "fov": 90 }
    },
    {
      "id": "r_focus_left",
      "from": "v_focus_left",
      "lens": { "t": "wide", "f": 18, "fov": 90 }
    },
    {
      "id": "r_focus_right",
      "from": "v_focus_right",
      "lens": { "t": "wide", "f": 18, "fov": 90 }
    }
  ]
}
```

- Add `r_focus_over` from `v_focus_over` if you use the overhead view.

Do **not** remove existing `outs`; just append or update the `r_focus_*` entries.

---

## 5. How to use this spec

### 5.1 Prompting pattern (core Pseudo 3D step)

In a Nano Banana (or other LLM) chat, once you already have the room JSON:

1. Paste the **current room JSON**.
2. Add the Pseudo 3D View Selector prompt (the separate `.md` file).
3. Add an instruction like:

   - “Using this spec, add pseudo-3D orbit views around the base cabinets `kit_run_1` and update `views` and `render.outs` accordingly. Return the full updated JSON only.”

4. Optionally refine the focus:

   - “Shift the focus slightly towards wall `w2` so that the fridge and tall cabinet are central in all `v_focus_*` views.”

The LLM:

- Identifies `focus_xy` from your description + existing elements.
- Computes camera positions per this spec.
- Inserts or updates `v_focus_front`, `v_focus_left`, `v_focus_right`, (and optionally `v_focus_over`) and their corresponding `r_focus_*` entries in `render.outs`.

### 5.2 Usage – keep view count small per call

For accuracy and token efficiency:

- Treat all `views` and `render.outs` entries as a **menu**, not as “everything must render now”.
- For a given design iteration, explicitly request ONLY the relevant focus renders, e.g.:

  - “Render `r_focus_front` only.”
  - or “Render `r_focus_front` and `r_focus_left`; do not render any other outs.”

This keeps Nano Banana’s attention and sampling budget focused on the current area instead of scattering it across many views.

### 5.3 Working copy / prune mode (optional)

If you want to minimise tokens for a specific Nano Banana call:

1. Start from the **full master JSON**.
2. Create a **working copy** where:
   - `views` contains ONLY the `v_focus_*` entries needed for this step.
   - `render.outs` contains ONLY the `r_focus_*` entries you plan to render now.
3. Use this pruned JSON in the prompt for that call.

Do not overwrite the master JSON with the pruned version unless you intentionally want to drop old views.

---

## 6. Design principles

- **Reuse the same JSON contract** – no new top-level keys; only more `views` + `render.outs`.
- **Stable naming** – the `v_focus_*` / `r_focus_*` pattern makes it trivial to script or automate.
- **Geometry-aware** – everything is grounded in `space.geom.pts` and the normalised `xy` frame.
- **Token-efficient** – the runtime JSON remains compact; heavy explanation lives in these `.md` files, not inside the JSON.
