---
description: Just have a try.
---

# Project 1: Unix utilization

**Basic Project from CS537:**

This project will let us to get familar to C. Basically what we want to do is try to implement some commands, i.e. look, across, diff in our own versions.

&#x20;However, as I try to have a glimpse of the functions here, I come across that the command may differ in various Linux distribution.

e.g. In Ubuntu, `Look` will perform a binary search, which will greatly depend on the file. It should be a sorted file.

Smoothly finished the first two.

The third part of this Project is my-diff function, basically it recalls me the rust code in CS110L we implemented beforehand, so this time we will use C to fork the rust code (Haha).

Basically we should pay more attention to Longest Common Subsequence(LCS) algorithm here, just have a try.

The implementation in C takes me a little more time in fact, because the string here is not so easy to be handled with (compared to C++ or Rust), I basically use malloc to help me to do this.

**Extra Project from OSTEP:**

**wcat & wgrep: very easy tasks.**

wzip is a little hard for the second test cases, but you can use **fgetc** to handle this challenge.
