+++
title  = "On the Topic of Conflict-Free Replicated Data Types"
date   = "2025-01-06"
author = "Jon"
tags   = []
draft  = true
+++

# Introduction

Let's say that we have an array that we want to share with others in a collaborative fashion. When one user inserts the letter 'a' at the index `0`, the others should then see 'a' at the index `0`. Then, if that same user removes the element at index `0`, the other users should see that the 'a' has now disappeared. Although we can easily describe the functionality we want, it is actually quite tricky to implement in practice.

A naive approach would be to communicate to others:

- the index we are inserting to / deleting (e.g., `0`), and
- the value we are inserting (e.g., 'a'), if we are inserting anything.

Consider the following group of events:

- Alice inserts 'a' at `0`.
- Bob inserts 'b' at `0`.
- Charlie inserts 'c' at `0`.

After Alice, Bob, and Charlie have inserted their respective elements, what does the final state of this array look like? Because you've read it top-to-bottom, you might say that the final state is `cba`. After all, Alice inserted at `0`; then Bob inserted at `0`; then Charlie inserted at `0`. However, two things:

- I never said that these operations were sequential (and in fact, we need to account for the case that these operations occur at the same time); and
- networks can be unpredictably unpredictable.

From the eyes of Charlie, there's no telling whether Alice's or Bob's operation would've reached us first. It doesn't matter whether, from an omnipotent standpoint, Alice and Bob issued their operations at the same time. The route from Alice to Charlie might be longer than the one from Bob; or issues in transmission might cause an otherwise shorter path to take a longer amount of time. In this case, we would say that the final state of our array is "inconsistent" (if you've ever heard of the CAP Theorem with regards to distributed systems, this is the 'C' part of that): Alice might see that the array is `cba`, but Charlie might see it as `cab`. Whether you want to argue both are correct, or neither are correct, there is one conclusion: that there is no single, collaborative array --- and that we have to do better.

# What to choose?

There are two ways to proceed, and we will talk about one of these options considerably more than the other.

- The first is to issue a centralized server or authority that will issue an "ordering" on these operations. If we are ever confused about the state of the array, we ask this authority; or, equivalently, we subscribe to the authority for changes, and only modify our local array when and how the authority tells us to. This is the first path: using operational transforms.

This is a common approach. Complexity is moved from the client and to the server and the actual data doesn't increase in size. You can consider almost everything that we do to be **operational transforms** on data:

- moving in a video game, server verifies the operation, server issues updated data to clients;
- inserting 'a' at the beginning of document in Google Docs (at the time of writing, it is not peer-to-peer, haha).

By the nature of how centralized using operational transforms are, if there is ever a conflict, it is the server's decision to decide the outcome state. If Alice and Bob both insert at index 0 at the same time, the server might implement a heuristic to say that, Alice's operation "beats" Bob's because Alice starts with an 'a'. A question, then, is how can we make it so disparate users have the same idea on how to reconcile conflicting operations; or on how to converge in the same, ultimate state?

# CRDTs!

CRDTs, or conflict-free replicated data types, was the result of researchers looking for the answer to this problem: on how to make disparate users converge on the same state without the need of a central authority. The [Wikipedia entry](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) on CRDTs says that they are characterized by the following properties:

> 1. The application can update any replica independently, concurrently and without coordinating with other replicas.
> 2. An algorithm (itself part of the data type) automatically resolves any inconsistencies that might occur.
> 3. Although replicas may have different state at any particular point in time, they are guaranteed to eventually converge.
