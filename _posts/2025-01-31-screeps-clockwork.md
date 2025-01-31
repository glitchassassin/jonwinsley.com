---
layout: post
title: "Screeps #27: Optimizing Pathfinding with Rust"
date: 2025-01-31 17:00:00
author: Jon Winsley
comments: true
summary: I beat the default pathfinder in Screeps by building a faster Rust/WASM library.
categories: screeps
series: screeps
---

In August of 2022, I [spun up a new library](https://github.com/glitchassassin/screeps-cartographer), dubbed Cartographer, to deal with some common issues with Screeps' built-in `moveTo()` method for pathfinding. Two of the main ones were long-distance pathing (including traversing portals, which my bot had no way to deal with in the previous Screeps Warfare Championship) and traffic management (coordinating swarms of creeps all moving around the same areas).

Cartographer was effective: its strategy for limiting rooms to be explored meant it could find longer paths more quickly than the default implementation, and though it may not have used a truly optimal traffic management algorithm, it worked well enough for swarms of creeps at low RCL.

Unfortunately, it was also slow. It was designed as a drop-in replacement for `moveTo()` that could handle everything, but that power came at the cost of efficiency.

So, recently (in October of 2024), I decided to tackle the problem from a new angle: instead of a single jack-of-all-trades `moveTo` function, I wanted a library that would provide small, efficient, composable elements. Units with different pathfinding requirements could plug together just the pieces they needed, and no more. And, for maximum efficiency, I wanted to write it in Rust.

## Rust/WebAssembly

Screeps supports calling WebAssembly from a Javascript codebase. Some players take advantage of this to write a whole codebase in Rust (or some other WASM-compilable language), but I like the flexibility of Typescript for the most part - working around Rust's more stringent rules is sometimes more painful than helpful. So, I set up an NPM-installable package. The WASM module is compiled from Rust, and then a [Typescript wrapper handles the interface](https://glitchassassin.github.io/screeps-clockwork/wasm/), converting some things to or from Screeps game objects (like RoomPositions).

This meant the build had a few pieces: the WASM module itself, the Typescript wrapper, and a test rig that runs in a Screeps environment and runs [a full suite of integration tests](https://glitchassassin.github.io/screeps-clockwork/testing.html).

I experimented with Nx for the monorepo build, but it did not work well with the Rust compilation step. Eventually, I pared it down to simple package.json commands to run cargo-watch and rollup watch in parallel, and configured the Rollup build to output separate artifacts for the distribution package and the test bot.

Now, in watch mode, changes to either the Rust or Javascript code are compiled and rolled out to the Screeps test server in seconds for rapid development iteration - and it feels good!

## Composable Algorithms

Pathfinding on a variable-cost grid like Screeps is a well-researched problem, and there are common algorithms like breadth-first search (BFS), Dijkstra's algorithm, and A\* that RedBlobGames [does an excellent job of explaining](https://www.redblobgames.com/pathfinding/a-star/introduction.html).

Besides simply improving performance of the algorithm, one of the most useful ways to speed up pathfinding is pre-calculating and caching data. For example, if you're frequently pathing from somewhere in the room to the same destination (say, the Storage), a cached distance map can give you a path to Storage from anywhere in the room very cheaply. So, Clockwork splits up "generating a distance map" from "generating a path," so you can cache the distance map separately and re-use it if that makes sense for your use case.

## Maximizing Performance

Although the main focus of Clockwork was on improving performance through pre-calculated data, I wanted to start out on a level playing field, such that the baseline A\* performance was comparable to Screeps' built-in PathFinder. Then, any improvements we could get from there would be icing on the cake.

This proved more challenging than I expected, and I learned a great deal about writing performant code in Rust - using indexed arrays instead of hashmaps, caching to avoid unnecessary lookups, etc. @RediJedi and other Rust fans in the Discord were instrumental in helping me work through some of these issues.

Ultimately, with some inspiration from the built-in PathFinder implementation, we were able to beat its performance even before any precalculation improvements were applied!

## Future Enhancements

We've built a pretty good foundation for further research. Enough pieces are in place that Clockwork _could_ be used for pathfinding in Screeps today. The core concepts have been proven out, though the dev interface is still pretty rough.

Future improvements could include extending an existing distance map (instead of creating a new one from scratch), better heuristics that take into account Screeps' unique topography, and maybe even some traffic management algorithms that use distance maps to allow crowds of creeps to follow multiple paths to a target.

I'm taking a break from Clockwork - and Screeps - for a bit, as I focus on some new side projects around generative AI, but if you do anything interesting with it, I'd love to hear about it! Ping me (Lord Greywether) in the Discord or on X.
