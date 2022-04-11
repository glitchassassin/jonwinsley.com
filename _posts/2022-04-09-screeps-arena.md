---
layout:     post
title:      "Screeps #25: Arena"
date:       2022-04-09 17:00:00
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

