+++
title = "Taking advantage of state machine designs"
description =  "Something something state machine"
date = 2023-05-05
draft = true
slug = "state-machine"
+++

In this post I am going to talk about exploring the concept of a state machine to help organize code flow. I am not going to build a full implementation of a state machine, but rather borrow a few concepts.

<!-- more -->

# What is a state machine and when should I use it?

[Wikipedia](https://en.wikipedia.org/wiki/Finite-state_machine) has a very nice article on the subject, but the part I am most interested on is this sentence:
> It is an abstract machine that can be in exactly one of a finite number of states at any given time.

In other words, a state machine can be in one state at time, and only one state, which makes it a good fit when there is a process that do only one thing at a time, but it is not particularly useful when said process need to do multiple things in parallel, although, it is possible to have multiple state machines running in parallel inside of the same process.

Another property that is also important, a state machine has a finite number of states, trying to model a dynamic number of states on top of a state machine design is probably a bad idea.


# Implementation requirements

When chosing what code design should be used for a task, there are different things that should be taken into consideration. A few questions that can be asked are:

- How easy it is to understand? 
- How easy it is to reuse?
- How easy it is to write and extend?
- How performant it is?

The first 3 questions are the most important for me, but unfortunately, very often overlooked by many people. Code that is simple and easy to reason about makes your life easier when you need to optimize it later. 
Of course, performance can become the most important aspect, if the situation demands.

# First, with enums

# Second, with dyn traits

# Third, composable traits

# Benchmarks

# Conlusion

