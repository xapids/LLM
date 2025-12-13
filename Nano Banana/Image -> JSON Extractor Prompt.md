TASK
You are a vision + geometry extractor.

PROCESS OVERVIEW (Strict Order):
1. **Inventory Reconciliation:**
   - You will be provided with a **"Inventory List"** text.
   - **MAPPING RULE:** You must create a JSON entry in "elems" for EVERY item listed in the Inventory List.
   - **Consistency Check:** If the Inventory lists "3x Casement windows", your JSON must have `win_1`, `win_2`, `win_3`. Do not skip items.
   - **Scan:** If you see visual items in the photos that are missing from the Inventory List, ADD them to your JSON inventory.

2. **Geometry Check:**
   - Look at the floor plan. Is it a simple rectangle, or does it have notches/L-shapes?
   - Ensure your geometry points match the actual corners (e.g., 6 corners for an L-shape).

3. **JSON Generation (The "Coding" Phase):**
   - Now, map every item from your Visual Inventory into the "elems" array of the schema below.
   - Calculate their [0,1] coordinates.
   - Output ONLY the final valid JSON.

{
  "task": {
    "intent": "Architecturally_prepare_room_for_future_design_renders",
    "scope": "render_all_views",
    "goal": "architectural_preparation",
    "description": "remove elements with 'rm': true; replace according to 'rm'"
  },
  "legend": {
    "cat": {
      "arch": "architecture surfaces (floor, wall, ceiling)",
      "open": "openings (window, door, arch)",
      "fix": "fixed elements (ceiling fan, AC, built-in light, radiator)",
      "furn": "furniture (tables, chairs, benches, sofa, shelves)",
      "appl": "appliances",
      "dec": "decor (art, posters, decorative lamps, plants)",
      "grp": "group of small items / clutter"
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
  },

  "proj": {
    "name": string
  },

  "media": {
    "floor": string,
    "refs": [
      { "id": string, "file": string }
    ]
  },

"space": {
    "geom": {
      "bounds": [number, number],
      "pts": [
        [number, number]
      ],
      "H": number,
      "orientation": "+x" | "-x" | "+y" | "-y",
      "walls": [
        {
          "id": string,
          "seq": int,
          "p0": int,
          "p1": int,
          "label": string
        }
      ]
    }
  },

  "views": [
    {
      "id": string,
      "ref": string | null,
      "cam": {
        "rel": "corner" | "wall" | "free",
        "w1": string | null,
        "w2": string | null,
        "xy": [number, number],
        "h": number
      }
    }
  ],

  "elems": [
    {
      "id": string,
      "cat": "arch" | "open" | "fix" | "furn" | "appl" | "dec" | "grp",
      "type": string,
      "pos": {
        "rel": "on" | "between" | "floor" | "ceil",
        "w1": string | null,
        "w2": string | null,
        "xy": [number, number],
        "h": number | null,
        "sz": [number | null, number | null, number | null]
      },
      "views": [
        {
          "v": string,
          "bb": [number, number, number, number],
          "vis": "f" | "p" | "o"
        }
      ],
      "d": string,
      "rm": boolean,
      "repl": string | null
    }
  ]
}

--------------------------------------------------
GEOMETRY & WALL ORDERING
--------------------------------------------------

1) Treat the floor plan as 2D with x→right, y→up (or down, but be consistent).

2) Find the closed polygon of the room footprint (the walkable interior boundary).
   **CRITICAL GEOMETRY RULE:**
   - Look strictly at the floor plan. If the room is L-shaped, T-shaped, or has a notch/alcove (e.g. for stairs), you MUST capture all corners.
   - Do NOT simplify the room into a rectangle.
   - If there are 6 corners in the plan, your "pts" array must have 6 coordinates.

3) Order the footprint vertices:

   a. Let C0 be the vertex with the smallest xr; if several share this, pick among them the one with smallest yr.  
   b. Starting at C0, walk around the perimeter CLOCKWISE, visiting each vertex once until you return to C0.  
   c. Call these ordered vertices C0, C1, …, C(n-1).

