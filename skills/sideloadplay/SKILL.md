---
name: sideloadplay
description: Prepare a web game to be published on Sideload — author sideload.json,
  produce the required store artwork, and verify both by running `sideload validate`.
  This skill gets a game ready *for* publishing; it does not itself sign in, upload,
  or publish anything. Those are the sideloadplay CLI's own commands (`login`,
  `publish`, `dev`), which the developer runs themselves once this skill's checks
  pass. Use when the user wants to list a Three.js, React Three Fiber, Babylon, or
  PlayCanvas web game on Sideload, or asks what a Sideload listing requires.
---

## Scope

This skill prepares a listing; it does not publish one. It writes `sideload.json`,
produces the artwork it references, and runs `sideload validate` to confirm both
are correct — then hands off. Signing in, uploading a build, and creating or
updating the listing are `sideload login` and `sideload publish`, CLI commands the
developer runs themselves; running the build in the launcher is `sideload dev`.
This skill does not invoke, script, or wrap any of the three, and nothing here
should ask for or handle the developer's invite token.

The point of doing this before validating: `sideload validate <dir>` reports what
is wrong, one `{ id, actual, expected }` failure at a time. Everything below is a
rule that check enforces but a developer (or an agent working from a build folder
alone) would not otherwise guess. Read it, author accordingly, and validation
should pass on the first run.

## Workflow

1. Confirm the game builds to a static directory with an entry HTML file
   (`index.html` unless you set `entry_file`).
2. Write `sideload.json` at the root of that directory — see **Authoring
   sideload.json** below.
3. Produce the artwork it references — see **Artwork** below.
4. Check the game against **The traps** before you consider it done. They are
   invisible until the game runs inside Sideload's container, and `validate`
   catches none of them — it never reads your entry HTML.
5. Run `npx sideloadplay validate <dir>` (see **Checking your work**). Fix every
   failure it reports and re-run until it passes. Do not tell the developer the
   listing is ready without having actually run this and seen it pass — reading
   this skill is not the same as validating.

Once `validate` passes, this skill's job is done. What comes next is the
developer's own, on the command line, not something to script on their behalf:

```sh
npx sideloadplay login    # once, reading an invite token from standard input
npx sideloadplay publish <dir>
```

## Authoring sideload.json

```json
{
  "$schema": "https://sideloadplay.com/schema/sideload.schema.json",
  "slug": "your-game",
  "title": "Your Game",
  "short_description": "One line, at most 140 characters.",
  "description": "./DESCRIPTION.md",
  "version": "1.0.0",
  "entry_file": "index.html",
  "engine": "threejs",
  "renderer": "webgl2",
  "tags": ["Racing"],
  "input": ["keyboard", "mouse", "gamepad"],
  "controller_support": "full",
  "lifecycle": "unmanaged",
  "estimated_playtime_min": 90,
  "languages": ["en"],
  "content_rating": "everyone",
  "developer": { "name": "Your Studio" },
  "requires": {
    "pointer_lock": true,
    "fullscreen": true,
    "network": false,
    "shared_array_buffer": false
  },
  "size": { "initial_load_mb": 8, "total_mb": 24 },
  "assets": {
    "cover": "./store-assets/cover.png",
    "screenshots": ["./store-assets/1.png", "./store-assets/2.png", "./store-assets/3.png"]
  }
}
```

The `$schema` line is what saves you reading a field reference: any editor that
understands JSON Schema gives you every field, its type, whether it is required,
and the exact list of valid tags as you type. Include it and let the editor
carry the schema; the rules below are the ones it cannot tell you.

**`description`** is either markdown inline or a path to a `.md` file, resolved
relative to `sideload.json`. Use a file once it runs past a couple of lines — a
4,000-character description is not something to hand-edit inside a JSON string.

**Paths under `assets`** resolve relative to `sideload.json`, wherever that file
sits in the build directory.

**`size`** is what the listing shows a player before they launch:
`initial_load_mb` is what must arrive before the game is playable,
`total_mb` everything it will eventually fetch.

**`requires.network`** must be honest, not just correct when you write it. The
launcher enforces it: with `false`, every outbound request at runtime is blocked
except to the game's own origin. Set `true` if the game calls anything — a
leaderboard, an analytics beacon, a CDN font — even once. `false` is a promise
the container keeps for you, not a setting the game can quietly outgrow.

**Never invent a value the developer has not given you** — a playtime, a content
rating, a tag that merely sounds right for the genre. `estimated_playtime_min`,
`content_rating` and `tags` are judgement calls about the game, not facts you can
read out of its code. Ask, or leave them.

## Artwork

Everything raster is 16:9 at exactly **1920×1080** — cover, hero, and every
screenshot, no exceptions and no "close enough." The logo is the only asset
with different dimensions: a transparent PNG that fits **within 1200×400**
(it does not have to fill that box, just not exceed it). Every file, other
than the trailer, is capped at **5 MB**.

| Asset | Dimensions | Format | Required |
|---|---|---|---|
| Cover | exactly 1920×1080 | PNG or JPEG | yes |
| Screenshots | exactly 1920×1080 | PNG or JPEG | 3 to 8 |
| Hero | exactly 1920×1080 | PNG or JPEG | optional |
| Logo | within 1200×400 | PNG, transparent | optional |
| Trailer | MP4, 1080p, ≤2 min | — | optional |

