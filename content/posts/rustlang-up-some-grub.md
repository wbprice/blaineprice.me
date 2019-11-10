+++
title = "Rustlang Up Some Grub at The Ten Top"
date = "2019-11-09"
author = "Blaine"
cover = ""
keywords = ["rustlang", "gamedev"]
description = "\"The Ten Top\" is the working title of a game that I'm writing in Rust using the [Amethyst](https://fixme.com) game development framework. My goals with the project are to make a fun casual simulation game in the vein of Game Dev Story and Overcooked."
showFullContent = false
+++

"The Ten Top" is the working title of a game that I'm writing in Rust using the [Amethyst](https://fixme.com) game development
framework. My goals with the project are to make a fun casual simulation game in the vein of Game Dev Story and Overcooked. 

The object of the game is to make enough money to:

- Cover operating costs of the restaurant every month
- Pay the worker(s)
- Eventually sell the restaurant and retire

A core mechanic of the game is taking orders from customers (e.g "one hot dog, please") and asking workers to prepare the given
dish, then give it back to the customer.

To deliver a hot dog, a specific set of conditions need to be met:

- A hot dog bun needs to exist
- A hot dog link needs to exist
- A hot dog link needs to be cooked
- A cooked hot dog link needs to be combined with a hotdog bun on a plate

These steps result in a (pretty bland, to be honest) hot dog.  A first attempt to code this logic might would include a lot of hotdog-specific code ("Does a hot dog bun exist", "Does a hot dog link exist?", etc)


