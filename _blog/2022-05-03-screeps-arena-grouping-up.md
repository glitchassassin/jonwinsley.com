---
layout:     post
title:      "Screeps #25: Arena - Grouping Up"
date:       2022-05-03 20:30:00
author:     Jon Winsley
comments:   true
summary:    
categories: screeps
series: screeps
---

# Unit Coordination

As discussed [last time](https://www.jonwinsley.com/screeps/2022/04/18/screeps-arena-pressing-attack/), one of our major weaknesses has been creep coordination. Without more specific directions, creeps tend to get spread out, and have trouble stacking their healing/damage on the same target. So what can we do to keep our creeps close together?

In World, a common pattern is quads: four creeps moving in lockstep. I've seen this a couple times in Arena, but quads can be overwhelmed by a larger horde of minions working together. So, we'll focus our effort on the Horde.

## For the Horde

Let's consider traffic management from the beginning.

The naïve/default approach is to just path around creeps with a cost matrix. This works all right when creeps are stationary, but causes collisions when two creeps want to move to the same square. When pathing as a group, this tends to result in the group spreading out, which is a less-than-ideal formation.

We can keep the group together a little bit more by pathing as a group: calculating the group's path from the center minion, then having all minions moveTo a point on that path just outside of their group. This keeps them from splitting up and taking different paths across the map, but they still end up colliding and spreading out as they follow that path.

A simple improvement is to path around intended movement squares. I gave the cost matrix a +3 cost to discourage other minions from pathing through squares that another minion intends to travel to this tick. This yields better cohesion, with the best results if minion paths are calculated in order of distance from the target. But it's not perfect; minions still tend to get spread out into a line as the group travels.

This spread can be mitigated by having the frontmost minions stop and wait for the others to catch up. The basic algorithm: if the creep's distance to target is less than the group's distance to target, it's in front; if the creep is in front and more than `n` distance from the group's center, wait for the group to catch up. (`n` is calculated based on the total number of minions in the group.)

This keeps the group closer together, but can still run into traffic jams: if one creep decides it's close enough to the target and doesn't need to move, minions behind it can get stuck (see illustration from test vs System user). We need shoving logic to bump non-moving minions to a square that is still in range of their goal.

![Stuck creeps](/assets/screeps-stuck.png)

### Pardon Me

I had previously been using a lightly modified version of [Robalian's ExcuseMe snippet](https://github.com/screepers/screeps-snippets/blob/master/src/prototypes/TypeScript/excuseMe.ts) for this purpose in World. I wasn't entirely happy with how it was working in Arena, so I decided to rethink and rewrite that shoving logic.

I liked the idea of modifying the prototype so I can continue using `Creep.moveTo` seamlessly. So, here's the new moveTo logic:

```plaintext
Given a creep, a target RoomPosition, and a range:
  Cache the move goal so we can look it up by creep later.
  If the creep's moveGoal doesn't allow it to move: 
    Mark the square as impassible in the pathing cost matrix; return.
  If the creep's moveGoal is met:
    We don't need to move; return.
  Calculate the path from the creep to its target. 
  Give the path's next square +3 cost in the pathing cost matrix (to discourage other minions).
  If there's a creep in the next square already, shove it.
  Finally, move to the next square in the path.
```

Most of this is pretty straightforward. Shoving gets just a little complicated. First, when a minion is shoved, we'll add it to a list; at the end of the tick, when all the planned movement is done, we'll call a `handleShoves()` function in our main loop that will deal with those minions. Some may already have decided to move somewhere else, so we don't need to shove them after all. For the rest:

```plaintext
While there are minions in our shove queue that aren't planning to move:
  Get the next minion.
  Find squares it can move to.
  Exclude any squares a minion is already planning to move into, or squares that take this minion outside its move goal.
  Pick the best possible square, preferring squares with a low terrain cost (not swamp) and no creeps that would also need to be shoved.
  If one is found:
    Move this creep to that square (caching its moveGoal).
    Add this creep to the handledShoves queue.
    If there is a creep already in that square:
      Add the blocking creep to the shove queue.
    Else:
      Clear the handledShoves queue (we've reached the end of this shove chain)
  If no possible square is found:
    Mark this creep's square as impassible in the pathing cost matrix
    Get the last creep from the handledShoves queue, if it exists
    Delete that creep's cached move and put it back on top of the shoves queue to try again
Finally, reset our shove queue for the next tick.
```

### A Charismatic Leader

One final improvement to the horde logic above: designating a leader. We want our ATTACK minions at the front, so we'll set one as the leader of the group. It will set the main path for the group; if the group's centroid (the average of all the other minions' positions) is lagging behind, it will slow down to let them catch up. The other ATTACK minion will always try to move to the leader, pushing its way through the other minions if necessary. And, finally, the rest of the horde will follow the stop-and-wait approach to staying together: if a minion is closer to the leader than to the center of the horde, it will stop and wait for the rest to catch up.

This final product does a good job of keeping the minions clustered together and keeping the attackers in front. This formation is fine for traveling across the map. Once we actually get in range of the enemy, group pathing is no longer necessary; we'll switch to a different formation to coordinate our creeps in combat.

## Squads

Each type of minion - ATTACK, RANGED_ATTACK, and HEAL - will have a different positioning tactic in the fight.

ATTACK will try to move between the enemy and our flag. When pathing, it will prefer squares with more healing and less incoming damage.

RANGED_ATTACK will try to stay at a range of three squares from the enemy, kiting (moving away) if they get too close.

HEAL will try to move close to minions that need healing, preferring squares with less incoming damage.

Each of these tactics will have its own cost matrix. This also means that our shove logic needs to take each minion's cost matrix into account. We're already caching this as part of the moveGoal, if a custom cost matrix is provided, so it's easy to adjust our shove square scoring algorithm to refer to the minion's cost matrix.

Finally, we'll use a simple algorithm to prioritize damage: the main target will be the enemy closest to the group's centroid. If an attacker is not in range of that enemy, it picks a secondary target - preferably one someone else is already attacking, but otherwise something close by.

Healing will be similar - for each damaged minion, we'll allocate enough healers to bring it back up to full, moving on to the next damaged minion if we still have healers left to work with.

# Conclusions

This tactical logic is fairly simple and, as I test it, I already see some room for improvements. But it's also a significant step forward: in one test scenario, these changes took us from a complete wipe to bringing all of our minions through with no losses.

After restoring the parts-harvesting code and running a series of ranking games, we bumped up from 1800 to 1950. This is not as much of an improvement as I'd hoped, but I do see more opportunities for fine-tuning, so let's see how far we can go!