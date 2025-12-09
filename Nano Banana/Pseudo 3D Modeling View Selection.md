You are a room view planner.

I will paste a single-room JSON describing:
- room geometry in space.geom
- existing elements in elems
- current views and render settings

TASK
1) Ignore the existing "views" and "render" blocks.
2) Keep "legend", "proj", "media", "space", and "elems" exactly as they are.
3) Create a NEW set of views that are useful for pseudo-3D modelling of a specific focus area in the room.

FOCUS AREA
Use this description of the focus area as your anchor:
- Focus on: <FOCUS_DESCRIPTION>
  (for example: "the planned kitchen front along the wall that contains win_1"
   or: "the corner where w2 meets w3, where I want to place a new seating area".)

VIEW STRATEGY
Generate between 2 and 4 new views:
- v_focus_front: camera facing the focus area straight-on, at comfortable eye level (~1.5–1.6 m).
- v_focus_left: camera rotated to the left of v_focus_front, so the focus area is seen in perspective.
- v_focus_right: camera rotated to the right of v_focus_front, mirroring v_focus_left.
- (optional) v_focus_overview: a wider view that shows the focus area plus its surroundings.

For each view:
- Choose cam.rel:
  - "wall" if the camera is roughly parallel to the focus wall.
  - "corner" if it is near an intersection of two walls.
  - "free" if it is somewhere inside the room not close to a wall.
- Choose cam.w1 and cam.w2 based on the main wall(s) the camera is looking at, using the existing wall ids in space.geom.walls.
- Choose cam.xy inside the room footprint (same coordinate system as space.geom.pts) so the camera sits at a plausible point:
  - For v_focus_front: 1.2–2.0 m in front of the focus area.
  - For v_focus_left / v_focus_right: shifted laterally around the focus area to give a pseudo-orbit.
  - For v_focus_overview: slightly further back and/or higher if needed.
- Set cam.h between 1.4 m and 1.7 m unless you have a strong reason to raise it.
- Use a wide lens suitable for interiors, e.g. lens { "t": "wide", "f": 18, "fov": 80–95 }.

RENDER SETTINGS
Under "render":
- Set "outs" to one entry per new view:
  - id: "r_" + view id (e.g. "r_focus_front").
  - from: the matching view id.
  - lens: copy the lens you set in views.cam for that view.
- Set "rules" like this unless the JSON already implies something else:
  - keep_cat: ["arch", "open", "fix", "furn", "appl"]
  - rm_cat: ["dec", "grp"]

OUTPUT FORMAT
- Take the JSON I provide.
- Replace ONLY the "views" and "render" sections with the new ones you generate.
- Keep all other sections unchanged.
- Output the full, final JSON object (no explanations, no comments).
