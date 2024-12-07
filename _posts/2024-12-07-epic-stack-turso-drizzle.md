---
layout: post
title: "Adding Drizzle and Turso to the Epic Stack"
date: 2024-12-07 18:31:00
author: Jon Winsley
summary: I swapped out Prisma/LiteFS for Drizzle/Turso as an experiment with Kent C. Dodd's Epic Stack
---

Kent C. Dodds' Epic Stack is a sample boilerplate for an opinionated full-stack Remix app. I like it: I've used it both as a reference implementation (while building Jobfolio) and as a template (for the [Covenant Foundry website](https://covenantfoundry.com/)). One of the interesting parts of the stack is that it uses a distributed SQLite database via [Fly.io's LiteFS](https://fly.io/docs/litefs/). This makes reads extremely fast, at the cost of [limiting write speed](https://fly.io/docs/litefs/faq/#what-are-the-tradeoffs-of-using-litefs).

Fly.io's managed service, [LiteFS Cloud](https://community.fly.io/t/sunsetting-litefs-cloud/20829) has been deprecated, however. I decided to explore swapping in [Turso](https://turso.tech/) instead.

## Swapping Turso for LiteFS

The `libsql` client driver can connect to a SQLite db remotely via HTTP (Turso's hosted solution). This enables [embedded replicas](https://docs.turso.tech/features/embedded-replicas/introduction), where an SQLite database can be copied locally (within a Fly.io VM, say) and writes can be replicated to the hosted SQLite db.

## Swapping Drizzle for Prisma

Prisma's support for Turso is [currently (December 2024) Early Access](https://www.prisma.io/docs/orm/overview/databases/turso). Prisma Migrate works for local SQLite databases, replicated via file system (e.g. LiteFS), but isn't supported over Turso's HTTP connection.

Drizzle has more complete [support for Turso](https://orm.drizzle.team/docs/tutorials/drizzle-with-turso), including migrations.

## Migrating the Epic Stack

The first step was converting the Prisma schema to a Drizzle schema and setting up migrations. Between `drizzle-kit pull` to extract the schema from the existing database and [Cursor](https://www.cursor.com/) to help set up the relationships, this part was pretty easy.

Drizzle has an object syntax for querying that is similar to Prisma's, but inserts & updates have to be handled with its SQL querybuilder syntax. I went through and rewrote the database queries with Drizzle instead. This was tedious, but Cursor was able to get most of the way there with a little help.

Once I got the end-to-end tests running, I discovered there was a problem. The e2e tests ran many queries in quick succession, and some started failing randomly with SQLITE_BUSY errors. I tried both of Drizzle's SQLite drivers (libsql and better-sqlite3), and though the errors were different, neither solved the flakiness.

Drizzle _does_ work fine with the [Turso dev server](https://docs.turso.tech/local-development#turso-cli), either connecting directly or with an embedded replica. This suggests the problem is something to do with write contention against a local database. (With an embedded replica, the tests _should_ be reading from the embedded sqlite file, but writing via the sync URL to the dev server).

And... searching for answers didn't help, so that's where I'm calling it for this experiment!

## Conclusions

Covenant Foundry is currently running the Drizzle + Turso combination (without embedded replicas - yet). I'm running a Turso dev server for local development and for end-to-end tests in the build pipeline. This has been working well.

I [started a fork](https://github.com/glitchassassin/epic-stack-turso-drizzle) of the Epic Stack to demonstrate the Turso setup. The pipeline build works, though I haven't migrated the secondary cache database from LiteFS, and I haven't tested deploying it to Fly.io. Use it for inspiration in your own setup!
