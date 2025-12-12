# README

This folder defines the JSON contracts that Nano Banana uses to work with a **single room at a time**. The JSON is an LLM-facing job description: Nano Banana reads it, and the render engine uses it to create images.

It is intentionally **per-room**, not a whole-house schema.

---

## Pipeline

For each room you want to work with, the intended flow is:

1. Create an "Inventory List":
   * Use the Elements Extractor Prompt.
   * Inputs: floor plan + interior photos.
   * Output: A structured text list of architectural features, joinery, and furniture.
   * This step ensures no items are missed before coding begins.

2. Extract a room model (once per room or per batch of images)
   * Use the Image → JSON Extractor prompt.
   * Inputs:
     * One floor plan image + one or more interior reference images of the *same* room.
     * The "Invetory List" text from Step 1.
   * Output: a JSON object describing:

     * room geometry (`space`),
     * contents (`elems` with `rm` / `repl` flags),
     * camera positions (`views`).

   * **⚠️ Manual Verification (Required):**
     * **Height:** The extractor estimates `space.geom.H`. You **must** manually replace this with the exact physical ceiling height (e.g., `3.0` = 3 meters).
     * **Geometry:** Check `space.geom.pts`. If the room is L-shaped or complex, ensure the polygon shape matches reality.

3. (Optional) Add canonical 3D-like views

   * Use the prompt in View Creator.
   * Inputs: the JSON from step 1 + a **focus area** and a `focus_key`.
   * Output: the same JSON, but with extra orbit-style camera views added to `views`.

4. Before each Nano Banana call, prune views to control outputs and tokens

   * Decide which camera ids you actually want images for on this call.
   * Delete any `views[*]` entries you do not need.
   * The JSON you send to Nano Banana should typically contain:

     * the full `space` and `elems`, but
     * only a small subset of `views` corresponding to shots you want now.

This README is **explanatory only**. For exact field-by-field rules, see the files:

* Elements Extractor Prompt - inventory list extractor
* Image → JSON Extractor Prompt – canonical single-room extractor.
* View Creator Prompt – canonical orbit-view generator around a focus area.

<br><br><br><br><br>

## Elements Extractor Prompt

This prompt acts as the project's **"Surveyor"**. It converts visual data into a rigorous, text-based inventory *before* any JSON coding happens.

By separating the "seeing" (inventory) from the "coding" (JSON formatting), we significantly reduce hallucinations and missing items in the final model.

### What it does

* **Zero Estimation:** It forces the model to count every item (e.g., "3x windows" instead of "some windows").
* **Decomposition:** It breaks complex units into parts (e.g., a Kitchen is broken into "Base unit", "Wall unit", "Sink", "Tap").
* **Architectural Scan:** It specifically targets structural features often missed, like beams, cornices, and risers.

### Inputs

* The prompt text from Elements Extractor Prompt
* The **same** image set you will use for the JSON Extractor (1 floor plan + interior reference photos).

### Output Structure

The output is a Markdown-formatted "Inventory List" divided into three strict categories. You will copy-paste this output into the *Image → JSON Extractor* in the next step.

A. Architecture & Openings (The Shell)
* **Windows & Doors:** Exact counts of every opening.
* **Structure:** Beams, columns, steps, and level changes.
* **MEP/Fixed:** Radiators, AC units, wall sconces, ceiling fans.

B. Fixed Joinery & Kitchen (The Built-ins)
* **Kitchen:** Breakdown of drawer fronts, cabinet doors, and appliances.
* **Fixtures:** Sinks and taps listed separately from counters.
* **Storage:** Built-in wardrobes and cupboards.

C. Loose Furniture & Decor
* **Seating:** Every chair, stool, and bench counted.
* **Surfaces:** Tables, desks, and freestanding shelves.
* **Groups:** Distinct clusters of clutter (e.g., "Laundry rack group").

<br><br><br><br><br>

## Image → JSON Extractor Prompt

This extractor prompt converts a **single-room floor plan + interior photos** into a compact JSON description that Nano Banana can use as a geometric + semantic scene model.

The extractor itself is LLM-based (e.g. Nano Banana in “text+vision” mode). The resulting JSON is what you feed into downstream render / design prompts.

---

