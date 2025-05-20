---
layout: post
title: "Forge: Rearchitecting"
date: 2025-05-20 10:30:00
author: Jon Winsley
summary:
---

[Dex](https://x.com/dexhorthy) from HumanLayer created [12 Factor Agents - Principles for building reliable LLM applications](https://github.com/humanlayer/12-factor-agents/tree/main).

Much of this resonates with my experience building the first iteration of Forge. My early attempts at [human-in-the-loop](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-8-own-your-control-flow.md) depended on awaiting a callback from Discord - keeping the state in memory. And many of my ideas for improvement center around [context engineering](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-3-own-your-context-window.md).

So, incorporating some of these ideas and my own lessons learned, I took a stab at a more robust architecture.

[Here's the Forge repo.](https://github.com/glitchassassin/forge)

## The New Architecture

Instead of being event-driven, the new architecture creates messages in a SQLite database table when an event (webhook, Discord message, tool result, scheduled event, etc) fires.

The main loop checks this table for new messages. Tool requests are processed first, but without blocking: an async promise is created, and when it resolves, it creates a message (either a tool response or an error, depending on the result).

Then, it checks if any of the new messages should trigger the LLM. Its own agent messages don't, of course, nor do the successful tool responses from sending a Discord message.

By default, providers expect any tool calls to be resolved before you continue inference. That may not be the case if the tool call is pending approval. So, at runtime, we insert a temporary "pending" or "waiting for user approval" tool response into the context if we're still waiting on the answer.

### Built-In Tools

I'd like to extract most of the functionality into MCP servers, to make it easier to swap out components in different implementations as I play around with agents. However, some tools are easiest to implement as part of the agent itself - like scheduled events. The [Model Context Protocol](https://modelcontextprotocol.io/introduction) doesn't have a good way to trigger client-side events, so I just built this into the base agent framework.

Importantly, the scheduler doesn't just send reminder messages to the channel: it sends reminder messages *to the agent*, which can then message the channel or take other actions as needed.

The other built-in tool is for sending Discord messages, and it's how the agent communicates with the user.

### MCP Tools

But agents are more fun with more options, so of course Forge can invoke tools from remote MCP servers. I built a [simple notes service](https://github.com/glitchassassin/mcp-notes), backed by Durable Objects on Cloudflare, and authenticated with an authorization header token.

MCP servers are connected dynamically via Discord interactions and modals:

![A series of Discord messages listing connected MCP servers](Pasted image 20250520065643.png)

Since tools might be dangerous, I built in a tool call approval flow: tools may either be always approved (in which case they are called immediately) or manually approved. The user is prompted with a Discord message when a tool call requires approval; based on whether they click "approve" or "deny," an appropriate message is pushed to the messages table, and the tool call runs (or doesn't) with the next main loop.

These tools can also be managed directly via Discord interactions:

![A series of Discord messages listing MCP tools](Pasted image 20250520065717.png)

## Results

Forge is still pretty rough, but functional! The most useful application so far has been keeping me accountable to daily routines:

![[Pasted image 20250513131143.png]]

Some of the rough edges that remain:

1. Each channel is its own conversation context, so Forge has no visibility to other conversations (except via artifacts like notes).
2. Scheduled reminders also exist in the conversation context, so it can't create reminders in other channels - which might be helpful.
3. Troubleshooting involves connecting to the Fly.io instance and querying the sqlite database - there's no easier way to inspect the actual messages.

Where to go from here? I accepted some limits to build Forge on Discord: we don't get custom UI, so visualizing tool calls, reasoning, etc. in the flow of the conversation is tough to do without being intrusive. And we're still only dealing with one-to-one conversations rather than embracing the possibilities of a many-to-many group chat.

For personal use, a custom "single-player" app with better debugging would be better - I could even build it as a PWA and still get push notifications.

I think there are probably useful roles for a group-chat capable Discord bot - imagine if Forge could intelligently decide which conversations to participate in, rather than eagerly responding to *every* message, or responding only when directly tagged. I'm not sure if the technology is quite there yet, but it would be fun to find out!