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
interface PreferredSpawnData {
    spawn: Id<StructureSpawn>,
    directions?: DirectionConstant[],
}
interface SpawnOrder {
    startTime?: number,
    duration: number,
    priority: number,
    spawn?: PreferredSpawnData,
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
        - If we can fit the current minion in before the next scheduled spawn, do so.

We'll simplify this a little bit by requiring scheduled orders to also specify a spawn. This will generally be the case anyway, and makes it easier to calculate the ticks to next scheduled spawn.

There's room for some more optimization here, but we'll save that for future us!

## Creating Missions

We'll call these discrete objectives Missions, to avoid conflict with the old concept of an Objective.

```typescript
export enum MissionStatus {
  PENDING = 'PENDING',
  RUNNING = 'RUNNING',
  CANCELED = 'CANCELED',
  DONE = 'DONE',
}

export interface Mission<T extends MissionType, D> {
  office: string,
  priority: number,
  type: T,
  status: MissionStatus,
  creeps: Id<Creep>[],
  data: D,
  // Budgeting
  estimate: {
    cpu: number,
    energy: number,
  },
  actual: {
    cpu: number,
    energy: number,
  },
}
```

We'll use objects instead of classes to make it easier to store these Missions to Memory. This interface represents a generic kind of Mission, and each MissionType will flesh out `data` with the details it needs to direct its minions.

When the objective is created, we'll fill in the estimated CPU and energy needed to complete the objective. As the objective runs, the actual amounts will be updated, and when complete we'll report on the estimated vs. actual budget. This will help us improve the quality of our estimates, and perhaps cancel an objective that is taking significantly more than estimated.

```typescript
interface HarvestMissionData {
  source: Id<Source>,
}
type HarvestMission = Mission<MissionType.HARVEST, HarvestMissionData>;

export function createHarvestMission(office: string, source: Id<Source>): HarvestMission {
  // Assume working full time
  const estimate = {
    cpu: CREEP_LIFE_TIME * 0.4,
    energy: minionCost(MinionBuilders[MinionTypes.SALESMAN](spawnEnergyAvailable(office)))
  }
  return createMission({ // Helper function that fills in default values
    office,
    priority: 10,
    type: MissionType.HARVEST,
    data: {
      source,
    },
    estimate,
  })
}
```

## Mission Control

We need a process to analyze the current world state and create Missions as needed. Missions are associated with an office, but this process need not be. It might dispatch missions to multiple offices (to coordinate a claim on a new room, for example). We'll call this Mission Control.
 
```typescript
declare global {
  interface OfficeMemory {
      pendingMissions: Mission<MissionType, unknown>[],
      activeMissions: Mission<MissionType, unknown>[],
  }
}

export function runMissionControl() {
  for (const office in Memory.offices) {
    // Create new harvest mission for source, if it doesn't exist
    for (const source of sourceIds(office)) {
      if (![
        ...Memory.offices[office].activeMissions,
        ...Memory.offices[office].pendingMissions
      ].some(m => 
        m.type === MissionType.HARVEST && 
        (m as HarvestMission).data.source === source
      )) {
        Memory.offices[office].pendingMissions.push(
          createHarvestMission(office, source)
        )
      }
    }
  }
}
```

We'll probably abstract the logic for these different types of missions into handlers later. For now, this example gives us a place to start. Here we're just checking to see if we already have a mission (scheduled or pending) for each source in the office and, if not, creating one.

Now that we have missions pending, we need to start them. The first step is to calculate how much bucket we have left (cpu or energy) for each office. Energy is easy; we can just look to room storage. But CPU is a codebase-level resource. We could track individual buckets for each office, but for now it'll be easiest to just split the bucket each tick, so each office gets 1/`n` (where `n` is the number of offices).

```typescript
  // Calculate already-allocated resources
  let cpuPerOffice = Game.cpu.bucket / Object.keys(Memory.offices).length;
  for (const office in Memory.offices) {
    const remainingCpu = Memory.offices[office].activeMissions
      .reduce((remaining, mission) => 
        remaining = (mission.estimate.energy - mission.actual.energy), 
        cpuPerOffice
      );
    const remainingEnergy = Memory.offices[office].activeMissions
      .reduce((remaining, mission) => 
        remaining = (mission.estimate.energy - mission.actual.energy), 
        storageEnergyAvailable(office)
      )
    // Start missions here
  }
```

Now, we want to start missions by priority, as we have resources available. Actually, this logic starts to look a lot like our spawn queue. And there's some overlap: we want to be able to schedule a mission's start time, so it can in turn schedule its spawn.

```typescript
// loop through priorities, highest to lowest
for (const priority of priorities) {
  if (remainingCpu <= 0 || remainingEnergy <= 0) break; // No more available resources

  const missions = Memory.offices[office].pendingMissions.filter(o => o.priority === priority);
  const sortedMissions = [
    ...missions.filter(o => o.startTime === Game.time),
    ...missions.filter(o => o.startTime === undefined)
  ];

  // Handles scheduled missions first
  while (sortedMissions.length) {
    const mission = sortedMissions.shift();
    if (!mission) break;
    const canStart = mission.estimate.cpu < remainingCpu && mission.estimate.energy < remainingEnergy;
    if (!canStart) {
      mission.startTime = undefined; // Missed start time, if defined
      continue;
    }
    // Mission can start
    Memory.offices[office].pendingMissions = Memory.offices[office].pendingMissions.filter(m => m !== mission)
    Memory.offices[office].activeMissions.push(mission);
    // Update mission status and remaining budgets
    mission.status = MissionStatus.RUNNING;
    remainingCpu -= mission.estimate.cpu;
    remainingEnergy -= mission.estimate.energy;
  }
}
```