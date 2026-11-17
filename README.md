# PDF to CAD

Turns vector PDF drawings (from Revit, AutoCAD, and most CAD exports) into DWG, DXF, and 3D Revit/IFC models using the layers and objects you define. Everything runs on fixed geometric rules—no AI, no cloud services, no guesswork. Built and tested against the Barnes Revitalization HVAC set (MT Studio, Revit).

**Building workflow:** Combine architectural, structural, and MEP sets, then label wall assemblies and ceiling heights directly on the drawings. See [the building workflow and YMCA validation](docs/building-workflow.md) for the deterministic rules, review steps, catalog formats, native Revit support, and current limits.

## Getting Started & Controls

Start the app from the desktop shortcut **PDF to CAD** (or run `PDF to CAD.cmd` in this folder). It spins up a small local server on port `5260` and opens the UI in an Edge app window. Close the console window to stop the server.

- **Selection:** Works like Revit. Click to replace, `Ctrl` to add, `Shift` to remove. Drag left-to-right for a window selection; drag right-to-left for a crossing selection.
- **Navigation:** Mouse wheel zooms, middle mouse pans, `F` zooms to fit the sheet.
- **Shortcuts:** `Ctrl+O` opens a PDF, `Ctrl+S` saves the standard, `Ctrl+E` jumps to Export, `Ctrl+Z` undoes.
- **Exclude (`Select` + `Delete`):** Select anything and press `Delete`. It stays visible on screen (red, dashed) so you know it's there, but gets left out of all exports, counts, and the Revit model. Press `Delete` on excluded items to bring them back, or click **Bring all back** in the status bar. Exclusions are saved per PDF in `exclusions/` (never beside the PDF) by line style and position so they survive reloads. If you change cleanup settings, they re-match by position and the app lets you know.

---

## How It Works (The Four Tabs)

The right panel follows the job in four tabs:

### 1. Import
Open PDFs and recent files. Every sheet is listed on the left by sheet number and title.
- **Scale:** Read automatically from the sheet title (`1/8" = 1'-0"` puts the plan at real size in inches). You can override it per sheet if the title is missing.
- **Cleanup switches:** Join pieces, rebuild dashes, rebuild arcs, drop duplicates, and honor clip paths. These save with the standard. A report shows what came in.

### 2. Details
Five sub-tabs control how the PDF maps to CAD:
- **Layers:** The layers going into the DWG (color, lineweight, show/hide).
- **Lines:** Every line style on the sheet (PDF color and thickness, solid or dashed, fills). Pick which layer each style goes to. Hover a row to light those lines up on the sheet. Applies to all sheets.
- **Text:** Rules sort text by pattern: `IDU-*`, `A#-*` (`*` matches anything, `#` matches a digit, `;` separates patterns, or use `/regex/`). First match wins.
- **Areas:** Boxes drawn around title blocks, key plans, or schedules whose contents go to a single layer or get left out.
- **Recognize:** Finds walls, windows, doors, and plumbing fixtures in the architectural background (tick its line styles) and puts them on `A-WALL`, `A-GLAZ`, `A-DOOR`, and `P-FIXT`.

**Rule priority for each line:**
1. Object
2. Style set to *Leave out*
3. Area
4. Recognized architecture
5. Line style rule
6. Auto layer

> **Note on Areas:** Areas are drawn in PDF points, so an area belongs to the sheet size it was drawn on (`sheetSize` on the area, or on the standard: the MT Studio standard is 36x24 in, `[2592, 1728]`). On a sheet more than 2% bigger or smaller, it is ignored. An area tied to a specific page (`pages: [5]`) only applies when that page matches the expected size.

### 3. Fabrication
- **Objects:** Window-select one symbol (a diffuser, thermostat, or cassette) and click **Make object**. Every copy on the sheet—even rotated ones—becomes the same CAD block, pulling nearby text (`IDU-*`) as its `TAG` attribute.
- **Count table:** Lists objects and recognized doors, windows, and fixtures for the current sheet or all sheets, and exports to CSV.
- **Revit model:** Lists what the trades turn into in 3D (ducts, flex, pipes, air devices, equipment, objects) with counts, levels, and default heights a 2D plan can't show. Click **Show on the sheet** to draw every 3D piece over the 2D plan.

