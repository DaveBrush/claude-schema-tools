# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Four standalone browser tools. Open any file directly in a browser; no build step, no server, no npm.

- [json_schema_explorer.html](json_schema_explorer.html) — JSON Schema viewer (zero-dependency, fully self-contained)
- [xsd_schema_explorer.html](xsd_schema_explorer.html) — XSD / GDSN catalogue schema viewer (zero-dependency, fully self-contained)
- [image_exif_viewer.html](image_exif_viewer.html) — Image metadata viewer: EXIF, IPTC, XMP, GPS, ICC, Photoshop clipping paths. Loads `exifr@7.1.3` and `utif@2.0.1` from jsDelivr CDN — requires an internet connection.
- [json_schema_csharp_converter.html](json_schema_csharp_converter.html) — JSON Schema → C# POCO class generator (zero-dependency, fully self-contained)

## Running the tools

Open any file directly in a browser (`File → Open`, or drag onto a browser tab). Changes take effect on reload. Use a local dev server (e.g. VS Code Live Server extension) for faster iteration. Note: `image_exif_viewer.html` requires internet access for the `exifr` and `utif` CDN scripts.

## Architecture

### Shared design conventions (all four tools)

All four tools share the same CSS custom-property theme system. Dark mode is the default; `html.light` on the root element switches to light. Theme preference is saved to `localStorage` under the key `schema-explorer-theme` — all files share this key so they stay in sync.

Font pair: **JetBrains Mono** (code/mono) and **Outfit** (UI), loaded from Google Fonts via a single `<link>` at the top of each file.

CSS variables define the full colour palette (`--bg` through `--bg5`, `--text` through `--text3`, named semantic colours). All component styles consume these variables; never hard-code colours.

Each tool has its own accent colour: JSON explorer = `--accent` blue, XSD explorer = `--teal`, EXIF viewer = `--pink` (`#e879a8`), C# converter = `--purple` (also reassigned as `--accent` in that file; a separate `--blue` variable holds the literal blue used for type-name syntax highlighting).

The schema explorers use a fixed-height layout (header → flex body → tree + sidebar). The EXIF viewer uses a scrollable content area (sticky header → `.content` with `overflow-y: auto`). The C# converter uses the same fixed-height header/sidebar layout as the schema explorers, with a scrollable code pane in place of a tree.

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
- `readTiffDirectMeta(buffer)` — parses the TIFF IFD in one pass. Extracts: tag 700 (XMP → `parseXmpToRows`), tag 33432 (Copyright, ASCII), structural tags 258/259/262/277/282/283/296/338 (BitsPerSample, Compression, PhotometricInterp, SamplesPerPixel, XResolution, YResolution, ResolutionUnit, ExtraSamples), and tag 34675 (ICC profile bytes → `parseIccHeader`). Returns `{xmpRows, copyright, structRows, iccRows}`. Handles little- and big-endian, standard TIFF only (magic 42). All four output fields are used as fallbacks when exifr's corresponding groups are empty.
- `parseXmpToRows(xmpStr)` — parses raw XMP XML using `DOMParser`. Iterates `rdf:Description` children; resolves lang-alt (`rdf:Alt`/`rdf:li`) to their `x-default` string. Returns `[[key, value], …]` using the local name (namespace prefix stripped).
- `parseIccHeader(u8)` — parses the 128-byte ICC profile binary header (big-endian). Extracts: device class, color space, PCS, rendering intent, ICC version, primary platform, creation date, profile size. Also walks the tag table to find the `desc`/`mluc` tag for a human-readable profile description (supports both ICC v2 ASCII `desc` and ICC v4 UTF-16BE `mluc` formats).

**8BIM / Photoshop clipping path parsing** (custom, no library):
- `extractApp13(buffer)` — scans JPEG for `0xFF 0xED` (APP13) segments, strips `Photoshop 3.0\0` headers, merges multiple segments.
- `extractTiffPhotoshop(buffer)` — scans raw buffer for first `8BIM` marker (TIFF/PSD fallback).
- `parse8BIM(buffer)` — iterates 8BIM resource blocks. Path resources are IDs `0x07D0`–`0x0BB6` (2000–2998); resource `0x0BB7` (2999) is the active clipping path name (UTF-16BE unicode string).

### json_schema_csharp_converter.html

Converts a dropped JSON Schema (draft 2019-09/2020-12 style, `$defs`/`$ref`) into C# POCO classes, matching the conventions of hand-tuned quicktype.io output (Newtonsoft `JsonProperty`/`NullValueHandling.Ignore`, `decimal`/`long` for number/integer, PascalCase from camelCase).

**No tree/sidebar-for-inspection here** — the sidebar is a live options form (root class name, namespace, JSON library, numeric types, and toggles); the main pane is a syntax-highlighted `<pre>` of the generated code plus Copy/Download buttons. There is no persistence between sessions; drop a schema, tune options, regenerate is instant and local (nothing leaves the browser).

**Generation model** (module-level `let ctx`, rebuilt on every regenerate via `makeCtx()`): each generated class/enum/union is a mutable "record" object pushed to `ctx.allRecords` in discovery order (this becomes render order) and, when it originates from a `$ref`, indexed in `ctx.registryByPointer` by the JSON Pointer string — this is both the dedup key (a second `$ref` to the same pointer reuses the record) and the cycle guard (the record is registered *before* its members are populated, so a self-referential schema resolves to the same record instead of recursing forever).

