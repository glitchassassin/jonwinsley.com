---
layout: post
title: "Forge: Evolving a Personal AI Agent"
date: 2025-04-16 17:00:00
author: Jon Winsley
summary: In which I begin to explore the potential of personal agents
image: /assets/Forge.png
---
The potential of AI agents continues to intrigue me, especially as newer models are released. OpenRouter demoed the mysterious "Quasar Alpha" model (now revealed to be a version of OpenAI's GPT 4.1) and it got further than any other model so far in my [text adventure agent experiment](https://www.jonwinsley.com/ai/2025/03/19/text-adventure-agents/).

I've been idly dreaming for a while about setting up a "real" agent that can run locally and interface with resources like HomeAssistant or a local Obsidian vault. After playing with agents for text adventures for a while, this seemed like it would be pretty easy to set up.

Ha! It's a rabbit hole.

## Iteration 1: Forge, A Discord Agent

<img src="/assets/Forge.png" alt="A generated profile photo representing Forge, an AI agent" style="float: right; max-width: 33%; padding: 1em 0em;">

Although I want the agent to run locally (to access resources behind my firewall, like my local home automation server), I really want to be able to access it from anywhere. So, rather than building a web interface for it, I'm just wiring it up as a Discord bot. To keep things simple, for now, I'm running it with no user authentication on a private server.

I experimented with this before for a [meme-bot](https://github.com/glitchassassin/meme-bot) that would generate appropriate memes based on recent conversations. That application flopped, but I did discover that Cursor is pretty good with [discord.js](https://discordjs.guide/#before-you-begin).

So, I kept the first iteration pretty simple: the Discord client listens for messages on each enabled channel and pops them in a queue. As they come in, the LLM processes the message, makes any tool calls, and its text output gets sent back as a Discord message.

For a basic "human-in-the-loop" confirmation, I added a wrapper for the tools which uses the [`awaitMessageComponent`](https://discordjs.guide/message-components/interactions.html#awaiting-components) strategy to wait for me to click "approve" or "reject" before continuing with the tool.

Then, I experimented with some local [MCP servers](https://modelcontextprotocol.io/introduction) to provide useful tools - read access to GitHub, access to a local filesystem folder for Markdown notes, etc. For now, these are all running as `stdio` servers. 

Eventually, I'd like to switch to `sse` servers to make it easier to re-architect things to swap in new MCP servers dynamically. I tried a couple docker containers that were supposed to proxy `stdio` servers to `sse` endpoints, but none of them worked with the MCP client in Vercel's [AI SDK](https://sdk.vercel.ai/cookbook/next/mcp-tools#mcp-tools). I'm putting that off for now.

Finally, I added a built-in scheduling tool, so the agent can schedule reminders, and they will fire off a message to the same queue as the Discord messages. This prompts the agent to do something (or remind me of something) at a specific time or on a recurring schedule - pretty handy.
### Lessons Learned

This works... pretty well! I can talk to the agent from anywhere, have it keep track of projects and to-dos in a local Markdown stash (which can then be synced by Obsidian), or do anything else an MCP server can do.

Reflecting on the progress so far, I think the client framework (wiring up Discord, the LLMs, and the tools) is a small piece of the puzzle; a great deal of the value will be in how I organize the tools/agents and script their prompts. Right now, I have to remind the agent which file to use to store things in, which tools to use, etc. A good memory system will help seed the system prompt with those relevant details so the agent has them at its fingertips.

Another frustration has been the dev experience: though I'm hosting the agent on my desktop, primarily, I also like to run it in dev mode from my laptop while I work on it. There isn't a great way to switch control of the Discord "app" between the two versions. I've hacked in a config flag so I can enable or disable a specific instance with a slash command, but productionizing this still needs work.

## Iteration 2: Next Steps

The system prompt needs to be more flexible, so we can enrich it with memory about the current context. I *did* add an MCP server that gives the agent a knowledge graph, but it doesn't always use it (unless I remind it to). We can do better than this.

I'd also like to separate out the scheduler as its own MCP service rather than being built in to the agent. To do this, we'll need another way to pipe events to the agent.

If we lean into the event-driven model, we can get a lot more functionality: an LLM's inference requests a tool call; the tool call triggers a request for human approval; human approval triggers the tool to run; when all the tool calls have run, the LLM is triggered to continue its reasoning. And, of course, we can listen for events from outside services (such as our scheduler).

The downside is that complexity (and troubleshooting difficulty) will immediately begin to spiral out of control if we aren't careful. We'll need extra visibility into the threads of conversation for debugging purposes.

I'd also like to start running at least some of these pieces on the cloud. Perhaps I'll just need to bite the bullet and set up a local proxy to connect the MCP servers running inside my home network. But a lot of this can work (and really would work better) hosted somewhere like Cloudflare.

Finally, of course, I aim to start publishing some of these reusable pieces. Stay tuned!