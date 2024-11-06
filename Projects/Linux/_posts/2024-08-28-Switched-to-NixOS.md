---
layout: post
tags: OS nix
title: I switched to using NixOS!
---

Hello! It's been a while since I've written anything here...

I had heard about NixOS from a few different sources, so over the summer I decided to give it a shot! It's really quite nice; I don't picture myself using anything else, since it can effectively mimic just about any linux distro out there.

NixOS uses the nix language, a purely functional programming language developed specifially for the nix package manager and for NixOS. The nix package manager is what NixOS uses to manage packages, but it can also be used on other disributions or even macOS.

The cool thing about this package manager is its declarative nature; you write nix programs which declare which packages you want to use, and the nix package manager handles getting the packages. Indeed, since the nix language is turing complete and whatnot, you can basically do *anything* with your package scripts!

Indeed, in NixOS, you can even declaratively create all settings for literally everything on your system. Many of these settings, for example to enable and configure systemd settings, are built in to the existing variables, but even if that's not the case, you can just write your own nix program to set your settings.

Another feature of the nix package manager is that you can use nix-shell to create a shell environment with specific packages available. This is useful for setting up development environments and whatnot, since you can choose the exact packages and their versions to use, and nix-shell keeps them separate from the rest of your system (sort of like venv in python, but for any nix package).


Also, the nix package repository is HUGE! It's literally [one of the biggest out there](https://repology.org/repositories/statistics/total) (it's *the* biggest at the time of writing). So missing out on packages is not a concern when switching to NixOS.

The other cool thing about the configuration of NixOS is that it builds up from scratch. What I mean by that is that you can configure everything from the desktop environment/window manager (if you need one) to which kernel version to use, to exactly which services and systems to enable (don't need audio because you're using NixOS on a home server or something? Just turn it off with one option!), allowing for lots of flexibility.

Because of the declarative nature, you can just copy your configuration nix files to a different computer, run a single "nixos-rebuild" command, and, without exaggeration, an effectively identically configured system will be created. How wonderful!

<br><br>
Anyways, I finally got around to setting up ruby and jekyll with nix-shell today, so here I am finally writing an update.

Happy linuxing!