**Key data flow**:
1. `generate(ctx, schemaRoot, options)` — resolves the root schema (handling a top-level `$ref` or `type: array`), creates the root class record, then calls `populateClassMembers()` which recurses through the whole schema via `resolveSchemaType()`.
2. `resolveSchemaType(ctx, schema, nameHint)` — the core dispatcher. Resolves `$ref` (via `resolveRefType`), merges `allOf` branches that aren't `if`/`then` conditionals (`mergeAllOfIfNeeded`), unwraps `type: [X, "null"]` and nullable `oneOf`/`anyOf` patterns, and dispatches on the effective JSON type to build a typed `{k, ...}` reference (`prim`, `class`, `enum`, `union`, `list`, `dict`).
3. `resolveRefType(ctx, ref, nameHint)` — resolves a `#/$defs/...` pointer. A bare constrained primitive (e.g. a def that's just `{type: "string", maxLength: 14}`) is inlined as a primitive with its constraints carried up (no class generated); an object schema becomes a class record named from the pointer's last segment (falling back to the full dotted path, then a numeric suffix, on name collision).
4. Real `oneOf`/`anyOf` unions (not the nullable-wrapper pattern) become a `partial struct` with one nullable field per alternative plus a hand-rolled `JsonConverter` that branches on `JsonToken` — the same shape quicktype itself emits for anonymous unions (Newtonsoft only; System.Text.Json mode always falls back to `object` with a warning).
5. `runStructuralMerge(ctx)` (when "merge structurally identical classes" is on) — iterates to a fixed point, grouping live records by a structural signature (member names/types/required-ness, resolved through any existing merge chain) and setting `.mergedInto` on later duplicates; `follow(rec)` walks that chain at render time, so merging never requires rewriting other records' type references.
6. `renderAll(ctx, meta, sourceLabel)` — renders every non-merged record via `renderClass`/`renderEnum`/`renderUnion`, then optionally a `FromJson`/`ToJson` helper (`renderHelper`, mirroring quicktype's own `Converter.Settings` pattern).

**Naming**: `toPascalCase()` splits camelCase/acronym runs the same way quicktype does (`nonGTINLogisticsUnitInformationModule` → `NonGtinLogisticsUnitInformationModule`) via two regex passes (lower/digit→upper boundary, then acronym-run→word boundary) rather than a lookup table. Enum member identifiers are sanitized separately (`sanitizeEnumIdentifier`) and checked against a C# reserved-word set — a colliding member (e.g. a JSON enum value literally `"string"`) is escaped with a leading `@` rather than renamed, so it still round-trips through `StringEnumConverter`/`JsonStringEnumConverter` without an `[EnumMember]`. A member whose PascalCase name would collide with its own enclosing class name (e.g. a `packagingWeightReduction` object with a same-named nested `packagingWeightReduction` field — a real pattern in GDSN schemas) is suffixed `...Property` in `ensureUniqueMemberName`, since C# forbids a member sharing its enclosing type's name; the JSON property name is untouched.

**No length/range validation converters**: `maxLength`/`minimum`/`maximum` constraints are read from the schema but never turned into a `JsonConverter` attribute or emitted as a class — deliberately dropped after v1 emitted per-length `String{N}MinMaxLengthCheckConverter`/`MinMaxValueCheckConverter` stubs (informed by a real hand-patched quicktype output file). That validate-on-deserialize behavior throws on the first oversized/out-of-range value in real-world data, which for interop formats like GDSN is worse than accepting it; removing it also shrank output ~17% on a large real-world schema. Constraint values are silently ignored rather than surfaced as XML doc comments — a possible future middle ground.

**Known simplification**: unions inside a union alternative that are themselves arrays (e.g. a recursive "any JSON value" schema with an array-of-self branch) are not modeled as a struct — `tryBuildUnion` returns `null` for any array alternative and the property falls back to `object`, surfaced as a warning rather than silently dropped.

**Class name overrides**: a textarea (`#optRenames`, one `GeneratedName -> DesiredName` line per rule, parsed by `parseRenameOverrides`) lets the user rename specific classes/enums/structs post-generation — e.g. GDSN's `Measurement` colliding with a name they already use elsewhere, or un-collapsing a merged class's arbitrary canonical name (`Description2500` → `Description`). `applyRenameOverrides(ctx, overrides)` runs after `generate()`/merging, matching by the record's *current* name, renaming in place (mutating `rec.name`, not touching `.mergedInto` chains — so `renderTypeRef`'s `follow()` picks up the new name automatically everywhere the type is referenced) and pushing a warning instead of applying a rename that would collide with another live record's name. If the root record itself is renamed, `regenerate()` re-reads `meta.rootRecordName` from the registry afterward so the `FromJson`/`ToJson` helper uses the new name. Persisted to `localStorage` (`json-csharp-converter-renames`, independent of the schema-explorer theme key) since the same schema is typically re-converted repeatedly as it evolves and the same renames apply each time.

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

**C# converter**
- Class/property names never collide with C# keywords (PascalCase always capitalizes), but enum *member* names can — handled via a leading `@`. A generated type name that shadows a common BCL type (e.g. an enum literally named `Type`, from a JSON discriminator field) is not renamed, but this is safe: all generator-emitted converter/helper code fully qualifies `System.Type`/`System.Exception`/`System.Nullable` rather than relying on `using System;`, specifically so a same-named generated type in the same namespace can never shadow them (this was a real compile break, found by actually compiling generated output against a large schema)
- `oneOf`/`anyOf` unions with an array alternative, or more than one object alternative, always fall back to `object` (see "Known simplification" above) rather than attempting a `List<T>` field on the union struct
- "Merge structurally identical classes" compares resolved member signatures, and since `maxLength`/`minimum`/`maximum` constraints aren't part of that signature (see "No length/range validation converters" above), `$defs` that differ only in those constraints always collapse into one class — same tradeoff quicktype itself makes
- No `patternProperties`, `if`/`then`/`else` branching (beyond being skipped inside `allOf`), or JSON Schema `not` support — these are simply ignored rather than modeled