### 1. High-level structure

The JSON has six main parts:

1.1 `legend` – mini dictionary of category codes and flags.  
1.2 `proj` – project metadata.  
1.3 `media` – input image filenames.  
1.4 `space` – room geometry (footprint and walls).  
1.5 `views` – camera positions for each reference image / render.  
1.6 `elems` – all room elements (floor, walls, windows, furniture, appliances, clutter).  

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
    "bounds": [4.5, 3.2],
    "pts": [[0,0],[1,0],[1,0.71],[0.4,0.71],[0,1]],
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

- `bounds`: [width, length] in **physical meters** (read from floor plan dimensions).
- `pts`: polygon vertices of the room footprint, **normalised** to [0,1] via Uniform Scaling (aspect ratio preserved).
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

- `id`: view id.  
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

### 2. Usage

#### 2.1 Step 1 – Extraction (this repo file)

1. In a Nano Banana (or other LLM) chat:
   - Paste the full text from `Image -> JSON Extractor Prompt.md`.  
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

   * The room JSON from step 1.
   * Any subset of the original reference images you want to stay aligned with.

2. Ask for changes in terms of this model, for example:

   * “Set `rm = true` on the existing sofa and armchair elements, extend the floor and skirting behind them, and update the JSON.”
   * “Add a kitchen run along wall `w2` under window `win_1`, about 2 m long, with base cabinets and a worktop.”
   * “Create three new orbit views around the kitchen area with ids `v_kit1_front`, `v_kit1_left`, `v_kit1_right`.”

3. The design or view-creation prompt updates:

   * `elems` (adding/removing/modifying elements and their `rm` / `repl` flags).
   * `views` (adding or updating camera positions).

4. Before each render call, keep only the `views` you want images for on that call and delete the rest, then send the pruned JSON to Nano Banana together with the relevant images.

---

### 3. Design principles

- **Geometry first** – `space.geom` + `xy` + `H` ensure the room shape is explicit and reusable.  
- **Per-Element Flags** – `cat` and `rm` / `repl` give you simple clear-out and redesign behaviour per element.
- **Compact but English-like** – field values like `furn`, `appl`, `tile_terracotta_30x30_matt_black_grout` remain readable for humans and LLMs while staying token-efficient.  

This setup lets you use the same JSON both as:

- a compact intermediate representation for Nano Banana; and  
- a stable “contract” between extraction, design, and rendering stages.

<br><br><br><br><br>
  
## View Creator Prompt

This spec takes an existing **single-room** JSON (from the Image → JSON Extractor) and adds a small family of pseudo-3D “orbit” views around a chosen focus area.

### What it does

* You choose a focus area (for example “front of the kitchen run”, “TV wall”, or a specific element / wall) and a short `focus_key` (e.g. `kit1`, `tv`, `desk`).

* The LLM computes a `focus_xy` point inside the room and creates up to four standard views:

  * `v_<focus_key>_front`
  * `v_<focus_key>_left`
  * `v_<focus_key>_right`
  * `v_<focus_key>_over` (optional overhead / high angle)

* All of these are added to the `views` array. They are **just more cameras**; which of them you render on a given call is up to you.

The spec never changes walls or elements. It only adds or updates `views` whose ids start with `v_<focus_key>_`.

### Inputs

To use this spec you need:

* The room JSON produced by the Image → JSON Extractor (single-room only).
* A text instruction that describes:

  * The `focus_key` you want to use.
  * How to locate the focus:

    * element id(s), or
    * wall id(s), or
    * an explicit normalised coordinate `[x, y]` in [0,1]×[0,1], or
    * a short description like “middle of the south wall” or “centre of the TV wall”.

### Outputs and usage pattern

The result is the same JSON object, but with additional `views` entries.

Typical pattern:

1. Paste the current room JSON.

2. Paste the full text from [3D Modelling View Creator](./3D%20Modeling%20View%20Creator.md).

3. Add an instruction, for example:

   * “Using `focus_key = "kit1"`, add orbit views around the main kitchen wall and return the full updated JSON only.”

4. From the updated JSON, keep only the view ids you want to render on this call (for example: `["v_kit1_front", "v_kit1_left"]`) and delete the rest of `views` before sending to Nano Banana.