4) Determine Physical Scale (CRITICAL):

   - Read the text labels on the floor plan image to find the physical Width and Length of the room.
   - If dimensions are in mm (e.g., 4500), convert to meters (4.5).
   - Store these in "space.geom.bounds": [width_meters, length_meters].
     (e.g., for a 4m x 3m room, use [4.0, 3.0]).

5) Normalise using UNIFORM SCALING (Preserve Aspect Ratio):

   - Compute raw bounding box:
       raw_w = xmax - xmin
       raw_h = ymax - ymin
   - Determine the scaling factor (max dimension):
       S = max(raw_w, raw_h)
   - For each Vertex Ci = (xr, yr), compute:
       x = (xr - xmin) / S
       y = (yr - ymin) / S
   - Store the ordered, normalised vertices in "space.geom.pts".
     *Note: The longer dimension will span [0, 1]. The shorter dimension will be < 1.0.*

6) Define walls:

   - For each edge Ci → C(i+1) (with C(n) wrapping back to C0), create a wall.
   - Assign wall ids in perimeter order:
       w1 for edge C0→C1, w2 for C1→C2, …  
   - For each wall, output:
       {
         "id": "w1",
         "seq": 1,
         "p0": 0,
         "p1": 1,
         "label": "short_descriptive_name"
       }

   Where:
   - p0, p1 are integer indices into "pts".  
   - label is a short English identifier (e.g. "window_wall_long", "entrance_wall").

7) Set "space.geom.H" to the approximate room height (e.g. 2.6).

8) Define "space.geom.orientation":
   - Determine the logical "front" of the room (usually facing the main window or the main activity wall).
   - Output one string: "+x", "-x", "+y", or "-y".
   - This tells downstream tools which direction in the normalised plan is "forward".

--------------------------------------------------
FLOOR COORDINATES (xy)
--------------------------------------------------

All floor positions in this JSON (cameras and elements) must be expressed in the same normalised coordinate system as "space.geom.pts":

- xy[0] is x in [0,1] across the bounding box of the footprint.  
- xy[1] is y in [0,1] across the bounding box of the footprint.

Place cameras and elements so that:

- xy lies inside or very close to the polygon defined by "pts".  
- The relative distances and proportions roughly match what you infer from the floor plan.

--------------------------------------------------
VIEWS
--------------------------------------------------

The "views" array defines the cameras for this room. Each entry is a camera definition:

* "id": string – unique view id, e.g. "v_ref_1", "v_kit1_front".
- "ref": string | null – id of the source reference image (for views tied to an input photo), or null for synthetic views (e.g. orbit cameras).
* "cam":

  * "rel": "corner" | "wall" | "free" – how the camera position is anchored.
  * "w1", "w2": wall ids or null (for "free" cameras).
  * "xy": [x, y] in [0,1]×[0,1] – camera position in the floor-plan coordinate system.
  * "h": number – camera height in metres above floor.

For each reference image you must:

1. Create a "views" entry.

2. Set cam.rel:

   * "corner" if the camera is clearly near the intersection of two walls.
   * "wall"   if the camera is clearly along one main wall.
   * "free"   otherwise (somewhere inside the room).

3. Set cam.w1 and cam.w2:

   * cam.w1: id of the main wall the camera is associated with, or null.
   * cam.w2: second wall for a corner camera (the other wall meeting at the corner), or null.

4. Set cam.xy:

   * Approximate floor position of the camera centre in the normalised footprint coordinates.
   * Use the floor plan plus your understanding of the views.

5. Set cam.h:

   * Camera height above floor (m).
   * Usually about 1.4–1.7 for eye-level interior photos; default ~1.6 if uncertain.

--------------------------------------------------
ELEMENTS
--------------------------------------------------

For each visible element, create an "elems" entry.
**GRANULARITY RULE:** Do not group distinct objects into single "summary" elements. You must decompose:
- **Furniture Groups:** A dining table and its chairs are separate elements.
- **Embedded Fixtures:** A counter is one element; the sink and tap embedded in it are separate elements.
- **Surface Items:** Large functional objects (e.g., a drying rack, a large basket, a standing lamp) are separate elements, not just "decor" or "clutter".
- **Structural Features:** Dominant visual features (like beams, large overhead lines, or railings) must be captured as fixed elements.

