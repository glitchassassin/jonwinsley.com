---
layout:     post
title:      "Screeps #26: World - Discrete Objectives"
date:       2022-06-13 20:30:00
author:     Jon Winsley
comments:   true
summary:    
categories: screeps
series: screeps
---

The article below describes the AI in its current state: I'm still expanding and refining my codebase. [Here's the GitHub repo](https://github.com/glitchassassin/screeps) if you'd like to follow along.

# Evolving Objectives

This post picks up where [Screeps #23: Botarena!](https://www.jonwinsley.com/screeps/2021/09/28/screeps-spawning-budget/) left off. In that installment, we implemented a budgeting system for objectives, to balance work by the critical factors of CPU, energy cost, and spawn time.

That implementation was "continuous," that is, it treated the objective as always-on with a certain average cost per tick that would scale up or down depending on resources available. There's no sense of discrete "chunks" of work that the objective accomplishes. This works fine for most of our economy objectives, like harvesting, hauling, upgrading, etc. which will (more or less) always be running. But it doesn't work as well to describe one-off tasks like "go plunder an abandoned room" or "go harvest a specific deposit".

In addition, the continuous system doesn't make good use of the CPU bucket or storage energy buffers. We have some fudging built in for storage energy - if the storage level is higher than our target, we scale up the available per-tick energy, and if it's lower we scale up logistics to haul from sources. But this assumes we're working with enough remotes that we have more franchise income than we can effectively use, and that assumption doesn't hold on shard3 (for example).

In general, the theory behind continuous budgets is sound. But it's difficult to visualize and implement these objectives. Is there a simpler approach to budgeting objectives?

# Discrete Budgets

The theory behind discrete budgets is simply this: each objective deals with a discrete task, and is estimated based on total cost (CPU and energy). Instead of scaling to income/logistics rate, we're running the objective if there's enough energy in storage (and CPU in the bucket) to pay for it. Otherwise, it goes on the back burner while the buckets fill up.

Of course, we don't have a "spawn bucket," but spawn time is still a limiting factor. So, we'll track spawn availability separately: we'll create spawn scheduling queues, and fit objectives in the queue.

## Spawn Queues

The main problem we want to solve here is scheduled minions. Certain minions want to be spawned at a specific time, to replace one that will be dying, for example. So, we want to be able to schedule those orders, and still override them (if needed) for higher-priority spawn orders.

Each Office will have up to three queues, one per available spawn.

```typescript
interface SpawnOrderData {
    name: string,
    body: BodyPartConstant[],
    memory?: CreepMemory
}
interface SpawnOrder {
    startTime?: number,
    duration: number,
    priority: number,
    data: SpawnOrderData
}
declare global {
    interface OfficeMemory {
        spawnQueues: {
            [id: string]: SpawnOrder[]
        }
    }
}
```

### Handling Conflicts

Each tick, if we are not currently spawning:

1. If we have a scheduled minion this tick, and it is the highest priority, spawn it.
2. If we have a scheduled minion this tick, and something else is the highest priority, the scheduled minion's schedule is revoked and it gets spawned at the next opportunity.
3. If we have no scheduled minions this tick:
  - For each priority in the queue:
    - calculate the ticks until the next scheduled spawn with the same or higher priority (`t`)
    - Find the combination of minions that we can spawn in `t` ticks that will use the most spawn time. Pick one and begin spawning.

When we create a scheduled spawn order, it may conflict with one or more existing orders. In this case, the lowest-priority order will be shifted forward (if there is room) or else un-scheduled (to run whenever the spawn has time).

### Handling Multi-Spawn Objectives

Let's suppose we want to spawn multiple minions: maybe we want an ATTACK/HEAL duo to crack power banks. How can we make sure they spawn together? Consider the case where we only have one spawn (they should spawn one right after the other) or where we have multiple spawns (they should prefer to spawn at the same time, or else sequentially).