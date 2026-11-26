# Project brief: "Exhibition Road", a 3D landing page you walk through

## 1. Vision (read this first)

Build a mobile-first 3D landing page that feels like walking through an open-air exhibition. The visitor moves slowly along a single road. At three stops ("stations") beside the road there is an exhibit, like a sculpture next to its placard in a gallery: a sign on the LEFT (title + description) and a 3D diorama on the RIGHT.

The visitor can stop anywhere, wait as long as they like, look freely in every direction (the sign, the diorama, the sky, back down the road), then move on or walk back. Nothing pulls or pushes them. The mood is calm and contemplative, not a racing game. The whole appeal is lingering in a world that is genuinely 3D, so every exhibit must be real 3D geometry, never a flat picture.

The sample content is for a retirement and life-insurance company; the road is a metaphor for a life's journey. All content lives in one config file so it can be swapped later.

Keep the code simple, readable and commented. A beginner should be able to add a station. Prefer plain code over clever abstractions. Every tunable number is a named constant in config.js.

## 2. Scope

IN:
- First-person camera on a road, three stations, each with a sign and a primitive-built 3D diorama
- Two on-screen buttons (forward / backward) plus free drag-to-look
- Small HUD (see section 9)

OUT (do not build, not even as a nice touch):
- No "next stop" button, auto-drive, auto-recentering, snapping, magnetic stops or slow-down zones. The user is always in full control.
- No limits on horizontal look; no camera limits other than pitch (see section 5)
- No saving of progress: no localStorage, sessionStorage, cookies. A refresh resets everything.
- No desktop fallback or "open on your phone" message, and no keyboard or mouse-specific controls
- No external 3D models (GLB/GLTF/OBJ), no image files, no external fonts, no audio
- No tap/zoom interaction on signs, no call-to-action buttons, no forms, no analytics
- No build tools, npm or frameworks

## 3. Tech constraints

- Three.js r164.1, loaded through an importmap from a CDN:
    <script type="importmap">{"imports":{"three":"https://cdn.jsdelivr.net/npm/three@0.164.1/build/three.module.js"}}</script>
- Vanilla JavaScript ES modules. It must run when served by any static file server (for example `python -m http.server`) and on GitHub Pages.
- Everything visual is procedural: meshes built from primitives in code, sign text drawn onto a canvas and used as a texture. Use a system font stack.
- Mobile portrait is the design target. Landscape must not break (handle resize); no special landscape handling.

## 4. World model: a tiny planet

- The camera stays fixed at (0, 1.7, 0), looking toward -Z by default. Vertical FOV about 70, near 0.05 (the user may walk right up to a sign), far about 300.
- A Group called `planet`, centered at (0, -PLANET_RADIUS, 0), contains everything on the ground: the ground sphere, the road, roadside props and all stations.
- One number, `progress` (distance travelled along the road, in world units), drives everything: planet.rotation.x = progress / PLANET_RADIUS. Moving forward must bring the ground toward the camera from -Z. Verify visually and flip the sign if needed.
- A station at distance s along the road: put it in a pivot Group at the planet's center with pivot.rotation.x = -s / PLANET_RADIUS, and the station Group at (0, PLANET_RADIUS, 0) inside the pivot.
- Station local frame: origin on the road, +Y up (away from the planet), -Z = direction of travel, +X = right.
- Starting values (all tunable): PLANET_RADIUS 150, first station at s = 60, stations 90 apart, road ends 40 after the last station. Progress is clamped to [0, last station + 40].
- The curvature is a feature: from a station, the NEXT station must not be visible. It appears over the horizon only as the user approaches. Keep dioramas at most 9 units tall and about 11 wide, and tune the radius, spacing and fog so this holds. Check it from each station.
- Sky and fog use the same soft color (fog roughly 25 to 90) so nothing pops in. The world must look good in every direction because the user can look behind and up.
- Hide (visible = false) any station farther than about 80 units from the camera, and only call update() on visible stations.
- Road: dark band along the planet's great circle with a dashed center line and slightly lighter verges. Ground: flat-shaded green sphere with enough segments (64 x 32) to look round.
- Roadside props: about 60 trees and lamp posts as InstancedMesh, placed with a SEEDED random function (same layout every load), alternating sides along the road. They give a sense of speed and parallax.

## 5. Controls

Movement is on rails: forward and backward always mean along the road, no matter where the camera is looking. The user can look at a sign while still moving forward.

