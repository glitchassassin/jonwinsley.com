---
layout:     post
title:      "Screeps #12: Strategic Directives"
date:       2020-10-17 14:30:00
author:     Jon Winsley
comments:   true
summary:    
categories: screeps
series: screeps
---

The article below describes the AI in its current state: I'm still expanding and refining my codebase. [Here's the GitHub repo](https://github.com/glitchassassin/screeps) if you'd like to follow along.

# Making Bigger Decisions

The framework we've been building so far has been focused on decisions I've made, about where to remote mine, or when to defend or attack. It's time to start delegating those decisions to the Grey Company itself.

![The Grey Company neighborhood](/assets/screeps-neighborhood.png)

In Shard3 we do not have a lot of uncontested space. I am currently managing two remote territories, which are regularly encroached upon by Invader cores. Another player tried to set up a colony in one of my territories, which I rebuffed (eventually) by implementing Guards. 

But looking beyond the immediately adjacent territories, I am now at GCL2, so I need to begin scouting sites for a second Office. Or rather, the Grey Company needs to do so: and, once we have settled on a likely spot, it needs to be able to pick remote mining sites and defend them.

## Directives

We ought to consider directives from two levels, the Office and the Boardroom. We'll let the Boardroom handle global strategies such as planting new offices; the Office will solely be concerned with managing and defending its own territories.

* Our default territory directive will be IGNORE. Minions won't try to do anything special: they can traverse the territory, but won't try to set up Franchises or defend it.
* If the territory has useful resources, and is safe for our minions, we'll EXPLOIT it.
* If someone else has already laid claim to the territory, and we don't think we can defeat them, we'll AVOID the territory.
* If we *do* think we can defeat them, we'll try to ACQUIRE the territory.
* If someone else is trying to take over a territory that we are exploiting, we'll DEFEND it.

## Intelligence

The above directives are based on intelligence gathered about the territory: is it currently owned? What is the controller level? Are there hostile structures or minions? What resources are available?

Right now, the only Territories we are tracking are the ones we can exit to from the main Office room. But we need to rethink that map so we can track Intelligence more broadly. We want to be able to expand into all eight surrounding rooms (if applicable), not just the directly adjacent four or so. And, when we are scouting for a new Office site, we need to be able to track intelligence for rooms well outside the Office.