### 4. Export
Export one sheet, all sheets, or a pick list. Choose scale, area, output formats (DWG, DXF, and/or `.rvt` Revit model), lineweights, and output folder.

**What comes out in CAD:**
- **Format:** DXF R2000 / DWG 2018, model space in inches at the chosen scale (`$INSUNITS` = inches, `$LTSCALE` = drawing scale, dash patterns in sheet units).
- **Layers:** ACI color and lineweight; entities set to `ByLayer`.
- **Text:** Mapped to the matching Windows font (Arial, Arial Bold, Narrow, Times, etc.), cap height, and a width factor fitted from real TrueType widths so text stays the exact same length as in the PDF.
- **Blocks:** Geometry on layer `0` inside, inserted on the object's layer with a `TAG` attribute.
- **Fills:** White masking fills are left out by default (otherwise they plot as black boxes in CAD).
- **Barnes starter objects:** 40 lay-in diffusers + 8 thermostats on H1.1; 29 cassettes (each with its `IDU` tag) + 29 thermostats on H1.2. Future (dashed) units stay as plain lines.

---

## Cleanup & Recognition Rules

### What the Cleanup Does (`lib/cleanup.mjs`)
Revit PDFs write every stroke piece as its own path (H1.1 has 113,898). Before anything is shown:
- Touching pieces of the same style are joined back into polylines in both directions.
- Straight-through vertices are removed, so a wall made of 12 pieces comes back as one line.
- 16-sided "circles" become true `CIRCLE` entities, and runs of chords on one circle become arcs (door swings, elbows).
- Revit's chopped dashes are rebuilt as a single line with a matching linetype (`PDF_DASH_n`).
- Duplicates and zero-length specks are dropped; clip paths are honored.

*H1.1 goes from 113,898 PDF paths down to ~16,500 CAD entities.*

### How Recognition Decides (`lib/recognize.mjs`)
- **Walls:** Two long parallel lines 3" to 18" apart (set on the tab), nothing at wall spacing between them, much longer than thick, not part of an evenly spaced series (stairs), and no sinks or boxes inside the band (counters).
- **Windows:** An opening in a wall (a 1 ft to 30 ft gap in wall coverage) where at least two rows of lines inside the wall band cover most of the opening.
- **Doors:** A 60° to 100° swing of 18" to 48" radius with its leaf line or panel from the hinge.
- **Plumbing fixtures:** Grown from a bowl or basin (a closed round shape 6" to 26") with the parts around it, at most 36"×36", standing within 4" of a recognized wall.

---

## Sets Drawn Differently: Learning Styles from the Set (`lib/learn.mjs`)

A standard is built on one office's sheets. Other offices draw their trades with different pens: black at another weight, a color per system (supply blue, return orange, exhaust magenta), or grey fills inside ducts. When a PDF opens, the app checks the standard against the set in the background (`lib/learn.mjs`, no AI):