- Two large round buttons (about 76 px) stacked in the bottom-right corner, forward on top. While held, the target speed is +/- MAX_SPEED (start at 8 units/s); when released, the target is 0. Smooth the actual speed exponentially (rate about 5 per second) so it eases in and out. Use delta time, clamped to 0.05 s.
- Dragging anywhere else on the canvas rotates the camera. Yaw is unlimited (full 360 degrees, wrap freely). Pitch is clamped to +/-89 degrees, only to avoid flipping upside down. Use camera.rotation.order = 'YXZ'. Style: grab the world, so dragging right pans the view left. LOOK_SENSITIVITY is about 0.005 rad/px. Add an INVERT_LOOK constant. No inertia and no recentering.
- Holding a button with one thumb while dragging with the other must work at the same time. Use Pointer Events (not touch events) with setPointerCapture on the buttons; track the drag by pointerId; ignore drags that start on a button.
- Set touch-action: none on the canvas and buttons so the browser never scrolls, pull-to-refreshes or zooms during a drag. Also: user-select: none, -webkit-touch-callout: none, no context menu on long press, viewport meta with user-scalable=no, use 100dvh, and respect env(safe-area-inset-*) for the buttons.

## 6. Stations

Each station = a sign + a diorama, defined in config.js:

  SECTIONS = [
    { id, title, description, build }   // build(parent: THREE.Group) => { update?(time, dt) } | void
  ]
  SIGN_LAYOUT  = { x: -4.5, z: -2, rotationY: 0.45, boardWidth: 5, boardHeight: 3.75 }
  SCENE_LAYOUT = { x:  7.0, z: -2, rotationY: -0.45 }

- Layout rule: the sign is on the left of the road, angled toward the road and the approaching visitor; the diorama is on the right, its front also angled toward the road. A build() function builds its diorama centered on its own origin, front toward +Z, footprint radius about 5.5, resting on y = 0. The engine positions and rotates it with SCENE_LAYOUT.
- The "sweet spot" (station origin) is only a layout reference. Users stop wherever they like. Tune the layout so that from about 20 units before the sweet spot both sign and diorama are at least partly visible in a portrait viewport, and from the sweet spot a turn of about 60 to 70 degrees left or right frames each one.
- Sign (sign.js): two wooden posts, a thick board, and a face plane with a CanvasTexture. Canvas at least 1536 x 1152. Cream background, dark text, a thin accent bar. Title is bold and large (wrap to at most 3 lines), description is smaller with comfortable line height. Write a small wrapText helper. Texture: SRGBColorSpace, max anisotropy, mipmaps on. The face uses MeshBasicMaterial with toneMapped: false so the text is crisp and lighting-independent. Text must stay readable when the user walks up close.

## 7. Sample content (placeholder copy; "..." stands for an amount to be filled in later)

Station 1: "Time together"
- Title: More time with the people you love.
- Description: Pay ... $ a year and be ready for the day your career ends. Spend it where it matters: at home, with the people you love. That's what our retirement insurance is for.
- Diorama, "Front-yard baseball": a rounded green lawn slab as the base. A classic two-storey suburban house: cream siding box, gabled dark-red roof (triangular prism), brick chimney, blue front door with two steps and a small porch roof on two posts, four windows with white frames and a faintly emissive pale-blue glass. A white picket fence along the front of the lawn with a gap. A mailbox with a red flag. One round tree (brown trunk, two or three overlapping green low-poly spheres). In the foreground, a father (tall, blue shirt, cap) and a son (small, red shirt, cap, holding a thin bat), both built with addPerson. A white ball flies in an arc between them about every 2 seconds, animated in update().

Station 2: "Peace of mind"
- Title: Let the storm pass. You're covered.
- Description: Life insurance that keeps your family protected, whatever tomorrow brings. For ... $ a month, the people you love never have to worry about the bills.
- Diorama, "Dry under the umbrella": a round base, lighter and warmer under the umbrella, with a few dark-blue flat puddles around it. A big red-and-cream striped umbrella (hemisphere canopy built from alternating-colored wedges, wooden pole planted in the ground). Under it, a wooden bench with an older couple seated, grey hair, a blanket over their knees (addPerson with seated: true). A grey cloud (4 to 6 flattened overlapping spheres) about 8 units up. Rain: about 120 thin blue-grey boxes as an InstancedMesh, looping from the cloud to the ground in update(); spawn positions are restricted to OUTSIDE the canopy radius, so the rain falls around the umbrella and never under it.

Station 3: "Freedom"
- Title: Retire into your next adventure.
- Description: The road doesn't end when the career does. Plan ahead with ... $ a year and be free to go wherever you've always wanted. No schedule, no worries.
- Diorama, "Campervan by the lake": a small lake (slightly emissive blue disc) at the back, with low squat mountains (3 flat-shaded cones, max 7 units tall) behind it. A mint-and-white camper van built from boxes: body, cab with a sloped windshield, dark-blue windows, four cylinder wheels with light hubs, a stripe, emissive headlights, a striped orange-and-cream awning on two poles. Two folding camp chairs with two seated figures seen from behind, facing the lake. A campfire: ring of grey rocks, three logs, and 2 or 3 stacked emissive cones (orange and yellow) whose scale flickers in update(). 4 pine trees (stacked cones). Do NOT use a PointLight for the fire; emissive materials only.

