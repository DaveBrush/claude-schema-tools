# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Three standalone browser tools. Open any file directly in a browser; no build step, no server, no npm.

- [json_schema_explorer.html](json_schema_explorer.html) — JSON Schema viewer (zero-dependency, fully self-contained)
- [xsd_schema_explorer.html](xsd_schema_explorer.html) — XSD / GDSN catalogue schema viewer (zero-dependency, fully self-contained)
- [image_exif_viewer.html](image_exif_viewer.html) — Image metadata viewer: EXIF, IPTC, XMP, GPS, ICC, Photoshop clipping paths. Loads `exifr@7.1.3` and `utif@2.0.1` from jsDelivr CDN — requires an internet connection.

## Running the tools

Open any file directly in a browser (`File → Open`, or drag onto a browser tab). Changes take effect on reload. Use a local dev server (e.g. VS Code Live Server extension) for faster iteration. Note: `image_exif_viewer.html` requires internet access for the `exifr` and `utif` CDN scripts.

## Architecture

### Shared design conventions (all three tools)

All three tools share the same CSS custom-property theme system. Dark mode is the default; `html.light` on the root element switches to light. Theme preference is saved to `localStorage` under the key `schema-explorer-theme` — all files share this key so they stay in sync.

Font pair: **JetBrains Mono** (code/mono) and **Outfit** (UI), loaded from Google Fonts via a single `<link>` at the top of each file.

CSS variables define the full colour palette (`--bg` through `--bg5`, `--text` through `--text3`, named semantic colours). All component styles consume these variables; never hard-code colours.

Each tool has its own accent colour: JSON explorer = `--accent` blue, XSD explorer = `--teal`, EXIF viewer = `--pink` (`#e879a8`).

The schema explorers use a fixed-height layout (header → flex body → tree + sidebar). The EXIF viewer uses a scrollable content area (sticky header → `.content` with `overflow-y: auto`).

Badge classes follow the pattern `bdg b-{type}`. The schema explorers use `b-str`, `b-ctype`, `b-seq` etc; the EXIF viewer uses `b-ok`, `b-warn`, `b-neutral`.

The schema explorers add a resizable left module panel (XSD) or toolbar (JSON). The EXIF viewer has no sidebar — results render as a vertical stack of collapsible `.sec` cards.

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

**Style**: Shares the full CSS custom-property theme system and font pair with the schema explorers. Accent colour is `--pink` (`#e879a8`). Scrollable layout (sticky header, `.content` scrolls). Collapsible sections use `.sec` / `.sec-hdr` / `.sec-body` / `.sec-tog` classes; toggling adds/removes `.collapsed` on the outer `.sec` element.

**External dependencies**: `exifr@7.1.3` (EXIF/IPTC/XMP/GPS/ICC parsing) and `utif@2.0.1` (TIFF pixel decoding for browser preview), both from `cdn.jsdelivr.net/npm`.

**Key data flow**:
1. Drop zone / header button → `processFile(file)` — creates object URL, calls `exifr.parse()` with all group flags enabled, attempts native `<img>` load for dimensions. Reads raw `ArrayBuffer` once; if native load failed (e.g. TIFF in Chrome/Firefox), decodes with `UTIF.decodeImages()` and replaces the blob URL with a canvas-derived PNG blob. For TIFF files, also calls `readTiffDirectMeta(buf)` to extract tag 700 (XMP) and tag 33432 (Copyright) directly as a fallback when exifr misses them (common when the IFD is stored at the end of the file).
2. `renderOutput(file, url, data, imgW, imgH, clipInfo, tiffDirectMeta)` — builds the full HTML output. `data` is the `exifr` result object keyed by group (`exif`, `tiff`, `gps`, `iptc`, `xmp`, `icc`). A flat merge (`flat`) is also built for convenience. `tiffDirectMeta` supplements exifr with `{xmpRows, copyright}` parsed directly from the TIFF IFD.
3. Section order: image preview → File → Basic Image Information → Clipping Path → GPS → EXIF → TIFF/IFD0 → IPTC → XMP → MakerNotes → ICC → Other/Composite.
4. `makeSection(title, rows)` — collapsible `.sec` card. `fmtVal(v)` renders values: objects with all-primitive properties show as `key: value` line pairs (handles XMP lang-alt); newlines in strings/values render as `<br>`.
5. After render, `attachReload()` changes the drop zone to a compact `.drop-small` strip.

**TIFF direct metadata parsing** (fallback for when exifr misses tags):
- `readTiffDirectMeta(buffer)` — parses the TIFF IFD in one pass. Extracts tag 700 (XMP, passed to `parseXmpToRows`) and tag 33432 (Copyright, ASCII). Returns `{xmpRows, copyright}`. Handles little- and big-endian, standard TIFF only (magic 42).
- `parseXmpToRows(xmpStr)` — parses raw XMP XML using `DOMParser`. Iterates `rdf:Description` children; resolves lang-alt (`rdf:Alt`/`rdf:li`) to their `x-default` string. Returns `[[key, value], …]` using the local name (namespace prefix stripped).

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
- TIFF preview via UTIF: CMYK TIFFs are displayed as RGB (no true CMYK render); only the first IFD (page 0) is previewed for multi-page TIFFs; very large TIFFs may be slow to decode