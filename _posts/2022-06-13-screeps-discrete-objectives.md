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

- For each priority level in the queue, while we have available spawns:
    - If we have a scheduled minion this tick, and it is the highest priority, spawn it.
    - If we have a scheduled minion this tick, and something else is the highest priority, the scheduled minion's schedule is revoked and it gets spawned at the next opportunity.
    - If we have no scheduled minions this tick:
        - Calculate the ticks until the next scheduled spawn with the same or higher priority (`t`)
        - Find the combination of minions that we can spawn in `t` ticks that will use the most spawn time. Pick one and begin spawning.

Calculating the ticks until the next scheduled spawn is more difficult than it seems when you can have multiple spawns. Consider:

1. Spawn1 is currently spawning a minion and has 12 ticks left
2. Spawn2 is available, and has a minion scheduled to begin spawning in 15 ticks
3. Spawn3 is available
4. There is another minion scheduled to begin spawning (on any spawn) in 15 ticks
5. There are three minions ready to spawn that will each take 24 ticks.

No minion will start on Spawn1, because it's currently spawning. No minion will start on Spawn2 because it has a minion scheduled. The scheduled minion for #4 should be allocated to Spawn1, so that we can begin spawning one of the other three minions.

We'll use the following rules to calculate next scheduled minion:

1. For each spawn, if the spawn has minion(s) scheduled for it specifically, set the earliest as its next scheduled minion.
2. For each spawn, while there are earlier scheduled minion(s) that do not conflict either with a currently scheduled/spawning minion or the next scheduled minion, set the *latest* as its next scheduled minion.


NOTE: Let's simplify this: assume all scheduled minions also specify a spawn. This will almost always be the case anyway.