## 8. Art direction and primitives

- Low-poly, flat-shaded, toy-like, one consistent palette. Suggested: sky/fog #bfe3f0, grass #7bbf6a, road #4a4d52, line #f2e9c9, wood #8a5a3b, cream #f3e9d2, brick #b5533c, roof #7a3b2e, red #d64541, blue #3f78c4, warm accent #f2a541. Tweak if you want, but stay consistent.
- primitives.js exports small helpers in the style of: addBox(parent, size, position, color, options), addCylinder, addCone, addSphere, addCapsule, addPrism (triangular prism for roofs), addDisc. Materials are MeshStandardMaterial with flatShading: true, roughness about 0.9, and are cached by color. Reuse geometries where possible. Meshes cast and receive shadows by default.
- addPerson(parent, { height, shirt, pants, skin, hair, hat, seated, pose }) builds a simple figure (legs, torso, sphere head, capsule arms), about 12 meshes. Stylized and faceless is fine.
- Lighting: one HemisphereLight plus ONE DirectionalLight fixed in world space whose shadow frustum covers only about +/-22 units around the camera (shadow map 1024). No other shadow-casting lights, and no PointLights. Provide ENABLE_SHADOWS in config.
- Each diorama stays under about 150 meshes.

## 9. HUD and page text

- Three small non-interactive progress dots at the top center (pointer-events: none); the dot of the nearest station lights when within a third of the spacing.
- A first-use hint ("Hold the arrows to move. Drag to look around.") that fades out after the first movement. Non-interactive.
- Buttons have aria-labels. A visually hidden <main> mirrors the three titles and descriptions as real HTML for screen readers and search engines, generated from SECTIONS.

## 10. Performance (mid-range phone)

Target 60 fps. Fewer than about 150 draw calls and about 100k triangles visible at once. Pixel ratio capped at 2. No postprocessing. No textures other than the three sign canvases. Dispose nothing per frame; allocate nothing in the render loop.

## 11. File: a single index.html

Deliver the entire project as ONE file, `index.html`. Nothing else: no separate .js or .css files, no README, no assets. The only external request allowed is the Three.js CDN import. The file contains a <style> block, the importmap, and one inline <script type="module"> block.

Organize the module script in this order, with each part under a banner comment such as `// ===== 3. SIGNS =====` so the file is easy to navigate. The order matters, because constants must be declared before code that uses them:

1. CONFIG: every tunable as a named constant (speeds, radius, spacing, fog, FOV, sensitivity, palette, SIGN_LAYOUT, SCENE_LAYOUT, ENABLE_SHADOWS, INVERT_LOOK)
2. PRIMITIVES: addBox, addCylinder, addCone, addSphere, addCapsule, addPrism, addDisc, addPerson, with the material cache
3. SIGNS: wrapText, the canvas drawing and the sign mesh builder
4. SCENES: the three diorama build functions, one per station
5. SECTIONS: the content array ({ id, title, description, build }), placed after the scenes because it refers to them
6. WORLD: renderer, camera, lights, planet, road, roadside props, station placement
7. CONTROLS: drag-to-look, hold buttons, speed easing
8. HUD: progress dots, first-use hint, hidden text generated from SECTIONS
9. MAIN LOOP: update and render, plus the resize handler

Put one block comment at the very top of the file (it replaces the README) with:
- how to run it: serve the folder with `python -m http.server` and open it on a phone over the same Wi-Fi
- how to add a station: write a build function, add an entry to SECTIONS
- a list of the main tunable constants
- a short "Decisions" list for anything you had to assume

Comment the code generously, so a beginner can follow it. At every milestone, deliver the complete working index.html, replacing the previous version.

## 12. Milestones (pause after M1 and after M3 so I can test on a phone)

M1: renderer, sky and fog, planet, road, controls, HUD, and three plain placeholder crates as stations. Verify movement, free look, simultaneous hold + drag, and that the next crate stays hidden behind the horizon.
M2: sign system and the station layout.
M3: primitives.js plus the Station 1 diorama.
M4: Stations 2 and 3.
M5: roadside props, hidden text, performance pass, README.

## 13. Acceptance checklist

- No console errors; only network request is the Three.js CDN
- Hold forward: eases up to speed; release: eases to a stop and stays stopped indefinitely; backward works; clamped at both ends of the road
- Drag turns the camera a full 360 degrees; pitch stops short of straight up and down; the view never moves by itself
- One thumb on a button plus the other dragging works at once
- From each station's sweet spot, the next station is not visible; the previous ones recede naturally behind
- Sign on the left, diorama on the right at every station; sign text is readable up close
- Refresh puts the user back at the start; nothing is stored
- Every diorama is made only of primitives and looks recognizable at a glance
- Stays smooth on a mid-range phone

If anything is ambiguous, choose the simplest reading, record it in the README "Decisions" section, and do not add features that are not listed.