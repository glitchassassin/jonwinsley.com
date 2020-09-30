---
layout:     post
title:      "Screeps #6: Remote Mining Hurdles"
date:       2020-09-27 12:00:00
author:     Jon Winsley
comments:   true
summary:    Finding a better way to track and prioritize work as we expand into other rooms
categories: screeps
---

The article below describes the AI in its current state: I'm still expanding and refining my codebase. [Here's the GitHub repo](https://github.com/glitchassassin/screeps) if you'd like to follow along.

# No Accounting for Taste

A great deal of my time fixing bugs lately has involved rebalancing priorities. How many haulers should be spawned? Do we need a standing army of builders if we have towers to handle repairs? Should we dump energy into the Room Controller, Storage, or both?

All of these priorities are currently handled by the different Managers. I'd like to centralize these prioritizations by Office instead. Managers should prioritize within their concerns - the ConstructionManager should decide which construction sites need built first - but the Office should decide how much priority to allocate to construction (and builder minions). This changes over time, and depending on the circumstances.

* At RCL1 our energy budget should be devoted roughly evenly to harvesting and upgrading.
* At RCL2 we want to shift most (all?) of our upgrading budget to construction, until we get our Mines, Upgrader Depot, and Extensions built, both at our Office and in surrounding Territories.
* At RCL2 with containers built, our harvesting budget should be relatively constant, and we can shift more into upgrading, leaving the ConstructionManager enough to keep building roads.
* At RCL3 we initially shift back, leaving a minimum budget for upgrading while we set up our new infrastructure (extensions, a tower.)
* At RCL3 with extensions and tower built, we shift back to upgrading, leaving enough in Construction for roads (if needed).
* At RCL4 we have more infrastructure (extensions, storage)

If we should lose infrastructure for some reason (hostile action?), of course, the Construction budget would need to be increased until that was remedied.

## Implementation Details

At present, the main barrier is that we don't have limits on requests. Our managers are only changing the priority, but they're filing requests each tick (in most cases) for the maximum amount needed. To control this better, we need to dictate the timing of requests, the maximum amount, and the priority.

This way, during the RCL2 pre-construction phase, we can specify that the controller should still get some energy periodically (to prevent it from downgrading), but we don't need to dedicate a whole minion to the task.

For now, we'll specify the following Modes for a Manager:

* OFFLINE
* MINIMAL
* NORMAL
* PRIORITY

We'll let each Manager determine the specifics of what this means.

# Three Days Later...

Expanding beyond my initial room has proven to be a more significant hurdle than expected. A few of the issues I ran into:

* I had started caching paths for TravelTasks in Memory with `RoomPosition.findPathTo`. These paths end abruptly at the end of the room, so I switched to PathFinder instead for inter-room travel.
* While I was at it, I shifted all of my architecture away from loading Memory each tick, and instead am only loading from Memory after a code push or global reset. From then on data is saved to Memory, but read from the heap.
* When my creeps aren't in a room, I lose visibility to Sources, ConstructionSites, and unowned structures like Containers. I had to start caching these as well.

This led to some further refactoring, along the lines of the previous post, 