- **Fit:** On every plan sheet with a scale (ignoring NTS, schedules, details, sections), each duct size tag (`18x12`, `16"x10" SA`, `10/10`, `8"ø`) looks for two parallel lines of one style at its width. The fit is the share of those tags whose pair sits on the standard's duct layer. At **20% or more**, the standard explains the set and nothing changes (Barnes is at 99%).
- **Another saved standard:** Under 20%, if another saved standard explains the set, the best fit is used instead with a note. That way a standard saved from another office's set doesn't stay stuck on Barnes.
- **Learning:** Under 20% across all saved standards, the set's own tags vote on which pen draws what across the whole set at once:
  - *Duct tags* vote for the pen whose line pairs match the tag's width (and not at decoy widths, which filters out hatches and grids).
  - *Air device & equipment marks* (including from schedules, and stacked tags like `TB` over `1-1` = `TB-1-1`) vote for the pen of adjacent shapes, tested against room names and notes. When a set colors by system, a device or unit pen learned in one color carries over to the other duct colors at half confidence.
  - *Pipe tags* vote for the long lines their leaders point at (never the leader's own line).
  - *Walls* are detected from grey line pairs 4" to 12" apart.
  - *Enlarged plans:* Pens print heavier on sheets drawn at a bigger scale, so sheets of each scale are also read on their own—a duct pen used only on 1/4" enlarged plans is learned from those sheets.
- **In use:** The learned standard takes over for that PDF and shows a banner: *"This set is drawn differently from MT Studio Revit HVAC. PDF to CAD learned its styles..."*. Click **Save as a standard** to keep it for that office's next set, or **Use MT Studio Revit HVAC instead** to switch back (remembered for that PDF). The **Lines** tab lists the learned rules, confidence, and reasoning. Assemble uses the learned standard too.
- **Areas:** Because the standard's areas were drawn on another office's layout, the learned standard finds the set's own title block and border instead: the title block is the strip right of the frame line (a vertical line nearly as tall as the sheet, just left of the sheet number), and the border is the margin out to the innermost frame line. That prevents a sheet border drawn in the duct pen from becoming a duct, or a title block grid from becoming pipes. When no frame is found, only standard areas matching the sheet size are kept (page-specific areas are dropped).
- **Storage:** Cached per PDF by content hash in `learned/` in this folder (never beside the PDF, never in git).
- **SHX stroked text:** A set whose text is drawn as strokes (AutoCAD SHX: no readable scale or tags) shows *"This set has no readable text; set the scale by hand"* in the banner, Import tab, and status bar.

---

## 3D Trades & Revit Models

### How the Revit Model Reads the Trades (`lib/trades.mjs`)
- **Ducts:** Two parallel lines on the duct layer at duct width (5.5" to 60", smaller only when confirmed by a tag) with nothing between them, proved by a line across them about as wide as the duct (an end, joint, or transition). The side of another duct or a long pipe crossing the gap does not count, so the space between two parallel ducts is never read as a duct. Size comes from the tag its leader lands on (`16x14` = width × depth, `10"ø` round; also `16"x10" SA`, `10/10`, `8"Ø/150`, `ø8`, `8" dia`, `Size: 16"x7"`, `24x12 FO`; the system word sets the duct's system), or adjacent text when the leader lands on no duct of its width, carried along the run to untagged pieces of the same width. Otherwise it falls back to round at a nearby air device neck size, or the measured width with an assumed depth. Elbows come from two concentric arcs, built as short pieces.
- **Flex:** A centerline or arc on the air device layer crossed by 3+ ticks (diameter = tick length). Repeated polygon rings are also read, including rings drawn in an equipment or duct pen: at least 5 similar rings must form a regular, unbranched chain near a duct end (ring centers form the path; transverse width sets the diameter). Ambiguous, short, or isolated patterns stay as 2D lines. The Fabrication table counts flex paths rather than the small solids used to preview curved paths.
- **Pipes:** Single lines on the duct layer at least 3 ft long that aren't a duct side. A tag like `0.75" D` sizes the line its leader lands on and carries along connected lines. When a learned standard is active, **Require pipe evidence** turns on in Fabrication: a pipe then needs a tag, a connection to a tagged pipe, or a pen learned from pipe tags. Unconfirmed candidates are counted in the panel and left out of the 3D model (their 2D CAD lines remain). Turn the option off to review untagged piping.
- **Air devices:** Rectangles or circles 10" to 48" on the air device layer, or a face drawn as open lines (hexagon tag bubbles are skipped), typed by the nearest tag (`A2-8"ø`, `B2-24x24`, `CD-2` over `150` over `8"`, `A 225`, `12X12-EG`), with CFM in Comments, placed at the ceiling.
- **Equipment:** Line clusters on the equipment layer at least 12" across, named by tag (`IDU-04.01`, `TB-1-1`, `TB` over `1-1`). Flex drawn as a chain of rings or a zigzag in the unit pen is ignored; an outline holding multiple lines of text is skipped as a schedule or legend. Heights by tag (`EH-*` at the floor, `EF-*` at the ceiling, `BCU-*`, outdoor units) come from the standard's `trades`.
- **Objects:** Every object instance, sized at `model.size_ft` from the standard (or the symbol's drawn size).

### Mode 1: Generic Solids
Every piece becomes a Generic Model solid at its real size (PlanToRevit's worker builds three families in the run: `PDF2CAD Box`, `Round Horizontal`, `Round Vertical`), with `Mark` and `Comments` showing where its size came from. The sheet goes behind it as an underlay in the plan view. Outputs one `.rvt` per sheet, built in a separate minimized Revit 2025 session (~40s per sheet); work files go in `Revit work\<sheet>\` next to the `.rvt`.
- *Barnes H1.1:* 309 pieces built in Revit and read back at their sizes, nothing skipped (142 duct runs, 13 elbows, 48 flex, 27 pipes, 14 air devices, 16 units, 49 objects).

### Mode 2: Real Revit MEP Model (`Build as: Real Revit ducts`)
The default since 0.3.0. Uses the same reading of the sheet, then `lib/network.mjs` turns it into a connected model for the PlanToRevit worker's `mep` list. Everything is fixed rules (no AI)—the same sheet and settings always produce the same model:
- **Network:** Collinear pieces of one size merge into one duct (biggest first, never through another duct, so two runouts on either side of a trunk stay two taps). Two sizes end-to-end meet in a transition; two runs whose centerlines cross near both ends meet in an elbow (including across a drawn elbow arc); a run ending on the side of another becomes a tap on that face; a run ending at a grille or unit plugs into it. Short pieces drawn inside a grille or unit outline (necks, plenums) are left out.
- **Flex:** Real Revit flex from the runout end into the diffuser's neck connector. A diffuser takes one connection: flex first, then hard duct.
- **Drops:** A duct plugging into a ceiling grille runs to above its center, turns down in an elbow, and drops onto the grille's connector.
- **Systems:** Each duct takes Supply, Return, or Exhaust from the nearest terminal it feeds (from the schedule description, else the tag letter); ducts at an `EF` fan are Exhaust.
- **Heights (coordination rules):** Diffusers and grilles sit at the ceiling; every duct is bottom-aligned at the duct bottom elevation. Where two unconnected ducts cross, the smaller hops over the bigger with 45° offsets if it has room under the structure. Pipes run above the ducts they cross (below when there is no room), and the smaller of two crossing pipes goes over. Ceiling, duct bottom, and structure elevations are settings since a 2D plan doesn't show them.
- **Revit checks itself:** The worker reads every built element back (size, elevations, connections) and runs a clash check on everything it built (connected pairs excluded). The server feeds the clashes and real measured connector heights back into the rules (lifting clashing pipes by overlap + gap, turning clashing ducts into a crossover, and aiming drops at measured connectors) and builds once more. Both passes are reported: what was built, what was skipped and why, and any clashes left.

---

## Schedules & Library Families (`lib/schedules.mjs`, `lib/families.mjs`)

HVAC schedules on a set are read into rows, and each row is matched to a family in the FamilyHub library so Revit models build with real `.rfa` families.

- **Reading schedules (`lib/schedules.mjs`):** `findSchedules(model)` finds tables from their ruling lines. The title is the top band with no column lines, the header runs down to the first full-width line, and multi-level headers are joined with `" / "` (e.g., `"INDOOR UNIT / BASIS OF DESIGN / MODEL"`). Each row is keyed by its `PLAN MARK` (or `MARK`, `TAG`) column. Wrapped cell text is joined, empty cells stay empty, side-by-side tables are split, a totals line goes to `footer`, and notes under a table go to `notes` (numbered notes also go into `noteItems`). Where a table has no row lines, rows split at the lines holding a mark. Text crossing a column line is cut at the line, and double-drawn text is read once.
- **Mapping tags (`scheduleIndex`):** Maps a plan tag to its schedule row with normalized `fields` (`manufacturer`, `model`, `description`, `mounting`, `cfm`, `neck_in`, `face`, `length_in`, `width_in`, `height_in`, `depth_in`, `weight_lb`, `kw`, `volts`, `phase`, `notes`, `cooling_mbh`, `tons`). Dimensions like `34"`, `181"`, and `2'-6"` convert to inches. Handles exact marks (`EH-1`, `BCU-4`, `IDU-04.01`), type marks with a size suffix (`A1-10"ø` = mark `A1` with a 10" neck; `B2-24x24`), family marks before a `.NN` unit number (`IDU-04.01` -> `IDU-04`), zero-padded marks (`EF-01` = `EF-1`), and schedule ranges (`EF-1 THRU EF-4`, `IDU-01.01 THRU 01.12`, `EF-1, 2`).
- **Matching families (`lib/families.mjs`):** `resolveFamily(fields, loadCatalog(), { root })` scores every family in `O:/Revit Development/FamilyHub/data/catalog.json` (read-only) on maker (`MITSUBISHI` = Mitsubishi Electric, allowing a 1-letter typo with less weight), model (`TMS` matches `TMS-08_24x24` but never `TMSA`; `350` in a filename covers `350RL`; `PLFY-P` covers `TPLFYP008FM1M0A`), mounting (`LAY-IN`, `SURFACE`), category (diffuser = Air Terminal, fan = Mechanical Equipment), and description words. The type comes from the face and neck size (`TMS-08_24x24`), plan tag size, or capacity (nominal size in the model number first, then nearest cooling MBH; a lay-in missing a 10" type gets a duplicated type with `Neck Size` set to 10"). Returns the `.rfa` path, type, score, plain-text reasons, and top 5 candidates. A row with no credible match gets `path: null` and becomes a box at the schedule's dimensions—never a guess. You can swap any tag's family/type across all sheets using **Change** on the Fabrication tab.
- **CLI tool:** `node tools/schedule-dump.mjs <pdf> [pages]` prints every schedule, row, matched family/type, plan tag mappings across sheets, and any unread text inside schedule boxes (`--json out.json` saves it).
- **Barnes validation:** 15 schedules on H0.2, H0.3, H0.4, and H1.1 (258 marks total, all 389 plan tags matched). Families found for Titus `TMS`, `50F`, and `350RL`; Mitsubishi `PLFY` cassettes and `PEFY` ducted units; and Greenheck `CSP-A390` and `SP-A90` fans. Cleanly reports where no credible family exists in the library (`S300`, `TURYH` outdoor units, branch controllers, `MSY-GS24NA`/`MUY-GS24NA` where library only has `GL`, Markel heaters, Valent DOAS/rooftop units, Greenheck `GRS-12`).

---

## The Model, Assemble & Revit / Design Companion

- **One model is the product (`lib/model.mjs`):** Defines `pdf2cad.model` v1 (saved as `<pdf name>.p2c.json`): levels, sheets with their placement, every duct, flex, pipe, terminal, unit, and generic element with a stable UID (`sheet + kind + geometry`, so rebuilds keep the same IDs), joints, coordination settings, and clashes. Revit, IFC, and MEP Design Companion are all exports of this model.
- **Assemble (Export tab -> Assemble the set):** Reads every scaled plan sheet (trades, network, schedule families) and assigns levels from sheet titles (`FIRST FLOOR`, `SECOND FLOOR`, `ROOF`, `BASEMENT`; 14 ft floor-to-floor unless set). Zone maps, reflected ceiling plans, demolition plans, photometric plans, and enlarged plans drawn at a bigger scale than the floor plans are left out (1/8" area plans named `"ENLARGED PLAN - 1ST FLOOR - AREA I"` stay; roof plans stay; a single-story `"FLOOR PLAN"` becomes Level 1). A level's anchor is its HVAC/mechanical plan first, then piping plans.
- **Placement & Deduplication (`lib/assemble.mjs`):** Places sheets by matching their grey architectural walls against sheets already placed. A sheet whose walls don't place it with certainty borrows the placement of the same area on another floor when its walls agree there, or gets flagged `"check"`. Ducts and units drawn on two overlapping sheets are merged (`dedupeAcrossSheets`).
- **Assemble 3D View (`/assemble.html`):** Runs an oriented-box clash check (`lib/clash.mjs`, joined pairs excluded) and opens a Three.js 3D view of the whole building colored by system. Shows each sheet's walls at its level to verify placement, lets you nudge or type sheet offsets, click a clash to fly to it, click an element for its properties, press `Delete` to leave it out (`Ctrl+Z` undoes), Save, and Build Revit model.
  - *Barnes:* 10 plan sheets on 4 levels in ~7s; 8 placed automatically, H1.9 (BOE roof) and H1.10 (basement) flagged; ~1,350 elements (535 ducts, 283 pipes, 74 terminals, ~150 cassettes and ~150 thermostats among 424 units).
- **Revit without buying anything:** Driven through your local installed Revit 2025 and its free API by the PlanToRevit worker: builds one `.rvt` with every level (`/api/revit-model`), runs Revit's own clash check, writes an `.ifc` next to it using Revit's exporter, and tags every element with `"PDF2CAD Id"` (also in extensible storage) so models can be read back and updated in place by ID. No Autodesk cloud services or paid readers.
- **MEP Design Companion:** Links the `.ifc` exported next to the `.rvt` directly (Link Revit / Link IFC) with `"PDF2CAD Id"` intact. Our own IFC toolkit (`O:\Development\IfcKit`) also writes the model straight to IFC without Revit, and our `.rvt`/`.rfa` reader (`O:\Development\RvtEngine`) is being built as a browser library Design Companion can embed.

---

## Structural Drawings

Fabrication includes **Model discipline: Auto / MEP trades / Structural**. `Auto` selects structural mode on `S-` series sheets and foundation/framing titles. Assemble includes foundation, floor, and roof plans (wind diagrams, details, sections, schedules, demolition, and unscaled sheets stay out).

- **Recognition:** Detects tagged beam faces (`SB`/`RB`/`TB`/`B`), rectangular pad footings (`F`/`PF`/`FTG`), and drawn column outlines (`W`/`HSS` section labels—a W-section's weight is never mistaken for its depth). Multiple beam marks on continuous linework are reported for span assignment.
- **Manual assignment:** Assign a structural layer, or select a straight member line / closed outline and click **Assign selected geometry** (tied to the PDF and page). Select slab outlines and their openings together: orthogonal outlines decompose into disjoint solids of the same area while preserving holes (a single rotated rectangular slab also works; open, curved, or crossing slab boundaries are refused).
- **Sizing:** Enter widths, depths, heights, and bottom elevations in Fabrication. **Sizes by mark** accepts rows in the format `pattern, role, width_in, depth_in, height_ft, bottom_ft` (blank fields use defaults). Schedule interpretation for structural dimensions is manual in this version. All entered/default vertical sizes are recorded in element metadata and Comments.
- **In Assemble:** Shows member colors, editable dimensions and level elevations, source sheets, and unresolved tags. Foundation and Level 1 share the zero datum; default roof elevation comes from floor spacing. Changing level datums requires **Build Revit** (since **Update Revit** preserves existing Revit levels).
- **Scope:** These are **Generic Model coordination envelopes**, not native analytical beams, columns, or rebar. Slabs are tiled box volumes, not native Revit floors. They use the `generic` kind in `.p2c.json` so existing IFC/Design Companion imports display them without new schemas, and structural box edits use the existing worker's update route with the same `PDF2CAD Id`.

> **Note:** Keep test outputs in development/scratch folders, never in the client's job folder. The YMCA structural set is the real drawing check (automatic result is partial and includes unresolved spans).

---

## Parametric Item Catalog (`/catalog.html`)

Open **Item catalog** in the app header (or `/catalog.html`). Works without opening a PDF. Contains **31 editable starter types** and **16 materials** across architectural, structural, mechanical, plumbing, and electrical work (drawing recognizers do not yet resolve elements against this catalog):

- **Architectural:** Wall with opening, slab with opening, door/frame, window/mullions/glass, stair flight, pitched roof panels.
- **Structural:** Pad and strip footings, rectangular and round columns, concrete beam, I-section, HSS, channel, angle.
- **Mechanical:** Rectangular/round duct, elbow, transition, corrugated flex, diffuser, grille, AHU.
- **Plumbing / Electrical:**
