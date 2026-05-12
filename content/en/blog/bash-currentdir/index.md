---
title: "Customizing Your Bash Terminal Prompt"
description: "Complete guide to customizing the bash command prompt"
date: 2026-03-18T19:59:07+01:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [linux, bash]
---

When I work, I like to have as few distractions as possible. It may surprise some, but seeing the full directory path in VS Code bothers me. Fortunately, it is entirely possible to modify what is displayed in the terminal when working with `bash`.

In this short article I will show you some tips to customize the information displayed in your bash shell.

## Understanding the PS1 Variable

The bash command prompt is controlled by the `PS1` (Prompt String 1) environment variable. This variable defines what appears before your cursor in the terminal.
