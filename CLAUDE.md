# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Three standalone browser tools. Open any file directly in a browser; no build step, no server, no npm.

- [json_schema_explorer.html](json_schema_explorer.html) — JSON Schema viewer (zero-dependency, fully self-contained)
- [xsd_schema_explorer.html](xsd_schema_explorer.html) — XSD / GDSN catalogue schema viewer (zero-dependency, fully self-contained)
- [image_exif_viewer.html](image_exif_viewer.html) — Image metadata viewer: EXIF, IPTC, XMP, GPS, ICC, Photoshop clipping paths. Loads `exifr@7.1.3` from jsDelivr CDN — requires an internet connection.

## Running the tools

Open any file directly in a browser (`File → Open`, or drag onto a browser tab). Changes take effect on reload. Use a local dev server (e.g. VS Code Live Server extension) for faster iteration. Note: `image_exif_viewer.html` requires internet access for the `exifr` CDN script.

## Architecture

### Shared design conventions (schema explorers)

Both schema explorer tools use the same CSS custom-property theme system. Dark mode is the default; `html.light` on the root element switches to light. Theme preference is saved to `localStorage` under the key `schema-explorer-theme` — both files share this key so they stay in sync.

Font pair: **JetBrains Mono** (code/mono) and **Outfit** (UI), loaded from Google Fonts. Loaded via a single `<link>` at the top of each file — the only external dependency.

CSS variables define the full colour palette (`--bg` through `--bg5`, `--text` through `--text3`, named semantic colours). All component styles consume these variables; never hard-code colours.

Badge classes follow the pattern `bdg b-{type}` (e.g. `b-str`, `b-ctype`, `b-seq`). Type badges use semantic colours consistent across both files.

Layout: fixed header → flex body → tree area + sidebar. Sidebar slides in/out via `width` transition (`sidebar.closed` removes it). The XSD explorer adds a resizable left module panel.

### json_schema_explorer.html

**State** (module-level `let`): `root` (parsed JSON), `expanded` (Set of path strings), `selected` (path string), `query`, `matches` (search hit paths), `matchIdx`.

**Path encoding**: the tree root is `'__r__'`; children are appended with `/key`, e.g. `__r__/products/name`. Paths double as DOM `data-path` attributes for click delegation.

**Key data flow**:
1. `load(schema, name)` — stores `root`, resets state, calls `render()`
2. `render()` → `buildNode()` — generates innerHTML from root. Recursive, depth-limited by `MAX_DEPTH = 30`.
3. `gkids(schema, isRoot, visitedRefs)` — computes children. For array schemas, transparently promotes `items.properties` children directly under the array node (no intermediate node). `$ref` schemas are resolved via `effective()` before being passed down.
4. `resolveRef(ref, visitedRefs)` — resolves `#/` JSON Pointer refs against `root`. `visitedRefs` (a `Set`) prevents infinite loops on circular `$ref` chains.
5. `onClick(path)` → toggles `expanded`, calls `render()` + `showSb(path)`
6. `showSb(path)` — calls `findAt(path)` to re-resolve the schema, then builds sidebar HTML including the JSONPath breadcrumb

**Search**: `runSearch()` does a full DFS of the schema tree, collecting paths where `nodeMatches()` returns true. `expandToPath()` ensures matched nodes are visible. Prev/next navigation cycles `matchIdx`.

**JSONPath building** (inside `showSb`): walks `jpParts` through `gkids()` and appends `[*]` when the schema type is `array`.

### xsd_schema_explorer.html

**Global indexes** (plain objects, mutated on load):
- `allFiles` — `filename → parsed XML Document`
- `allTypes` — `typeName → {el, filename, kind}` — all `complexType` and `simpleType` elements across all loaded files
- `allElements` — `elementName → {el, filename}` — all top-level `element` declarations
- `modules` — array of `{name, filename, rootTypeName, rootElementName, rootEl, isModule}`

**Module detection**: a file is a module if its name matches `*Module.xsd`. The root type is `<filename>Type` (e.g. `batteryInformationModuleType`). The root label comes from the global element whose `type` attribute matches that type name. The module tree root label is the element name (e.g. `batteryInformationModule`), not the type name.

**Path encoding**: root node is `'__root__'`; children appended as `__root__/elementName/@attributeName/…`. Paths stored as `data-path` on rows.

