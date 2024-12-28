---
layout: post
title: "Portfolio: Jobfolio"
date: 2024-12-28 14:47:00
author: Jon Winsley
summary: ""
categories: portfolio
series: portfolio
image: /assets/jobfolio.png
---

![Screenshot of Jobfolio app](/assets/jobfolio.png)

Jobfolio was my first "real" software startup venture. Gandalf, a friend I met through the Reformed Devs Discord group, was looking for help to finish building out his job management app for the trades. He'd hired developers in the past to begin the MVP, but really wanted a technical cofounder rather than a contractor. In January 2024, we struck an agreement and set out to finish the MVP.

I'd planned to change as little as possible of the basic tech stack, but quickly ran into some important issues: the existing Java backend relied on proprietary libraries owned by one of the contracted developers, and there were some scalability and security concerns. So, we pivoted, and I created a fresh build with [Remix](https://remix.run/) and [Prisma](https://www.prisma.io/), running on [AWS Lambda](https://aws.amazon.com/lambda/).

Since we already had the prototype half-built, we could skip past the most time-consuming part of development - figuring out requirements. By the end of March, we had rebuilt the MVP, and Gandalf worked to gather feedback and sell subscriptions while I continued development.

My focus from the beginning was to create an environment where development would be fun. I minimized friction by prioritizing simple end-to-end tests over thorough unit tests; setting up CI/CD to immediately release changes to production; and PR environments with temporary infrastructure, for manual QA before merging changes.
