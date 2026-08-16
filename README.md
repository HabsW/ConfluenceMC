# ConfluenceMC

ConfluenceMC is a high-performance, Paper-compatible Minecraft server engine built to optimize CPU execution across both single-threaded tick loops and multi-threaded system environments.

The project addresses traditional bottlenecking in server execution by combining data-oriented serial optimizations with parallel execution pipelines, without sacrificing the ecosystem compatibility required by production networks.

---

## Core Objectives

* **Full API & Plugin Compatibility**  
  Maintains complete parity with Bukkit, Spigot, and Paper APIs. Standard plugins run directly without custom builds or breaking API behaviors.

* **Vanilla & Technical Mechanics Integrity**  
  Preserves core game logic. Redstone networks, mob farms, and technical contraptions operate with identical timing and behavior to standard Paper and Vanilla.

* **Adaptive Hardware Scaling**  
  * **Serial Single-Core Performance:** Applies sparse data structures and reduced loop overhead to accelerate execution on high-frequency, single-threaded setups.  
  * **Multi-Core Pipeline:** Distributes isolated world tasks and background processes across multi-core server processors.

---

## Upstream Contributions

Selected serial optimizations engineered for ConfluenceMC are extracted and contributed upstream to projects such as Paper and Moonrise prior to full engine releases.

* **Sparse Section Bitmasking (SSTI)**  
  Per-chunk bitset of sections with randomly ticking blocks. Sparse chunks skip empty sections; dense chunks keep the linear scan. Same RNG and section order. The mask is rebuilt on chunk load and when section objects are replaced.

---

## Legal & Disclaimers

ConfluenceMC is an independent open-source software project licensed under the [GNU General Public License v3.0](LICENSE).

*Disclaimer: ConfluenceMC is an independent open-source project and is not affiliated with, sponsored by, or endorsed by Atlassian Pty Ltd, Mojang AB, or Microsoft Corporation. The name "ConfluenceMC" (and "Confluence Engine") refers strictly to this software architecture and shares no connection with Atlassian Confluence or Mojang AB.*