**Key data flow**:
1. File input → `Promise.all(FileReader)` → parse XML via `DOMParser` → `rebuildIndex()` → `renderModuleList()`
2. `selectModule(name)` → `renderTree(mod)` — builds tree HTML. `buildNode(nodeId, label, xsdEl, typeEl, depth, visited)` takes both the display element (`xsdEl`) and the resolved type element (`typeEl`) separately, because the type drives child traversal while the element drives display.
3. `getChildren(xsdEl, visited)` — the core XSD traversal. Handles `complexType`, `sequence`/`choice`/`all`, `complexContent`/`extension`/`restriction` (merges base type children inline), `group` refs (flattened), attributes, `any`. `visited` is a `Set` of type names to detect cycles.
4. `resolveChild(el, visited)` — resolves a single `element` or `attribute` declaration into `{el, label, resolvedType}`. Follows `ref` attributes to global elements.
5. `onNodeClick(path)` → toggles `expanded`, calls `renderTree()` + `showSidebar(path)`
6. `showSidebar(path)` — calls `resolvePathEl(path)` which re-walks the path through `getChildren()` from the root to find the `{el, typeEl}` pair, then builds sidebar HTML.

**Support files**: non-module XSDs appear below a divider. Clicking one calls `selectSupportFile()` which gathers top-level roots; if more than one exists, a picker replaces the tree until the user selects one.

**Search**: `runSearch(q)` calls `searchInType()` (recursive DFS, depth ≤ 8) across all modules. Results grouped by module in a dropdown. Search result items carry `data-mod` and `data-path` for navigation — these are populated from `mod.name` and the constructed path in `searchInType()`.

**XSD namespace handling**: `getSchemaPrefix(doc)` finds the `xs`/`xsd` prefix by inspecting `xmlns:*` attributes. `localName(el)` strips any namespace prefix from element tag names. Type lookups always strip prefixes before indexing/lookup.

### image_exif_viewer.html

**Style**: Light-only, navy/cream palette (not the shared theme system). Georgia serif body, monospace for labels. No dark mode toggle, no `localStorage` theme key.

**External dependency**: `exifr@7.1.3` from `cdn.jsdelivr.net/npm/exifr`. All EXIF/IPTC/XMP/GPS/ICC parsing is delegated to this library.

**Key data flow**:
1. Drop zone / file input → `processFile(file)` — creates object URL, calls `exifr.parse()` with all group flags enabled, loads image for natural dimensions, reads raw `ArrayBuffer` for 8BIM parsing.
2. `renderOutput(file, url, data, imgW, imgH, clipInfo)` — builds the full HTML output. `data` is the `exifr` result object keyed by group (`exif`, `tiff`, `gps`, `iptc`, `xmp`, `icc`). A flat merge (`flat`) is also built for convenience.
3. Clipping path section rendered first (prominent position) from `clipInfo`.
4. `makeSection(title, rows)` — collapsible section with `toggleSection(id)`.
5. After render, `attachReload()` shrinks the drop zone to a compact "Load another image" strip.

**8BIM / Photoshop clipping path parsing** (custom, no library):
- `extractApp13(buffer)` — scans JPEG for `0xFF 0xED` (APP13) segments, strips `Photoshop 3.0\0` headers, merges multiple segments.
- `extractTiffPhotoshop(buffer)` — scans raw buffer for first `8BIM` marker (TIFF/PSD fallback).
- `parse8BIM(buffer)` — iterates 8BIM resource blocks. Path resources are IDs `0x07D0`–`0x0BB6` (2000–2998); resource `0x0BB7` (2999) is the active clipping path name (UTF-16BE unicode string).

## Known issues / areas to improve

**JSON explorer**
- JSONPath builder in `showSb` has fragile array-step logic (lines 350–368)
- Search doesn't match against `enum` values or `const`

**XSD explorer**
- Search result `data-path` and `data-mod` on `sd-item` elements are often empty (navigation after clicking a result doesn't work reliably)
- `_inherited` flag set on children from base type extension but never used for visual differentiation
- No `xsd:include` support (only implicit cross-file resolution via the shared index)
- Documentation snippet comments truncated to 80 chars in `formatXmlSnippet`

**EXIF viewer**
- 8BIM path record coordinates are not decoded (selector type counts shown, not actual Bezier point data)
- `extractTiffPhotoshop` is a naive byte scan — may mis-detect `8BIM` sequences in image data for non-PSD TIFFs
- MakerNotes detection relies on key-name heuristics; structured MakerNote groups from exifr aren't broken out separately