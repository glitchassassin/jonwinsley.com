---
layout:     post
title:      "Screeps #10: Questioning Everything"
date:       2020-10-07 08:00:00
author:     Jon Winsley
comments:   true
summary:    In which we look back over our progress so far and evaluate what has worked and what needs improvement.
categories: screeps
thumbnail: /assets/screeps-10-8-status.png
---

The article below describes the AI in its current state: I'm still expanding and refining my codebase. [Here's the GitHub repo](https://github.com/glitchassassin/screeps) if you'd like to follow along.

# Current State

We have reached GCL2 and have one room at RCL6. The Grey Company is currently franchising in two adjacent territories. Our Salesmen are effectively tapping out the sources in all three rooms, but the energy is piling up there and not being moved anywhere useful:

![Sales report from active franchises](/assets/screeps-10-8-sales.png)

Our fleet of Carriers is averaging about 85% capacity (85% full, 15% empty). This seems high: given the simple example of one Storage and one Source, the Carriers should be averaging 50% capacity (full from Source -> Storage, empty from Storage -> Source).

Meanwhile, our overall outputs (spawning, upgrading, building, repairing) have been comparatively low, averaging 15.5 energy/tick compared to the 40 energy/tick input from all Franchises.

