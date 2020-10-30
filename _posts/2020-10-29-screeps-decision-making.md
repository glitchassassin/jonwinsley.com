---
layout:     post
title:      "Screeps #14: Decision Making"
date:       2020-10-29 15:45:00
author:     Jon Winsley
comments:   true
summary:    
categories: screeps
series: screeps
---

The article below describes the AI in its current state: I'm still expanding and refining my codebase. [Here's the GitHub repo](https://github.com/glitchassassin/screeps) if you'd like to follow along.

# Decision Making

Let's talk briefly about the current state. It has grown up somewhat organically as our requirements have evolved.

At a low level, TaskActions (fulfilled by minions) mostly operate as State Machines. The creep switches from the "Getting Energy" state to the "Working" state, much like in the tutorial. For some simpler actions, such as harvesting, there is only one state: doing work. Movement is part of those states (where needed), not its own state.

At a higher level, the Managers maintain a Priority Queue of tasks. The task requestors have a set of rules to define the priority for their tasks, and then the Managers fulfill them based on that relative priority.

The Office adjusts the relative emphasis of the Managers by setting their status to DISABLED, MINIMAL, NORMAL, or PRIORITY. This allows us to focus on some managers at the expense of others (when booting up a new Office, for example, Defense is often less of a priority than getting the energy pipeline flowing).

## Reflection

What's working well: the segmented units of TaskActions nicely encapsulate the reusable aspects of traveling, getting energy, etc. A state machine is a simple and logical way to handle the steps of a well-known task.

What's not: The Priority system is not translating to a clear, unified game plan. Each Manager is acting on its own initiative, rather than according to broader strategic interests. It can be difficult to get insight into why we're spawning a Lawyer to reserve a controller when we really need some Guards to clear the room first.

I'd like to see the system be more configurable. For example, I'm considering a test of abandoning remote mining on shard3 to reduce the number of active minions. But we don't currently have a RemoteMining strategy that we can enable or disable: it's just hardwired into the SalesManager.

Let's see if we can come up with a better system.

# Objectives

To improve our development and testing process, we should be able to isolate and modify or replace individual tactics or strategies. We should be able to enable or disable RemoteMining (for example).

A Strategy might affect multiple Managers: RemoteMining involves defending a room with Guards, reserving the Controller with Lawyers, building Containers with Engineers, mining the Sources with Salesmen, and hauling the energy back to the home Office with Carriers.

On a lower level, I think our state machine-based task actions are a good solution, but could be made more atomic to reduce boilerplate.

## Abstracting Tasks

The more we can abstract the implementation details from the strategy logic, the easier it will be to think through the design of our AI. For example, this would be a concise and clear way of expressing a task implementation (pseudocode, not real JS):

```
upgradeTask = Sequence(
    Selector(
        getEnergy(fromAssignedDepot),
        getEnergy(fromClosestStash)
    ),
    markController(target, 'Property of the Grey Company'),
    upgradeController(target)
)
```

A "Sequence" here represents a series of tasks to be followed one after the other, and a "Selector" represents a series of tasks to be tried until one succeeds. (I'm indebted to Millington and Funge's Artificial Intelligence for Games for the terms and concepts.) So the upgradeTask will get energy from its assigned depot, or, if that fails, from the closest supply of available energy. Then it will mark the controller (or skip the step, if that's already done), and finally use the energy it has gathered to upgrade the controller.

These steps can be broken down further:

```
moveAndMarkController = Selector(
    markController(target, 'Property of the Grey Company'),
    moveTo(target)
)
```

This will allow us to easily compose task logic in a way that's easy to interpret at a glance.

## Abstracting Office Strategies

It's comparatively easy to manage tasks for individual minions. Let's skip past the Manager level for the moment and talk about broader office strategies; then, we'll talk about how to use Managers to delegate those strategies as minion tasks. We'll start by throwing out a handful of broad strategies and try to draw some connections between them.

- Drop mining: Salesmen drop harvested resources on the ground for Carriers to collect
  - Salesmen harvest sources
  - Carriers transport resources back to main Office
- Container mining: Salesmen drop harvested resources into Container for Carriers to collect
  - Salesmen harvest sources
  - Carriers transport resources back to main Office
  - Engineers build road/container infrastructure to increase efficiency
- Snowgoose mining: Salesmen deposit harvested resources into extensions, then Spawn creates and immediately recycles a creep to transfer the resources to a central location
  - Salesmen harvest sources
  - Engineers build extensions
  - Spawn creates and recycles creeps
  - Carriers move energy from Spawn to wherever it's needed
- Remote Mining: exploiting sources in nearby rooms.
  - Salesmen harvest sources
  - Carriers transport resources back to main Office
  - Engineers build road/container infrastructure to increase efficiency
  - Lawyers reserve controller to increase capacity
  - Guards defend the room against invaders

### Mining Tasks

Let's break this down. When we are drop mining, we (currently) assign one or more dedicated Salesmen to the Source. They travel to the source and harvest, dropping any collected resources. The source is designated as a LogisticsSource, so nearby Carriers will collect resources from it to fulfill requests. When we graduate to container harvesting, the process for Salesmen and Carriers is the same, but we create a container construction site, which serves as a request for an Engineer. Remote mining adds requests for a Lawyer to reserve the controller and Guards (if hostile activity is detected) to protect the room.

Following our current paradigm, we would assign requests:

1. Harvest requests for SalesManager
2. Logistics requests (if nothing else, standing order for storage)
3. Build requests for FacilitiesManager
4. Reserve requests for LegalManager
5. Defense requests for DefenseManager

Then each of those managers would request minions from HRManager. The spawn queue is where the most difficult part of the resource contention arises.

Which should come first? Logically, we want the Salesmen to begin to harvest energy. But do we spawn all Salesmen until the requests are filled? If we're starting from scratch, or restarting from scratch, we have a steady trickle of 300 energy over 300 ticks regenerating at spawn. Without energy coming in, that's enough to maintain five 300-energy minions. If we have enough Harvest requests (perhaps from remote rooms), we'll never get any Carriers created.

