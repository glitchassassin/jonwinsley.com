---
layout:     post
title:      "Developing APIs in Rust"
date:       2022-02-21 00:00:00
author:     Jon Winsley
comments:   true
summary:    "Rust is a language designed for the next forty years. Let's try building an API!"
---

Rust has been skyrocketing in popularity over the last five years or so. Stack Overflow's Developer Survey voted it the [most loved language](https://insights.stackoverflow.com/survey/2021#technology-most-loved-dreaded-and-wanted) every year since 2016. And it's designed to be around for the long haul: Rust has been described as [a language for the next 40 years](https://www.youtube.com/watch?v=A3AdN7U24iU).

Rust gives you a lot of low-level control while preventing many dangerous security issues, making it an ideal candidate to replace systems languages like C or C++. But it's also a high-level language with a growing ecosystem of tools and crates to accelerate application development. As an example, we'll use [Rocket](https://rocket.rs/) to develop a simple, efficient API server.

## Setting Up the Workspace

Visual Studio Code's [devcontainers](https://code.visualstudio.com/docs/remote/create-dev-container) are a great way to spin up a self-contained dev environment with compilers and extensions pre-configured. You can do the work yourself, or you can just have Code spin up an environment for you:

![Adding devcontainer config files automatically](assets/rust-create-devcontainer.png)

After opening your workspace in the devcontainer, you'll have the Rust `cargo` tool and some useful extensions. (Side note: I had to uninstall and re-install the `rust-analyzer` extension to get it to work; your mileage may vary.)

## Planning Our API

