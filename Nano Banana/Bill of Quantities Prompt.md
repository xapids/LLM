#### Task: Analyze floor plan and reference photos to extract:
1. **Geometry:** List of walls including measurements; room height
2. **Inventory:** Rigorous "Bill of Quantities" of ALL interior elements

#### CRITICAL RULES:
1. **Geometry:** Scan floor plan for dimension text (e.g., "1.9m"). List ONLY walls with explicit text. Ignore unmeasured lines.
2. **Inventory:** : Zero estimation. Count exactly (e.g. "3x Chairs", not "some chairs"). Decompose assemblies (Table + Chairs = separate items)
3. **Architectural Scan:** Look specifically for structural features: beams, columns, arches, steps, dado rails, cornices, and skylights.

#### OUTPUT FORMAT:
- Return ONLY valid, concise JSON
- Render array objects on single lines
- Use this exact schema:

{
"bill_of_quantities": {
  "legend": {
      "arch": "architecture / structural surfaces & features (beams, columns, exposed trusses, steps, level changes, distinctive floor sections (if multiple types), ceiling features)",
      "open": "openings (window, door, passage/arch)",
      "fix": "fixed elements & joinery",
      "appl": "appliances",
      "furn": "furniture",
      "dec": "decor",
      "grp": "grouped clutter volumes",
      "wall": "walls",
      "L": "length_in_meters",
      "H": "height_in_meters"
    },
  "space": {
    "wall": [
      { "id": string, "L": number },
      { "id": string, "L": number }
    ],
    "H": number
  },
  "elems": [
    { "cat": "arch" | "open" | "fix" | "furn" | "appl" | "dec" | "grp", "qty": number, "d": string },
    { "cat": "arch" | "open" | "fix" | "furn" | "appl" | "dec" | "grp", "qty": number, "d": string }
  ]
}
}
