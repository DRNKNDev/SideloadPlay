---
name: sideloadplay-controller
description: Add gamepad support to a web game so it plays on Sideload, a desktop
  launcher built around a controller. Covers the pad the launcher actually hands a
  game, aiming and movement that feel right, pausing, and the submission rules a
  listing is checked against. This skill changes game code; it does not write
  sideload.json, produce artwork, or publish anything. Use when a Three.js, React
  Three Fiber, Babylon, or PlayCanvas game needs controller support, or when
  controller input behaves differently inside Sideload than in a browser.
---

## Scope

This skill adds gamepad support to a game. It does not prepare a listing: writing
`sideload.json`, producing store artwork and running `sideload validate` are the
`sideloadplay` skill's job, and publishing is the developer's own command.

When the work is done, two fields in `sideload.json` change: `controller_support`
becomes `full` or `partial`, and `input` gains `gamepad`. Point the developer at
the `sideloadplay` skill for the rest of the listing.

Do not claim `full` without having played the game with a pad inside the launcher.
That is the one thing this skill cannot verify for you.

## The pad the launcher hands you

The ordinary Gamepad API works. No SDK, no import, nothing Sideload-specific to
depend on, and a game written against it runs the same in a plain browser.

```js
const pad = navigator.getGamepads()[0]
if (pad) {
  const moveX = pad.axes[0]
  const firing = pad.buttons[7].value > 0.1   // right trigger, analogue
}
```

Four things about it that are not obvious:

**No user gesture is needed.** A browser withholds pads until the page sees a
click or a keypress. The launcher does not: a player who picks up a controller
and never touches the mouse still gets input. Do not gate your pad setup behind a
click.

**Only slot 0 is ever filled.** One pad, one player. Slots 1 to 3 are always
`null`. Do not build a local-multiplayer path expecting more.

**17 buttons and 4 axes.** Index 6 and 7 are the analogue triggers, 12 to 15 are
the d-pad, and 16 is Guide.

**Index 0 is the bottom face button, whatever is printed on it.** The launcher
remaps Nintendo-style pads so position wins over label. That is the W3C rule and
it is why button prompts must name positions rather than letters: the same
physical button is A on an Xbox pad, B on a Switch one, and Cross on a
PlayStation one.

### `gamepadconnected` is a plain Event

The launcher dispatches a plain `Event` with a `gamepad` property attached, not a
`GamepadEvent`, because that constructor rejects a non-native pad. Reading
`event.gamepad` works exactly as you expect. Checking the type does not:

```js
window.addEventListener('gamepadconnected', (event) => {
  if (event instanceof GamepadEvent) connect(event.gamepad)   // never runs
  connect(event.gamepad)                                      // correct
})
```

This fails silently and only inside the launcher, which makes it expensive to
find. Do not type-check the event.

### Buttons the launcher keeps

Three never reach a game, in any state:

| Index | Button | Used for |
|---|---|---|
| 9 | Start | opens the launcher's pause overlay |
| 10 | Left stick click | with 11, toggles the launcher's pointer mode |
| 11 | Right stick click | with 10, the same |

They read as unpressed in every snapshot, the same way Escape never reaches a
game's `keydown`. You cannot bind them, so there is nothing to avoid: this is
information for designing a control scheme, not a rule to follow.

Everything else is yours, **including Guide at index 16**. The launcher has no use
for it.

## Pausing

The launcher cannot stop a game's code, so it asks. Two listeners, no import:

```js
window.addEventListener('sideload:pause', () => { /* stop the loop, mute */ })
window.addEventListener('sideload:resume', () => { /* carry on */ })
```

Honouring both is exactly what `lifecycle: "managed"` declares in `sideload.json`.
A game that ignores them still runs and is still listed, just as `unmanaged`.

If the game has its own pause menu, suppress it while the launcher owns the
pause, or two menus stack: the launcher's overlay on top, the game's underneath,
waiting. A flag set in the pause handler and cleared in the resume handler is
enough.

## Movement and aiming

**Apply the dead zone to the vector, not to each axis.** Sticks rest slightly off
centre, and a per-axis dead zone leaves a square notch that makes diagonal
movement feel wrong.

```js
const mag = Math.hypot(x, y)
if (mag < DEAD_ZONE) return { x: 0, y: 0 }
const scale = (mag - DEAD_ZONE) / (1 - DEAD_ZONE) / mag
return { x: x * scale, y: y * scale }
```

**Scale by delta time.** Sticks are sampled per frame and report position, not
movement, so multiplying by frame time is what keeps aiming speed the same at 60
and 144 Hz. Keyboard movement in most games already does this; match it.

**A response curve helps.** Squaring or cubing the magnitude, sign preserved,
gives precise aim at small deflections and fast turns at full push. Linear
aiming feels twitchy near centre and slow at the edge.

**Triggers are analogue.** `buttons[6]` and `buttons[7]` carry travel in `.value`.
`.pressed` is derived at half travel, which is fine for a binary action and
wasteful for a throttle or a bow draw.

## Aiming without pointer lock

This is the one most often got wrong.

Mouse look needs pointer lock, and pointer lock needs a click. A controller player
may never click at all, so **stick aiming must work with the pointer unlocked**,
and picking up a pad must not silently request a lock. Requesting one outside a
user gesture is rejected by the browser anyway.

Keep the mouse working too. A player using both should not have to tell the game
which they mean, and a pad connecting should not disable the keyboard.

## What a listing is checked against

From the submission checklist, all of it verifiable by playing:

- Still playable with keyboard and mouse.
- Gamepad input verified working **inside the launcher**, not only in a browser.
- Survives a disconnect without crashing. Clear held input when the pad goes away,
  or the player walks forever.
- Never uses the gamepad `index` as a persistent player identity. Indices are
  reused on reconnect.
- Pointer lock requested only on a user gesture, released on Escape and on blur,
  and handles rejection.
- Fullscreen requested only on a user gesture.
- On-screen prompts name button **positions**, not letters. If naming positions is
  too much work, show no prompts rather than wrong ones.

## Never

- Never claim `controller_support: "full"` without playing the game with a pad
  inside the launcher. A browser test does not cover the synthesised pad, the reserved
  buttons, or the pause events.
- Never add a gamepad library. The Gamepad API gives you everything here, and a
  dependency ships to every player.
- Never disable the keyboard or mouse when a pad connects.
- Never bind an action to a button the launcher keeps. It will never fire.
