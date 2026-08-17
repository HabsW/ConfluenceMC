# Tests

Skip empty sections in the outer random-tick loop. Numbers are isolated `optimiseRandomTick` time, not whole-server MSPT.

## Machine

AMD Ryzen 9 9950X3D, Windows 11, Eclipse Adoptium JDK 25.0.4, `-Xms4G -Xmx4G`. Paper `c9e894d3cc03f21f80de4f4061a795e11941e89a` (Minecraft 26.2). 2026-08-17. The Java process was not pinned to a CCD.

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

## Reproduce

1. Clone Paper at `c9e894d3cc03f21f80de4f4061a795e11941e89a`. Build an unpatched paperclip jar (`./gradlew createPaperclipJar`).
2. Apply [`../patches/0001-Skip-empty-random-tick-sections.patch`](../patches/0001-Skip-empty-random-tick-sections.patch) and build a second paperclip jar from the same commit.
3. Add a local timer. Do not leave it in the published patch.

   In `ServerLevel.tickChunk`, around `optimiseRandomTick`:

   ```java
   final long t0 = System.nanoTime();
   this.optimiseRandomTick(chunk, tickSpeed);
   this.paper$rtPhaseNs += System.nanoTime() - t0;
   ```

   In `ServerChunkCache.iterateTickingChunksFaster`: zero `paper$rtPhaseNs` before the chunk loop. After the loop, append `gameTime,tickingChunks,rtNs` to `rt-phase.csv`.
4. Use the same world save for both jars. `pause-when-empty-seconds=-1`. Heap `-Xms4G -Xmx4G`.
5. Forceload 1024 chunks, then set tick speed:

   ```text
   forceload add -256 -256 -1 -1
   forceload add 0 0 255 255
   forceload add -256 0 -1 255
   forceload add 0 -256 255 -1
   gamerule minecraft:random_tick_speed 3
   ```

6. Worlds:

   * **Normal:** `level-type=minecraft:normal`, seed `125125`.
   * **Superflat:** `level-type=minecraft:flat`, `generator-settings={}`, seed `125125`.
   * **Skyblock:** void world, then a 5x5 grass platform:

     ```text
     level-type=minecraft:flat
     generator-settings={"biome":"minecraft:the_void","layers":[{"block":"minecraft:air","height":1}],"structures":{"structures":{}}}
     ```

     ```text
     fill -2 64 -2 2 64 2 minecraft:bedrock
     fill -2 65 -2 2 65 2 minecraft:grass_block
     ```

   * **The End:** void overworld, forceload the End with the same four `forceload add` commands (`execute in minecraft:the_end run forceload add ...`). Overworld `random_tick_speed 0`, End `3`.
7. Keep rows where `tickingChunks` is 1024. Drop the first 100 of those rows. Report the median of the next 300 (`rtNs / 1000` for microseconds).

## Equivalence

Compact walk vs current Moonrise linear walk (section order, RNG count, packed hits). Includes empty/sparse/dense chunks, load bind, 0/non-zero updates, and `getSections()[i] = newSection` (dirty on getSections, rebind on the next random tick).

```text
67 cases, 0 failed
```

Empty sections were never in the random-tick RNG stream.

## FAWE / WorldEdit

Block sets through `setBlockState` update the mask live. Replacing a section object via `getSections()[i] = newSection` marks that chunk dirty; the next random tick rebinds it. Writes that bypass `setBlockState` / `recalcBlockCounts` already break Moonrise's ticking list; this change does not claim more than that.
