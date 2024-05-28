---
layout:     post
title:      "Screeps #25: Arena - Pressing the Attack"
date:       2022-04-18 17:30:00
author:     Jon Winsley
comments:   true
summary:    
categories: screeps
series: screeps
---

# Hitting a Wall

If you've played Screeps: Arena for a while you'll be familiar with the sensation of hitting a wall. You make some improvements to your codebase, begin to climb the rankings, and then start to encounter codebases with a new strategy you haven't accounted for. And, sometimes, that strategy is your own.

Around 1600 rating (CTF) I started running into codebases with a very similar mindset to mine: keep the fight in range of your own towers, and if your opponent decides not to engage, a draw is as good as a win.

Until seven out of ten of your rated matches are draws, because everyone else is thinking the same thing, and then it's just excruciatingly boring.

Fortunately, as is often the case, the key to leveling up lies in our opponents' strategies. Even playing defensively, there are codebases that are beating us. These are, mostly, taking advantage of the River.

## Playing Frankenstein

In the River of swamp through the center of the CTF map, new body parts (`MOVE`, `ATTACK`, etc.) spawn randomly and are picked up automatically by any minion that steps on one. These extra parts can shift the balance of power - even, potentially, outweighing the benefit of fighting in range of your own towers.

And if they don't, we can still try for a last-minute push for the enemy's flag, right before the time runs out. If we fail, the time runs out and we draw; but if we succeed, we win!

This means we can now break the map into a couple different combat zones.

Everything in range of a tower is one side's "safe zone." This is where that side *wants* combat to take placed. Everything in between the safe zones is the River Zone.

1. When the enemy is in our Safe Zone or the River Zone, group up and press the attack.
2. When the enemy is in their Safe Zone, collect body parts from the River Zone.

At all times, we'll stay closer to our flag than the enemy.

## Reverse Russian Roulette

How do we pick which minion gets a given body part? By default, creeps have four role parts (`ATTACK`, `HEAL`, or `RANGED_ATTACK`) and four `MOVE` parts. Adding more role parts will increase fatigue generation, meaning those creeps will suddenly move slower than the rest. So, we'll prioritize move parts, then role parts if a minion has fatigue capacity to spare. For now, we'll have each minion pick up only their own role parts; later, we'll evaluate combined roles. But I want to work on formations more first.

We'll set a rally point close to the middle of the river for minions to wait. They'll run out, fetch their new parts, and return to the group, where healers will restore the new parts if needed.

## Results

As I tested these changes, I did see some improvement - fewer draws against defensive (or broken!) codebases. This bumped my rating from 1600 to about 1800 before leveling off. But my scattered minions are still struggling against codebases with better coordination.

Bonus: [Here's a replay](screeps-arena://game/IMH7APVL2D) if you have Arena installed to view it!

The obvious next step is to improve our formations.