A cover is required to leave `draft`. **Three screenshots is the minimum** that
gets a listing into review — fewer than three, and it stays a draft no matter
how complete everything else is. Eight is the maximum; extra screenshots beyond
eight are a validation failure, not a trimmed gallery.

**Screenshots must be actual gameplay** — no mockups, no concept art, no photo
of a monitor. This is a review-time rule, not something `sideload validate`
can check by inspecting pixels, but violating it gets a listing sent back all
the same. A gameplay screenshot at the right dimensions is also a perfectly
valid Cover — you don't need a separate marketing render.

**Safe area:** keep the subject inside the middle 60% of the image's
height — roughly the center 650px of a 1920×1080 source. Every surface that
shows this artwork crops it, and every one of them crops by height, never by
width: a wide card, a square-ish tile, and the game page's header band all
just take a shorter vertical slice of the same 1920-wide image. Put the
subject anywhere within that vertical band and no surface cuts it off; put it
near the top or bottom edge and something eventually will.

## The traps

`sideload validate` catches none of these — it checks `sideload.json` and the
artwork files, and never reads your entry HTML. They are what actually costs
time: a game that looks correct on your own machine and fails only once it
runs inside Sideload's launcher. `sideload dev` is what surfaces them, because
it serves the build the way the launcher does.

### An import map is an inline script

The launcher's game container serves every game under a Content-Security-Policy
that allows scripts only from the game's own origin — there is no
`unsafe-inline`. That means nothing executable may live inside a bare
`<script>` body or an inline event-handler attribute, and **`<script
type="importmap">` counts**: it is inline, so the container refuses it, and
every subsequent bare-specifier import (`import * as THREE from "three"`) then
fails.

There is no external-file workaround: browsers do not load an import map from
a `src` attribute. Rewrite every bare specifier to a relative path and delete
the map:

```js
// before — resolved through an inline importmap
import * as THREE from "three"

// after — resolved directly, no importmap needed
import * as THREE from "./vendor/three.module.js"
```

A bundler does this for you. If your build ships unbundled ES modules with a
hand-written import map — common in small Three.js projects — the rewrite is
yours to do as a build step.

### A build is served one directory down

A build is served at `/g/<gameId>/<buildId>/`, never at an origin root —
`sideload dev` serves it the same way on purpose, so this surfaces before
publish. A root-absolute path like `/assets/thing.glb` resolves against the
origin root, one level above where the build lives, and 404s.

Every reference to your own assets — texture URLs, a `GLTFLoader` base path,
audio `src`, a `fetch()` for a manifest — must be relative to the document
(`./assets/thing.glb`, or built from `import.meta.url`), never rooted with a
leading `/`. That applies to paths in `index.html` and to paths built at
runtime alike.

### No inline event handler attributes

`onclick="restart()"`, `onerror="..."` and the rest are inline scripts for the
same reason — refused, not downgraded. Attach the listener from an external
script:

```html
<!-- refused: inline handler -->
<button onclick="restart()">Retry</button>

<!-- works -->
<button id="retry">Retry</button>
<script src="./boot.js"></script>
```

```js
// boot.js
document.getElementById('retry').addEventListener('click', restart)
```

## Checking your work

```sh
npx sideloadplay validate <dir>
```

This runs every check `publish` would run — the manifest against its schema,
every asset's presence, size, and dimensions, the screenshot count — with no
token, no network access, and nothing uploaded. It is the same verdict
`publish` will reach for the same directory, so there's no reason not to run
it repeatedly while you fix things.

Each failure prints as one line, `<id>: expected <expected>, actual <actual>`,
and the same data is available as JSON with `--json` — an array of
`{ id, actual, expected }` objects, meant for a script or an agent to act on
directly rather than parse out of prose. When everything passes, it also
previews what `publish` would do next: submit for review, or create a draft
and name what artwork is still missing. For example, a listing with a cover
and a hero but no screenshots yet prints:

```
All checks passed. publish would create this listing as a draft. Missing: screenshots (have 0, need >=3).
```

The failure ids you'll actually meet, and what each means, are in
`references/validation-failures.md`. Do not guess at an id that isn't listed
there — every id `validate` can emit comes straight from the checks described
in this skill.

**Never tell a developer their listing will pass without having run `validate`
yourself and watched it pass.** A listing that "should be fine" based on
reading `sideload.json` is not verified — dimensions, byte sizes, and schema
shape are exactly the kind of thing that's wrong by a pixel or a missing file
even when everything reads correctly.

## Never

- Never handle or ask for the developer's invite token. `sideload login` reads
  it from standard input and nothing else — not this skill, not a flag, not an
  environment variable — should ever touch it.
- Never invent a value for a field the developer hasn't stated: a playtime, a
  content rating, a tag chosen because it sounds close enough.
- Never claim a listing will pass, or is ready to publish, without having run
  `sideload validate` and seen it report no failures.

## References

- `references/validation-failures.md` — every failure id `sideload validate`
  can report, and what each one means.
