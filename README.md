# System Thinking

A collection of technical notes and mental models built while engineering systems.

This repository captures the thoughts, reasoning, and debugging processes I worked through while building systems — primarily in the context of my own projects. 

I don't focus on textbook definitions; I focus on **why** things are designed the way they are.

---

## What is in this repository

Whenever a project forces a deep dive or triggers an interesting realization, I document it here:

* **Indexes & Trees:** B+ Tree splitting, B+ Tree deletion, HNSW
* **Storage & Memory:** Memory layout, page sizing, persistence
* **System Design:** Concurrency, vector search, and core storage mechanics

Notes vary from quick engineering logs to detailed design breakdowns depending on the complexity of the topic.

---

## Relationship to my projects

These notes stem directly from hands-on projects. For example, while building my **vector database**, I had to handle B+ Tree node splitting:
* What exactly triggers a split?
* How much data fits onto a single 4KB page?
* What updates must be propagated to the parent node after a split?

While the implementation lives in the project source code, the architectural reasoning belongs here.

> **Code lives in the project. The mental model lives here.**
