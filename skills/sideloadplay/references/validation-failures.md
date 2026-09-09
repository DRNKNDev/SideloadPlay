# `sideload validate` failure ids

Every id `sideload validate` (and, for the same checks, `sideload publish`) can
report, in the `{ id, actual, expected }` shape. This is the complete set —
don't invent an id that isn't here.

## Schema failures

| id | Means |
|---|---|
| `schema.<field.path>` | `sideload.json` fails schema validation at `<field.path>` — wrong type, a string outside its length limit, a value not matching a required pattern, or an enum value that isn't one of the allowed ones. The path uses dots for nesting, e.g. `schema.developer.name`, `schema.requires.network`. Array entries appear as a zero-based index segment, e.g. `schema.tags.0`, `schema.screenshots.2`. |

Common ones you'll actually hit:

- `schema.slug` — doesn't match the lowercase-hyphenated pattern, or is empty / over 64 characters.
- `schema.title` / `schema.short_description` / `schema.description` — over their length limit (60 / 140 / 4,000 characters).
- `schema.version` — not valid semver (`1.10` instead of `1.10.0`, for example).
- `schema.tags` — an empty or over-5 array; `schema.tags.<n>` (e.g. `schema.tags.0`) for an entry not in the fixed vocabulary.
- `schema.engine` / `schema.renderer` / `schema.controller_support` / `schema.lifecycle` / `schema.content_rating` — a value outside that field's enum.
- `schema.input` — an empty array, or an entry that isn't `keyboard`, `mouse`, `gamepad`, or `touch`.
- `schema.estimated_playtime_min` — missing, not an integer, or not positive.
- `schema.developer.name` — missing (the only required field inside `developer`).
- `schema.requires.<pointer_lock|fullscreen|network|shared_array_buffer>` — one of the four is missing or not a boolean. All four are required with no default.
- `schema.size.initial_load_mb` / `schema.size.total_mb` — missing or not a number.

For any of these, `actual` shows what was actually in `sideload.json` at that
path (or `"missing"` if the field wasn't present at all), and `expected`
describes the constraint that failed — a type, a range, a pattern, or the list
of allowed values.

## Structural failures

| id | Means |
|---|---|
| `entry_file_missing` | The file named by `entry_file` (default `index.html`) does not exist in the build directory. |
| `description_file_missing` | `description` names a path ending in `.md`, but that file doesn't exist relative to `sideload.json`. |
| `description_length` | The resolved description text — inline, or read from the `.md` file — exceeds 4,000 characters. For a file reference, this is checked against what the file actually contains, not the length of the path string. |
| `manifest_unreadable` | `sideload.json` is absent from the directory, or is not valid JSON. `actual` carries the parser's own message. Nothing else runs until this passes — it is the read that every other check depends on. |
| `asset_missing` | A path referenced under `assets` (`cover`, `hero`, `logo`, an entry in `screenshots`, or `trailer`) doesn't exist relative to `sideload.json`. `actual` names which asset slot and the path that was missing. |

## Artwork failures

Checked for every asset that's actually present and exists on disk.

| id | Means |
|---|---|
| `asset_size` | The file (any asset other than the trailer) exceeds 5 MB. |
| `cover_format` | The cover file isn't a recognizable PNG or JPEG. |
| `hero_format` | Same, for the hero. |
| `screenshot_format` | Same, for a screenshot. |
| `cover_dimensions` | The cover isn't exactly 1920×1080. `actual` shows the real dimensions, e.g. `1920x1200`. |
| `hero_dimensions` | Same, for the hero. |
| `screenshot_dimensions` | Same, for a screenshot. |
| `logo_format` | The logo file isn't a PNG (it must be a PNG specifically, not just any raster format). |
| `logo_dimensions` | The logo exceeds 1200×400 in either dimension. `expected` reads `within 1200x400` — unlike the other dimension checks, the logo doesn't need to be an exact size, just fit inside the box. |
| `screenshot_count` | At least one screenshot is declared, but the total is fewer than 3 or more than 8. `expected` reads `3 to 8`. (Declaring zero screenshots doesn't trigger this — it's a `draft`-vs-`in_review` outcome instead, reported separately; see below.) |

## What isn't a failure: draft vs. in review

A cover plus at least three screenshots is what moves a listing out of
`draft` into review — but having fewer than that is not itself a validation
*failure*. `sideload validate` still reports success (`ok: true`, no
failures) and instead previews what `publish` will do, naming what's missing
by the same `{ id, actual, expected }` shape (`id: "cover"` if the cover
itself is absent, `id: "screenshots"` if there are fewer than three):

```
All checks passed. publish would create this listing as a draft. Missing: screenshots (have 0, need >=3).
```

or, once artwork is complete:

```
All checks passed. publish would submit this listing for review.
```

This is reported as `missing: [{ id: "screenshots", actual: "0", expected: ">=3" }]`
in the `--json` output, alongside `status: "draft"` — a different field from
`failures`, and not something to treat as a reason validation "failed."
