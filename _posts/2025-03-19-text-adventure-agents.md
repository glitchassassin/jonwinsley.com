---
layout: post
title: "Text Adventure Agents: The Adventure Begins"
date: 2025-03-19 16:50:00
author: Jon Winsley
summary: There are a ton of agents that will simulate a free-form text adventure for you. But how many can actually play the classics? Let's see if we can build one.
categories:
  - ai
---

I've always loved automating things. The whole [Screeps saga](https://www.jonwinsley.com/posts-by-categories/#screeps-ref) has been about the joy of fully automating an MMORTS empire. Now, AI agents are being hyped as the wave of the future, but they have some issues to work through.

It's been said that large language models exhibit [jagged](https://www.hbs.edu/faculty/Pages/item.aspx?num=64700) [intelligence](https://x.com/karpathy/status/1816531576228053133): they are very good at some things and very bad at others. Tool-calling can fill in some of those gaps, just like when I use a calculator because I'm bad at mental math. I want to get a better handle on these limitations (and how to mitigate them).

So, I started thinking about a good playground to experiment with. [Claude has been playing Pokemon](https://www.twitch.tv/claudeplayspokemon). [Screeps](https://screeps.com/) is probably too complex and visual. But I had a lot of fun back in the day playing MUDs, which are real-time multiplayer text-based games. The real-time element would be tough, so why not start simple with single-player interactive fiction (or text adventure) games?

## West of House

Classic text adventure games like Colossal Cave, Adventure, and Zork have a fairly simple interface: you type a command like `go south`, `get sword`, or `kill troll with sword`; the game describes the results; and you continue exploring, gathering treasure, and solving puzzles until you finish the game.

This is exactly the kind of back-and-forth text interface that LLMs are being trained for. But how well do they handle the higher-level concepts - making plans, solving puzzles, spatial reasoning/navigation? And, if there *are* gaps, can we fill them in by giving our agents tools?

For example, if we confirm that spatial reasoning is a weakness ("I am in the Forest and need to get back to the House"), can we use tools like an auto-mapper and pathfinder to plan a route back?

How do smaller local models compare to more powerful hosted models?

Let's try to answer these questions.

## Test Setup

I used Cursor to help me whip up a test environment. I'm running [a Docker container with `dfrotz`](https://github.com/glitchassassin/interactive-fiction-api) (a dumb terminal client for Infocom-type text adventures) exposed via HTTP API, and I've set up [a simple agent](https://github.com/glitchassassin/interactive-fiction-agent) with Vercel's AI SDK that can start a game session and call a tool to send commands to the game. I'm running the same agent with a series of models, both local and hosted, to compare results.

The baseline just uses [a system prompt](https://github.com/glitchassassin/interactive-fiction-agent/blob/963561577e054ae237d329b755e80b1897b9f29f/src/prompts/generic-solver.ts) that explains at a high level how to play interactive fiction games; lets the LLM call the `sendCommand` tool up to five times; and then prompts it to continue if it stops early. It repeats this in a loop, stopping early at 100 iterations, in case the LLM is just stuck.

Zork reports a score, based on the number of treasures found, so I've hard-coded some checks around the score to use it as a metric to report each model's performance.

And here are the initial results:

```
┌────────────────────────────────────────────┬───────┬───────┐
│ Name                                       │ Score │ Moves │ 
├────────────────────────────────────────────┼───────┼───────┤
│ simple|mistral                             │ 0     │ 2     │ 
│ simple|claude-3-5-haiku-20241022           │ 35    │ 307   │ 
│ simple|ebdm/gemma3-enhanced:12b            │ 0     │ 11    │ 
│ simple|claude-3-7-sonnet-20250219          │ 40    │ 189   │ 
│ simple|MFDoom/deepseek-r1-tool-calling:14b │ 10    │ 248   │ 
│ simple|grok-2-1212                         │ 0     │ 315   │ 
└────────────────────────────────────────────┴───────┴───────┘
```

## Analysis

Immediately, there are a few problems. The smaller local models really struggled. 

- **Mistral** crashed out in this run. In other runs, it had other issues: it frequently began making up its own text adventure game and playing that instead!
- **Gemma 3** doesn't have tool calling built in, so I tried a variant that adds tool-calling support. Unfortunately, this wasn't reliable and crashed out quickly.
- **Deepseek-R1** also doesn't have tool calling built in, but the variant I used does work reliably enough. It manages to score a few points by solving the first puzzle (a closed window), but then gets stuck in a loop exploring the same area of the forest over and over again.

Some of the hosted models do better.

- **Grok 2** did well on a previous run, but here it too got stuck in a loop.
- **Claude 3.5 Haiku** (the smaller, faster model) did well, exploring several areas and defeating a troll, but eventually got stuck in a similar loop examining items in a single room.
- **Claude 3.7 Sonnet** scored slightly higher, but it, too, got stuck trying to open a locked door over and over again regardless of failure - until I ran out of token budget.

That's our baseline: in the next installment, we'll experiment with giving the agents tools to better keep track of the game's progress.
