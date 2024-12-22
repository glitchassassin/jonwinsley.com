---
layout: post
title: "Portfolio: Deacon Notes"
date: 2024-12-22 14:47:00
author: Jon Winsley
summary: ""
categories: portfolio
series: portfolio
image: /assets/deacon-notes.png
---

![Screenshot of Deacon Notes app showing sample contact data for Jon Winsley](/assets/deacon-notes.png)

I am a deacon at my church. We help the pastoral staff to care for the congregation in a few ways; one of the most important, in a growing church our size, is simply staying in touch with everyone to try to make sure no one falls through the cracks.

Each deacon is assigned a "care group," a subset of families from the congregation. He personally stays in touch with each family, in case they have prayer requests or other needs the church can meet. When something comes up, he brings it to the rest of the deacons and pastoral staff, and the church comes together to pray for and/or meet the needs.

These conversations happen across several platforms - in person, email, phone, the church's chat app - and it can be tough to keep track of updates. Our church uses Subsplash as our church management platform, but the complex interface is intimidating for some of our deacons who aren't as technically inclined. We discussed a few other options, such as a shared spreadsheet, but these all had various disadvantages and would be difficult to keep in sync with our church database.

When I started looking into the platform, I realized that Fluro (the people management side of Subsplash) had [a developer-friendly API](https://developer.fluro.io/)! This meant I could build a simpler frontend and still use Subsplash as our backend.

Since we didn't need (or want) a separate backend, this was an ideal case for a static single-page app (SPA). I spun up an app with React Router 7, hosted for free in Cloudflare Pages, and with Cursor's help had a clean and functional app scaffolded in a few hours.

[This app is also open source!](https://github.com/glitchassassin/deacon-notes)
