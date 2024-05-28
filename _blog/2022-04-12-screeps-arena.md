---
layout:     post
title:      "Screeps #25: Arena"
date:       2022-04-12 17:30:00
author:     Jon Winsley
comments:   true
summary:    
categories: screeps
series: screeps
---

# Screeps Arena

[Screeps Arena](https://store.steampowered.com/app/1137320/Screeps_Arena/) is now in early access! I've taken a six-month break from Screeps, give or take, but I have been excited to give Arena a try once it was finally available.

I began going through the tutorials, and set up the [Typescript starter kit](https://github.com/screepers/screeps-arena-typescript-starter/) (still in early stages). It's much easier to break into than World, thanks to the simplicity of the rulesets. I picked Capture the Flag as my first arena specifically for that reason: minions and structures are already spawned, and you can focus on improving your combat rules, iterating over and over to test new tactics.

# Where's the code?

Unlike my [Screeps: World](https://github.com/glitchassassin/screeps) codebase, I've elected to keep my Screeps: Arena codebase closed-source for now. An MMO codebase includes a lot of commonly useful things that don't give a direct competitive edge; but Arena is a purer competition, and in the interests of keeping things fair, I don't intend to give away a competitive codebase to everyone.

In these articles, I'll discuss tactics and strategies in general terms, and perhaps talk through some specific technical issues. You're more than welcome to implement any of these ideas in your own codebase.

# Capture the Flag

The basic CTF arena puts you in one corner of the map and your opponent in the other; you both have a flag, two towers, and fourteen pre-spawned minions. Your goal is to move any of your minions to step on the opponent's flag.

I started out with the most minimal of tactics: just hurl all the minions at the enemy's flag, and let towers attack anything that gets close. This works great on idle opponents, but clearly runs into issues against a minimally competent codebase.

Then I tried a simple modification, huddling all the healer minions on top of my own flag, and having them heal each other. The rest of my minions would go after the enemy's flag while these protected my own. This worked better, and nicely counters my original strategy, which I saw from a few other codebases. However, I started running into opponents that would proactively clean up my attacker minions - and with no supporting healers, that did not take long. Then their force would move on and dismantle my unprotected healers.

So, the main force needs supporting healers. I left one healer to sit on the flag and heal itself, supported by the towers, as a sentry; the rest would follow the attackers, healing whoever needed it. This was somewhat better; the main force would properly engage the enemy and survive, but if the enemy split into two groups (and it commonly does), one group would be ignored until it was too far past the main force's defenses, and then the single healer-sentry could not survive the coordinated attacks.

Naively, let's try to keep our minions between the enemies and our flag. If the enemies are spread out, we should hang back to make sure we can intercept them; if they are close together, we can move in for the kill. The solution that occurs to me first is geometric: draw a line from our flag through the center of the enemy forces. Then, draw two lines at 45-degree angles from this line, so that the enemy are all on the far side. Consider this expertly-crafted illustration:

![Intercepting the enemy](/assets/screeps-arena-angles.png)

Actually implementing this was more of a headache than it should have been, but I finally figured out a way to use the Rule of Sines to calculate the intercept point. It ended up working fairly well; when the enemy's force splits in two to cross the river, our army immediately begins backtracking to our flag and intercepts the enemy under our towers.

At this point I took a break from implementation and just started running a bunch of ranked games, stopping whenever we lost a game. I watched the replays and took notes until we eventually settled out at a rating of around 1000.

## Analysis

As we climbed the ladder, I ignored victories and focused on replays of defeats and draws. I began to see two key patterns in the codebases that we lost to:

1. Better coordination between minions. They were clustering better, focusing healing and damage. A couple were running quads.
2. Defensive. They waited for our minions to get in range of their towers before engaging, using the towers to swing the encounter in their favor.

## Improving Coordination

Right now our minions are very naive: they move to the intercept point, attacking any enemies in range, and healing any friendly minions in range. But they frequently split up into smaller groups because of terrain and separate pathing calculations. The healers try to stay close to the ranged attack minions, but the group still gets spread out considerably, which makes it difficult for healers to be effective.

For the next major iteration, we'll path as a group. We'll pick the most central minion as the "centroid," and calculate a path for the group from there to the current objective. Then we'll pick a point on that path just past the edge of the group and have all the minions path to it. This does a better job of keeping the minions clustered together, and as a bonus reduces the complexity of our pathing each tick.

We'll consider healing as a group too: starting with the most damaged minions in the group, we'll allocate healers within range until we run out of available healers. This could be optimized further to maximize healing - we'll come back to this.

Finally, damage. Not all minions will be in range to attack the primary target; if they are not, they'll pick a secondary target, preferring one that is already being attacked (to focus fire even on secondary targets).

One more change: calculating the intercept point. Instead of trying to hover between the enemy force and our flag, we'll follow some simple rules:

1. If the enemy is inside our tower radius, attack them - but always stay three steps closer to the flag.
2. If the enemy is outside our tower radius, but we outnumber them two to one, engage them.
3. Otherwise, wait for them to get in range of our towers, to tip the engagement in our favor.

These improvements gave us a significant boost from a 1000 rating to a very respectable 1600!

From here, I already see clear areas to improve. I'll branch out into posts that dig deep into some of these - coordinating damage and healing, high-level arena strategies, etc.

Stay tuned!