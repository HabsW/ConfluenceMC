# Tests

Skip empty sections in the outer random-tick loop. Numbers are isolated `optimiseRandomTick` time, not whole-server MSPT.

## Machine

AMD Ryzen 9 9950X3D, Windows 11, Eclipse Adoptium JDK 25.0.4, `-Xms4G -Xmx4G`. Paper `c9e894d3cc03f21f80de4f4061a795e11941e89a` (Minecraft 26.2). 2026-08-17.

`random_tick_speed=3`, seed `125125`, 1024 forceloaded chunks (`-16..15` squared). Drop the first 100 samples at 1024 chunks; median of the next 300.

## Isolated random-tick phase

Same world save, two paperclip jars from the same Paper commit.

| World | Unpatched | Patched |
|---|---:|---:|
| Normal overworld | 613 µs | 614 µs |
| Superflat | 422 µs | 387 µs |
| Skyblock (void + 5x5 grass) | 231 µs | 121 µs |
| The End (main island) | 234 µs | 115 µs |

Overworld is unchanged. The skip shows up when most sections have `tickingBlockCount == 0`.

## Equivalence

Compact walk vs current Moonrise linear walk (section order, RNG count, packed hits). Includes empty/sparse/dense chunks, load bind, 0/non-zero updates, and `getSections()[i] = newSection` (dirty on getSections, rebind on the next random tick).

```text
67 cases, 0 failed
```

Empty sections were never in the random-tick RNG stream.

## FAWE / WorldEdit

Block sets through `setBlockState` update the mask live. Replacing a section object via `getSections()[i] = newSection` marks that chunk dirty; the next random tick rebinds it. Writes that bypass `setBlockState` / `recalcBlockCounts` already break Moonrise's ticking list; this change does not claim more than that.
