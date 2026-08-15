# Tests and Measurements: Sparse Section Bitmasking (SSTI)

Same algorithm as the Moonrise / Paper change: skip empty sections in the outer random-tick loop. These numbers reflect isolated random-tick performance and are **not whole-server MSPT**.

## Scope

* **Primary:** In-game time spent in `optimiseRandomTick` only.
* **Correctness:** Compact walk vs. current Moonrise linear walk (48 test cases).
* **Supporting:** Walk-only microbench with inner tick execution stubbed.
* **Smoke Test:** Verified functional crop growth (wheat) under patched build.
* **Out of Scope:** Full tick MSPT, Folia multi-threading, ice/snow melting, and scheduled fluid/block ticks.

---

## Testing Environment

* **Hardware:** AMD Ryzen 9 9950X3D (32 threads), Windows 11
* **JDK:** Eclipse Adoptium JDK 25.0.4 (`-Xms4G -Xmx4G`)
* **Upstream Commit:** Paper `c9e894d3cc03f21f80de4f4061a795e11941e89a` (Minecraft 26.2, 2026-08-14)
* **Configuration:** `random_tick_speed=3`, seed `125125`, 1024 forceloaded chunks (`-16..15` squared), `online-mode=false`. Drop first 200 ticks after loading; report p50 median of subsequent ticks (up to 600).

---

## 1. Isolated Random-Tick Phase (Primary Benchmark)

Timers measured `System.nanoTime` directly around `optimiseRandomTick`, summed per server tick across two paperclip jars built from the same Paper commit.

### Results (p50, microseconds per tick)

| World Type | Unpatched | Patched | Ratio / Delta |
| :--- | :---: | :---: | :---: |
| **Normal Overworld** | 694 µs | 685 µs | ~1.00x *(no regression)* |
| **Superflat World** | 499 µs | 409 µs | **1.22x** |
| **Skyblock (Void + Grass)** | 289 µs | 141 µs | **2.05x** |
| **The End (Main Island)** | 241 µs | 52 µs | **4.67x** |

*Note: Normal overworld was re-checked across 3+3 alternating boots on the same world save. Run medians overlap (unpatched 629–738 µs, patched 635–712 µs). On normal terrain, inner block logic dominates execution time; skipping empty sections shines when `tickingBlockCount == 0` for majority chunk sections.*

### How to Reproduce

1. Clone Paper at commit `c9e894d3cc03f21f80de4f4061a795e11941e89a`. Build unpatched `paperclip` jar (`./gradlew createPaperclipJar`).
2. Apply the skip-empty-sections patch.
3. Add a temporary local telemetry timer in your build:
   * **`ServerLevel.tickChunk`** (around `optimiseRandomTick`):

     ```java
     final long t0 = System.nanoTime();
     this.optimiseRandomTick(chunk, tickSpeed);
     this.paper$rtPhaseNs += System.nanoTime() - t0;
     ```

   * **`ServerChunkCache.iterateTickingChunksFaster`**:
     Zero `paper$rtPhaseNs` prior to the chunk loop; output `gameTime,tickingChunks,rtNs` to a CSV afterwards.
4. Run both jars on the identical world save (`pause-when-empty-seconds=-1`).
5. Forceload 1024 chunks:

   ```text
   forceload add -256 -256 -1 -1
   forceload add 0 0 255 255
   forceload add -256 0 -1 255
   forceload add 0 -256 255 -1
   gamerule minecraft:random_tick_speed 3
   ```

6. **World Generation Settings:**
   * **Normal:** `level-type=minecraft:normal`
   * **Superflat:** `level-type=minecraft:flat`, `generator-settings={}`
   * **Skyblock:** Void world with a floating 5x5 grass platform (skyblock simulation):

     ```text
     level-type=minecraft:flat
     generator-settings={"biome":"minecraft:the_void","layers":[{"block":"minecraft:air","height":1}],"structures":{"structures":{}}}
     ```

     Then run:

     ```text
     fill -2 64 -2 2 64 2 minecraft:bedrock
     fill -2 65 -2 2 65 2 minecraft:grass_block
     ```

   * **The End:** Void overworld with End forceloaded (`execute in minecraft:the_end run forceload add ...`). Set overworld `random_tick_speed 0` and End `3`.

---

## 2. Equivalence Harness

Compares compact vs. linear fingerprints (section order, RNG call count, packed hit indices). Covers empty/sparse/dense chunks, custom height limits, 0 to non-zero transitions, recalculations, load bindings, same-tick cursor movement, and bulk edits.

```bash
javac --release 25 CompactRandomTickTest.java RandomTickSimulator.java RandomTickSectionMask.java FakeSection.java
java paper.rtcompact.CompactRandomTickTest
```

**Result:** 48 cases, 0 failed.

> Empty sections were never part of the random-tick RNG stream. Skipping them cannot cause RNG drift for `SimpleThreadUnsafeRandom`. Same-tick behavior matches Moonrise: the walk is a live forward scan (`from = index + 1`), not a static snapshot.

---

## 3. Crop Growth Smoke Test

Verified basic world mechanics on patched Paper (`random_tick_speed=4096`). Placed a 9x9 farmland plot with wheat (`age=0`). Wheat successfully advanced to `age=7` over time.

---

## 4. Walk-Only Microbenchmark (Supporting)

Tests algorithm overhead only (section walk & bitset maintenance with inner tick body stubbed):

* **Sparse (~4% eligible, 24 sections):** Compact p50 was ~4 ns vs. linear ~10 ns.
* **Dense (100% eligible):** Production fallback retains linear scan. Vanilla 24-section dense p50 added +2 to +5 ns per chunk walk (drowned out by inner block ticks).
* **Bitset Upkeep:** Bit flips on 0 to non-zero transitions take tens of nanoseconds in isolation—negligible cost during block mutations.

---

## Known Limits & Architectural Invariants

1. **Dense Chunks (`eligible * 2 >= section count`):** Falls back to original linear scan to avoid bitmask traversal overhead.
2. **Direct Section Swaps:** Replacing a `LevelChunkSection` object directly inside `getSections()` without rebinding the mask is unsupported.
3. **Bypassing API:** Writes that bypass `setBlockState` or `recalcBlockCounts` already break Moonrise block counts; this patch maintains those existing invariants.
