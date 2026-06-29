# lumen-snake

The Snake game for **AspisOS**, a capability-based, no-ambient-authority
x86-64 operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

snake is the classic grid Snake game: a snake crawls a square board, eats food
to grow a tail, and dies on a wall or itself. It is a leaf component of the
Lumen desktop, distributed as a [herald](https://github.com/AspisOS/AspisOS)
package, and runs as an **external client** of the
[lumen](https://github.com/AspisOS/lumen) compositor — it connects to
`/run/lumen.sock` over the Lumen window protocol and is handed a shared-memory
buffer to draw into, rather than being an in-process compositor built-in.

## Where snake fits

AspisOS is decomposed into independent repositories. snake sits at the leaf of
the graphical stack:

| Repo | Role |
|------|------|
| [`AspisOS/Aegis`](https://github.com/AspisOS/Aegis) | The kernel: capability model, `AF_UNIX` sockets, `memfd`, the syscalls the desktop runs on. |
| [`AspisOS/lumen`](https://github.com/AspisOS/lumen) | The compositor / display server. Owns the framebuffer; every GUI app is one of its clients. |
| [`AspisOS/glyph`](https://github.com/AspisOS/glyph) | The GUI toolkit snake links against: the software renderer (`draw_*`, `font_*`), theme/accent values, and the client side of the Lumen protocol (`lumen_client.h`). |
| `AspisOS/lumen-snake` | **This repo.** The Snake game. |

## What it does

Grounded in `src/main.c`:

- Opens a fixed **480x520** window titled "Snake" via `lumen_window_create` and
  draws a centered **20x20** grid (`GRID`) of **22px** cells (`CELL`) into the
  shared surface, with a header showing the title and live `Score` / `Best`.
- Pure userspace, **integer-only** logic (no floating point — the Aegis FPU is
  noted as fragile). The snake is a flat `cell_t snake[GRID*GRID]` array with
  `snake[0]` the head; `step()` advances it one cell per tick by shifting the
  body back and writing a new head.
- A **timeout-driven tick loop**: `lumen_wait_event` is called with the tick as
  its timeout, so a timeout (`r == 0`) advances the game one `step()` while a
  real event (`r == 1`) feeds input without stepping. Pace starts at
  `TICK_START` (150ms) and shaves `TICK_STEP` (4ms) per food down to `TICK_MIN`
  (70ms), so the snake speeds up as the score climbs.
- **One input dispatch** (`feed_key`): arrow keys (the synthetic `KEY_UP/DOWN/
  LEFT/RIGHT` codes Lumen delivers) or WASD steer, `P` toggles pause, `R` starts
  a new game (`new_game`), and Esc / the close request quits. `set_dir` rejects
  an instant 180° reversal so you can't suicide into your own neck.
- **Food and growth**: `place_food` picks the k-th empty cell via an xorshift32
  PRNG (`rng_*`, seeded per launch from pid/time/address) to guarantee a free
  spot without retry-looping; eating grows `len`, adds 10 to the score, and
  re-places food. Filling the board is a perfect game.
- **Game over** on a wall hit or self-collision (the tail tip is allowed when
  not eating, since it vacates that cell); `draw_overlay_text` dims the board and
  centers a "Game Over" / "Paused" message. Rendering is gated on a `dirty` flag
  and pushed with `lumen_window_present`.

## Capabilities

AspisOS grants a process no ambient authority; it can touch the system only
through capabilities declared for it at exec time. snake's policy
(`pkg/etc/aegis/caps.d/snake`) is the baseline:

```
service
```

The `service` profile and **no** elevated capabilities — snake is pure compute
over a window surface and touches nothing beyond the compositor socket.

## Status

snake is intentionally small: a single, self-contained game. It is complete for
what it is — board, tail, food, scoring, pause/restart, and a speed ramp — and
honest about its scope rather than feature-padded. There is no persisted high
score (`Best` lives only for the session) and no menu beyond the in-window hint.

## Building

snake builds with a musl cross-compiler against a **pinned**
[glyph](https://github.com/AspisOS/glyph) toolkit artifact (the GUI libraries it
links), then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `make` runs `tools/fetch-glyph.sh $(GLYPH_VERSION)` to download and unpack the
  pinned toolkit into `toolkit/`, compiles `src/*.c` against it, then packs.
- `MUSL_CC` is the musl cross-compiler (defaults to `musl-gcc` on `PATH`; the
  only toolchain assumption — point it at an Aegis-native `cc` to build on-device
  in the future).
- `HERALD_KEY` is the ECDSA-P256 key that signs the `.hpkg`.
- `GLYPH_VERSION` pins the toolkit release; `VERSION` is this app's own version.

Output: `lumen-snake.hpkg` (a `class=system` herald package) +
`lumen-snake.hpkg.sig`.

## Package payload

`lumen-snake.hpkg` is a **herald `class=system` package**: a manifest-first
uncompressed POSIX `ustar` archive with a detached ECDSA-P256/SHA-256 signature
(`tools/pack.sh`). Its herald id (`lumen-snake`) deliberately differs from the
bundle/exec name (`snake`), and it installs across two trees — which is exactly
why it is `class=system` (first-party, signature-trusted, installed verbatim)
rather than an ordinary single-prefix package:

```
/apps/snake/snake            the app binary
/apps/snake/app.ini          the bundle descriptor (name=Snake, exec=snake)
/etc/aegis/caps.d/snake      its capability policy
```

## Repository layout

```
src/        snake source (main.c)
pkg/        install-tree skeleton shipped verbatim (apps bundle + caps.d)
tools/      fetch-glyph.sh (pinned toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this app's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — snake is an external client of the compositor, so installing
it pulls [lumen](https://github.com/AspisOS/lumen). lumen also ships the desktop
fonts (Inter, JetBrains Mono), so snake inherits them transitively; there is no
separate font package.
