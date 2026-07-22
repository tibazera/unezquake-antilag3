# Antilag 3

Antilag 3 is a target-side extension to Dusty-QW's existing antilag 1
(projectile catch-up). It changes **nothing** about rocket/nail physics,
collision, or timing — it is purely a rendering addition on top of the
existing server-authoritative catch-up.

## Background

Antilag 1 advances a fired rocket's spawn position by up to
`ANTILAG_REWIND_MAXPROJECTILE` (80ms / 80 units at 1000u/s) to compensate for
the shooter's ping (`antilag_lagmove_all_proj()` / `_bounce()` in KTX's
`src/antilag.c`). Low-ping players have reported that this reads as an
unearned advantage for high-ping shooters in close combat: the rocket
"appears" already advanced, with no visible cause on the target's screen.

Dusty-QW's `unezQuake` fork already ships a hardcoded client-side CSQC layer
("ezcsqc", `src/cl_ezcsqc.c`, based on `QW-Group/ezquake-source#1010`) that
renders the shooter's own rocket immediately, before server confirmation.
That layer only benefits the shooter — it does nothing for the target.

Antilag 3 reuses that same infrastructure to also give the **target**
something to see: a brief ghost trail showing where the rocket would have
spawned with zero catch-up, so the sudden advance has a visible, legible
cause instead of looking like nothing happened.

## What changed

**Server (`ktx-antilag3`, `src/weapons.c` + `include/progs.h`):**
- New additive wire flag `PROJECTILE_CATCHUP` (bit 5) in the EZCSQC
  projectile payload (`SendEntity_Projectile()`).
- Only sent when `self->s.v.armorvalue > 0` (the antilag catch-up in
  seconds, already computed and stored by `antilag_lagmove_all_proj()`/
  `_bounce()` — reused, not recomputed).
- Quantized to a single byte, ceiling `ANTILAG3_CATCHUP_QUANT_CEILING`
  (0.100s, intentionally above the 0.080s cap for headroom).
- No existing field, byte order, or sendflag changes — legacy clients that
  don't know this bit simply never see it set (protocol is a bitmask by
  design).

**Client (`unezquake-antilag3`, `src/cl_ezcsqc.c` + `src/ezcsqc.h` +
`src/cl_main.c`):**
- Parses `PROJECTILE_CATCHUP` into `ezcsqc_entity_t.catchup_ms`.
- On a projectile's first predraw, if `catchup_ms >= ANTILAG3_GHOST_MIN_MS`
  (8ms) and `cl_antilag3_ghost` is enabled, draws a one-shot `RAIL_TRAIL2`
  ghost from `trail_origin` (the true, non-antilagged spawn point — already
  carried by the existing `PROJECTILE_SPAWN_ORIGIN` field) to `s_origin`
  (the actual, catch-up-advanced spawn point). Both points are already
  authoritative server data; nothing is inferred or guessed client-side.
- New cvar `cl_antilag3_ghost` (default `1`). Set to `0` to disable the
  ghost and fall back to plain antilag 1 rendering — this is the intended
  A/B toggle for testing.
- The ghost renders identically for every client that sees the projectile
  (shooter and target both run the same code path); there is no
  server-side shooter/target distinction in this protocol.

**mvdsv (`mvdsv-antilag3`):** unmodified. It only negotiates/transports the
CSQC extension (`MVD_PEXT1_EZCSQC`) as an opaque byte stream — the payload
contents are entirely defined by KTX's QC/C code, so no engine-level change
was needed.

## Testing antilag 1 vs antilag 3

`sv_antilag` now has a real fourth value on the server:

```
sv_antilag 0   // disabled
sv_antilag 1   // classic antilag 1 (byte-for-byte upstream behavior)
sv_antilag 2   // MVDSV-side antilag (unchanged)
sv_antilag 3   // Antilag 3: identical physics/collision to mode 1,
               // plus PROJECTILE_CATCHUP advertised to clients
```

`ANTILAG_ENABLED()` (`include/progs.h`) treats `1` and `3` identically for
every rewind/collision/damage check in `antilag.c` and `weapons.c` — hit
outcomes are byte-for-byte the same in both modes. The *only* difference is
in `SendEntity_Projectile()`: catch-up data is only put on the wire when
`sv_antilag` is exactly `3`, arming the client-side ghost.

Client-side, `cl_antilag3_ghost` (default `1`) can still disable the ghost
render even when the server is running mode 3, for per-player preference —
but the A/B comparison itself is meant to be done by switching the server:

```
rcon sv_antilag 1   // baseline: classic antilag 1
rcon sv_antilag 3   // Antilag 3: same hits, ghost rendered to clients
```

This lets an operator run the exact same match/demo twice, changing only
the server cvar, to compare perceived fairness directly — no second server
build, no protocol renegotiation, no client-side-only toggle required. The
player-facing `.antilag` vote command still only cycles `0->1->2->0`
(unchanged) — `sv_antilag 3` is operator-selected only (rcon/config), since
it is a testing/comparison mode, not a player-facing option.

## What this does NOT do

- Does not change catch-up magnitude, the 80ms cap, or rocket collision in
  any way. A hit that lands today lands identically with Antilag 3.
- Does not fix the projectile catch-up numbers for low-ping players by
  itself. It is explicitly a perception-layer fix, meant to be evaluated
  independently from (and potentially alongside) a numeric mitigation such
  as scaling catch-up by a fraction of ping before the clamp.
- Does not touch antilag 2 or `sv_antilag 0/2` behavior at all.

## Verification performed

- `ktx-antilag3`: `./build_cmake.sh linux-amd64` — builds clean, zero
  warnings in the modified files (`src/weapons.c`, `include/progs.h`).
- `unezquake-antilag3`: `./build-linux.sh` — builds clean, zero warnings in
  the modified files (`src/cl_ezcsqc.c`, `src/cl_main.c`, `src/ezcsqc.h`).
- `mvdsv-antilag3`: `./build_cmake.sh linux-amd64` — builds clean
  (unmodified from `dusty-qw/mvdsv antilag-new`).

Not yet done: in-game validation of the ghost trail's visual quality/timing,
and the actual player test battery described in
`quakeworld-antilag-projectile-test-plan.md` (server pings, distances,
scenarios, sv_antilag 0/1/2 comparison, 500 reps/combination).