If an object is large enough to have a distinct material or function, list it separately.

Do NOT ignore items just because they look like "clutter" if they are large or distinct.

1) id:

   - Stable machine id, e.g. "win_1", "door_1", "tbl_1", "appl_wash_1".

2) cat (category):

   - "arch" = structural surfaces (floor, ceiling, structural wall, beam)  
   - "open" = window, door, arch opening, niche, pass-through  
   - "fix"  = fixed non-structural element (ceiling fan, AC, radiator, built-in light)  
   - "furn" = furniture (tables, chairs, benches, sofa, loose shelves)  
   - "appl" = appliances  
   - "dec"  = decor (artwork, posters, decorative lamps, plants)  
   - "grp"  = grouped small items / clutter treated as one element

3) type:

   - More specific subtype within cat:
     - Example: "window", "door", "base_cabinets", "dining_table", "sofa", "washing_machine", "wall_art", "small_appliance_group".

4) pos (position):

   - rel:
     - "on"      = attached to one wall (sconce, radiator, artwork).  
     - "between" = at/between two walls or in a corner.  
     - "floor"   = free-standing on the floor (table, freestanding shelf).  
     - "ceil"    = mainly attached to the ceiling (ceiling fan, pendant).

   - w1, w2:
     - Use wall ids from "space.geom.walls".  
     - Consistent with rel:
       - "on": w1 is the wall it is on; w2 = null.  
       - "between": w1 and w2 are the two walls it is between.  
       - "floor"/"ceil": w1/w2 may be null or indicate the nearest wall(s).

   - xy:
     - Element floor position in the same normalised footprint coordinates as "pts".  
     - For on-wall elements, xy should lie on or close to the associated wall segment.  
     - For floor elements, xy is the plan position of the element’s centre.

- h:
     - Vertical position of the element's **anchor point** (metres from floor).
     - **Floor items:** h = 0 (Anchor is bottom).
     - **Wall items:** h = height of the **centre** of the object (e.g., for a sconce or painting).
     - **Ceiling items:** h = approximate height of the fixture's attachment point (usually close to Room H).

   - sz:
     - Approximate [width, height, depth] in metres, if you can estimate.  
     - If you cannot estimate, use [null, null, null].

5) views (appearances in reference images):

   If you can locate the element in a reference image:

   - Add an entry:
     {
       "v": "v1",
       "bb": [xmin, xmax, ymin, ymax],
       "vis": "f" | "p" | "o"
     }

   Where:
   - v: view id ("v1", "v2", …).  
   - bb: [xmin, xmax, ymin, ymax] in normalised frame coordinates of that image:
     - x, y in [0,1], with x from left→right, y from bottom→top.  
   - vis:
     - "f" = fully visible  
     - "p" = partially visible / cropped by frame  
     - "o" = mostly occluded

6) d (description):

   - Short compressed English tokens, not long sentences.  
   - Use underscores between tokens:
     - "tile_terracotta_30x30_matt_black_grout"  
     - "wall_plaster_warm_cream_matt"  
     - "window_casement_black_aluminium_clear_glass"  
     - "table_rect_black_metal_top"

7) rm and repl (clear-out semantics):

   - rm:
     - true  = element should be removed in a clear-out / redesign render.  
     - false = element should remain.

   - repl:
     - If rm=true: short instruction for what to show instead, e.g.:
       - "plain_wall"  
       - "extend_floor_and_walls"  
       - "empty_floor"  
     - If rm=false: null.

--------------------------------------------------
OUTPUT FORMAT
--------------------------------------------------

**Scenario A: You have questions**
- Output a bulleted list of questions regarding the geometry or elements you are unsure about.
- Wait for the user to reply.

**Scenario B: You are confident (or have received answers)**
- Output ONLY the final JSON object.
- No explanations or markdown chatter before/after the JSON.
- Ensure JSON is valid (double quotes, correct commas).
