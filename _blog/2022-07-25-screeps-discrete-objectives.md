---
layout:     post
title:      "Screeps #26: World - Discrete Objectives"
date:       2022-07-25 17:30:00
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

### Handling Conflicts

Each tick, if we are not currently spawning:

- For each priority level in the queue, while we have available spawns:
    - If we have a scheduled minion this tick, and it is the highest priority, spawn it.
    - If we have a scheduled minion this tick, and something else is the highest priority, the scheduled minion's schedule is revoked and it gets spawned at the next opportunity.
    - If we have no scheduled minions this tick:
        - Calculate the ticks until the next scheduled spawn with the same or higher priority (`t`)
        - If we can fit the current minion in before the next scheduled spawn, do so.

We'll simplify this a little bit by requiring scheduled orders to also specify a spawn. This will generally be the case anyway, and makes it easier to calculate the ticks to next scheduled spawn.

There's room for some more optimization here, but we'll save that for future us!

## Creating Missions

We'll call these discrete objectives "Missions," to avoid conflict with the old concept of an Objective.

We'll use objects instead of classes to make it easier to store these Missions to Memory. When the mission is created, we'll fill in the estimated CPU and energy needed to complete the mission. As the mission runs, the actual amounts will be updated, and when complete we'll report on the estimated vs. actual budget. This will help us improve the quality of our estimates, and perhaps cancel an mission that is taking significantly more than estimated.

Each office will run a series of processes that generates more missions as needed. We'll call these Controllers. These don't need to align 1:1 with the missions themselves; for example, the Harvest controller will generate missions for harvesting (locally and in remotes) as well as reserving remote rooms. The Defense controller will spawn different kinds of missions for defending remotes or local rooms. And more complex combat logic will also end up spawning different kinds of missions.

Each mission will have its own associated logic to create a mission (perhaps precalculating things like source distance), spawn, and run the mission.

## Mission Control

We'll maintain a separate list of pending and active missions, per office, in Memory. Each tick we'll review pending missions and, if we have budget and an available spawn, we'll start a new mission.

What is the baseline for our budgets? When we hit RCL4, we can go based on the energy in storage; it's not hard to have enough margin to accommodate the entire energy usage of several Upgraders over the next 1500 ticks. But before we have a storage, we don't have the capacity to budget over a creep's full lifetime - we can easily harvest far more than we can store.

I'm still working on a good heuristic for this stage. My current "best effort" uses a baseline of `hqContainerEnergy + roomEnergy + allHaulersUsedCapacity + Math.min(allHaulersFreeCapacity, allFranchisesHarvestedEnergy)`. This still isn't enough to balance energy input; it's only a fraction of the room's actual energy input over the new mission's lifespan. 

For now, I'm compensating by scaling down the estimate for energy used by engineers and upgraders. It's only estimating for the next `CREEP_LIFE_TIME / 5 = 300` ticks, as this is a little greater than the furthest remote round trip distance, and then updating that estimate each tick (to make sure the energy it needs is reserved). 

In theory, this means that as long as we maintain about the same level of energy income, our estimate should be close to on target. In practice, the actual energy income still fluctuates, so the estimate ends up being off, and we end up wasting resources.

Despite these issues, the system does work, and is hitting RCL4 at 16k ticks. I'm certain this can improve further if we work out the low-level balance issues.

## Other Benefits

Tracking the actual energy used per mission makes it easier to evaluate our progress: we can compare the original estimated CPU/energy with what we actually used, to improve the estimation process (or identify issues); and we can get a better sense for exactly where the energy and CPU are going. This will help with debugging and optimizing specific kinds of work.

While at it, I added an "efficiency" metric to the mission. This just tracks how often the creep is doing its primary task - building or repairing, upgrading, hauling - vs. moving around or idling. This also provides useful insight into periods where logistics minions find themselves idling - either empty, if there aren't enough harvesters, or full, if there aren't enough workers.

## Conclusions

Overall, I think the benefits are there, and will be more fully realized as I delve into more complex sorts of logic. Dispatching combat missions has become significantly easier, for example. 

Right now there are definite cycles as the bot swings from harvesting to fill the storage, to slacking off harvesting because storage has enough energy, to spawning more harvesters again once storage runs down. I haven't made up my mind yet if this is a bad thing. It does seem to hurt us at early levels, but at higher RCLs the bucket balances this enough that I *think* there's no real impact.

One barrier that we've definitely been running into through these tests has been spawn time. As we get more extensions, our refillers simply can't keep up to maintain spawn uptime. So, next, I think I'll take a break from optimizing missions and revamp room planning. Adding a fastFiller, rearranging extensions, and perhaps making a few other adjustments to make room planning a little more robust. Stay tuned!