# Sideload

A desktop launcher for web games. Three.js, React Three Fiber, Babylon,
PlayCanvas: things that already run in a browser, played with a controller
instead of a mouse and a tab. Everything in the catalog is free. There is no
purchase flow in this alpha.

Playing needs nothing but the launcher. Publishing needs an invite token, and
those are issued by hand to developers we approach directly. There is no
sign-up. If you don't have one, the rest of this still tells you what
publishing involves.

This repository holds the public parts: the launcher's releases, the listing
skill, and this file. The launcher and API source are private for now, because
Sideload runs other people's game code and that has not had the review it
would need before opening up.

## Installing the launcher

Download the latest `.dmg` from
[Releases](https://github.com/DRNKNDev/SideloadPlay/releases). Each release
has notes saying what changed. macOS on Apple Silicon only, for now.

Builds are signed and notarized, so Gatekeeper opens them without complaining.
After that the launcher keeps itself up to date: a new version downloads in the
background and is applied when you next quit, so you should not need this page
again.

## Publishing a game

Publishing runs through a CLI. There is nothing in this repository to build or
run.

```sh
npx sideloadplay --help
```

That is the whole install. Nothing to add globally, nothing to configure. The
command reference, the `sideload.json` shape, and the artwork requirements are
on [npm](https://www.npmjs.com/package/sideloadplay), and this file does not
repeat them.

## Preparing a listing with an agent

```sh
npx skills add DRNKNDev/SideloadPlay
```

That installs [`skills/sideloadplay/`](./skills/sideloadplay/SKILL.md), a skill
for coding agents that follow the Agent Skills format. Ask an agent to prepare
a listing and it can write `sideload.json`, produce the artwork that
references, and run `sideload validate` until the listing passes. It teaches
the rules up front rather than letting you find them one failed check at a
time.

It prepares a listing and stops there. It does not sign in, does not publish,
and never asks for your invite token. Only `sideload login` touches that, and
only by reading it from standard input.

## Reporting a problem

Open an [issue](https://github.com/DRNKNDev/SideloadPlay/issues). A launcher
crash, a failed publish, a listing that renders wrong: all of it belongs here,
even though the code at fault usually lives in a repository you cannot see.
Its issues have nowhere else public to go.
