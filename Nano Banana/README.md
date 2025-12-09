# Nano Banana – Image → JSON Extractor

This extractor prompt converts a **single-room floor plan + interior photos** into a compact JSON description that Nano Banana can use as a geometric + semantic scene model.

The extractor itself is LLM-based (e.g. Nano Banana in “text+vision” mode). The resulting JSON is what you feed into downstream render / design prompts.

---

## High-level structure

The JSON has six main parts:

- `legend` – mini dictionary of category codes and flags.  
- `proj` – project metadata.  
- `media` – input image filenames.  
- `space` – room geometry (footprint and walls).  
- `views` – camera positions for each reference image (and later, render views).  
- `elems` – all room elements (floor, walls, windows, furniture, appliances, clutter).  
- `render` – requested outputs + simple keep/remove rules by category.

Very compact, but enough for coherent geometry and repeated render passes.

---

## 1. `legend`

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

## 2. `proj`

```json
"proj": {
  "name": "Room_Renovation_Clearout"
}
```

Just a human label for the current room / scenario. Useful for logging or multi-room batches, but not strictly required for geometry.

---

## 3. `media`

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

## 4. `space` – geometry and walls

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

## 5. `views` – cameras for each reference / render

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

## 6. `elems` – all elements in the room

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

## 7. `render` – outputs and keep/remove rules

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

## Usage

### Step 1 – Extraction (this repo file)

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

### Step 2 – Design / clear-out / pseudo-3D work

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

### Design principles

- **Geometry first** – `space.geom` + `xy` + `H` ensure the room shape is explicit and reusable.  
- **Categories for bulk ops** – `cat`, `keep_cat`, `rm_cat` and `rm`/`repl` give you simple clear-out and redesign rules.  
- **Compact but English-like** – field values like `furn`, `appl`, `tile_terracotta_30x30_matt_black_grout` remain readable for humans and LLMs while staying token-efficient.  

This setup lets you use the same JSON both as:

- a compact intermediate representation for Nano Banana; and  
- a stable “contract” between extraction, design, and rendering stages.
