# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Commands

```bash
pnpm dev           # Start dev server (Vite)
pnpm build         # Type-check + production build + copy spine assets to dist
pnpm preview       # Preview production build locally
pnpm lint          # Run ESLint with auto-fix
pnpm type-check    # Run vue-tsc type checking
```

- Package manager: **pnpm**. Use `pnpm install --frozen-lockfile` for CI-reproducible installs.
- Node >= 20, pnpm >= 10 (see CI workflow `.github/workflows/gh-pages.yml`).
- The build outputs to `/dist`, deployed to GitHub Pages.
- The `build` script chains `build-main` (type-check → vite build) then `copy-spines` (copies `src/assets/spines/` → `dist/assets/spines/`).

## Architecture Overview

This is a client-side **Vue 3 + TypeScript** SPA for previewing **Spine 4.1** character animations from Brown Dust 2. It runs entirely in the browser with no backend.

**Core rendering engine:** `@esotericsoftware/spine-player` (v4.1.55). All animation, skeleton, and texture logic flows through the `SpinePlayer` class from this library. WebGL canvas rendering.

**Styling:** Tailwind CSS v4 via the Vite plugin (`@tailwindcss/vite`).

### Key Architectural Points

- **`src/main.ts`** — Entry point. Creates Pinia store, applies URL query params to restore state (character/anim/skin/type), mounts Vue app.
- **`src/stores/characterStore.ts`** — Single Pinia store (`characterStore`) holding all global state: selected character, animation, skin, playback speed, background color, layer visibility, zoom state, custom background image, etc. Characters are populated from `character_list.ts`.
- **`src/App.vue`** — Root layout: Navbar (top), three-column layout (AnimationSidebar left | SpineViewer center | CharacterSidebar right) with responsive mobile fallback. Wires component events together.
- **`src/components/SpineViewer.vue`** (~2750 lines) — The heart of the app. Manages the SpinePlayer lifecycle, composite cutscene scheduling/rendering, camera/zoom controls, background image editing, layer selection (click-to-select with polygon hit-testing), screenshot export, animation export (WebM video and ZIP frames via MediaRecorder + JSZip), and the seek bar. Exposes methods via `defineExpose` for parent components.
- **`src/utils/character_list.ts`** — Static character data mapping character IDs to names, costumes, spine skeleton paths, cutscene paths, and dating paths. Each character costume is a separate entry (e.g., `100101` is "Gynt: Lugo Hunter").
- **`src/utils/cutscene_mappings.ts`** — Defines composite cutscene animations. Composites layer multiple skeletons/animations together with offset timing and external skeleton sources. Used for ultimate/cutscene animations that combine multiple Spine assets.
- **`src/utils/charIcons.ts`** — Uses Vite's `import.meta.glob` to dynamically load character icon PNGs from `src/assets/char_icons/`.
- **`src/utils/urlSync.ts`** — Bidirectional URL query parameter sync: reads `?char=...&anim=...&skin=...&type=...` on load, writes state changes to `history.replaceState`.
- **`src/types/spine-player-internal.d.ts`** — Augments the `SpinePlayer` type from `@esotericsoftware/spine-player` to expose internal fields (`config`, `bg`, `context`, `drawFrame`) used throughout SpineViewer for low-level rendering control.

### Component Tree

```
App.vue
├── Navbar.vue (title, upload, background, changelog, social links)
│   ├── UploadSpineModal.vue (drag-drop custom .skel/.json/.atlas/.png upload)
│   ├── UploadBackgroundModal.vue
│   └── ChangelogModal.vue
├── AnimationSideBar.vue (skins, animations list, layers panel, speed, bg color, export)
└── CharacterSideBar.vue (searchable character list with icons)
```

### Data Flow

1. User selects a character → `App.vue.onSelectCharacter()` sets `store.selectedCharacterId`.
2. `SpineViewer` watches `selectedCharacterId`/`animationCategory`/`showDatingBg` → calls `load()` which:
   - Resolves the animation type path (`character`/`cutscene`/`dating[_nobg]`) from character data.
   - Constructs asset URLs: `{assetRoot}/{charId}/{typeBasePath}.skel|.atlas`.
   - Creates a new `SpinePlayer` instance with the URLs.
   - On success: emits available animations/skins, selects the first animation, sets up camera bounds.
3. Changing animation sets `store.selectedAnimation` → SpineViewer watch calls `setSpineAnimation()` (or `startComposite()` for composite cutscenes).
4. Layer visibility changes in `store.layerVisibility` are deep-watched → alpha settings applied to skeleton slots.

### Spine Asset Structure

Assets are stored under `src/assets/spines/{characterId}/`:
- `char*.skel`, `char*.atlas`, `char*.png` — Character/ultimate skeleton data
- `dating/{costumeId}.skel`, etc. — Dating (Fated Guest) scenes
- `dating_nobg/{costumeId}.skel`, etc. — Dating scenes without background
- `cutscene/` — Ultimate cutscene skeleton files
- `cutscene/glow/`, `cutscene/tera/`, etc. — External skeletons referenced by composite mappings

### Cutscene Composite System

Some ultimate/dating animations are "composites" that layer multiple skeletons:
- The `cutscene_mappings.ts` defines arrays of animation segments with optional `source` paths to external `.skel` files.
- Segments can be parallel arrays (play simultaneously) or sequential items.
- `SpineViewer` loads external skeletons via `AssetManager`, creates separate `Skeleton` + `AnimationState` instances ("overlay instances"), and renders them on top of the base skeleton in a custom render loop (`renderCompositeFrame`).
- Composite playback bypasses the normal SpinePlayer render loop — it uses its own `requestAnimationFrame` cycle.

### Layer Selection Feature

- Toggled via `store.layerSelectionEnabled` (button in toolbar, or keyboard shortcuts).
- Click on the viewer canvas → raycasts through skeleton draw order slots using polygon hit-testing on world vertices.
- Selected layer is highlighted with an SVG overlay drawn on a separate canvas.
- Keyboard shortcuts: `H` hides selected layer, `U` undoes last hide, `Esc` resets all.
- Hidden layers have their alpha set to 0 in the skeleton slots.

### Background Customization

- Users can upload a custom background image (shown behind the Spine canvas).
- When editing is toggled, the background image can be dragged and resized.
- The Spine player background is rendered as transparent (the custom bg handles the background), with the configured `backgroundColor` only used as fallback.

### Rendering Tweaks

- **Premultiplied alpha patch:** `ensureGLTexturePremultiplyPatch()` patches `GLTexture.prototype.update` to always enable `UNPACK_PREMULTIPLY_ALPHA_WEBGL` — fixes incorrect alpha blending on character sprites.
- **Trimmed bounds:** `computeTrimmedBounds()` calculates character bounds excluding background-named slots to avoid large empty space from full-scene backgrounds.
