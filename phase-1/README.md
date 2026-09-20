
# System Thinking

A collection of small technical learnings from building systems.

This repository contains my notes on things I encounter while building, debugging, and understanding systems — mainly through my own projects.

The goal is not to collect definitions, The goal is to understand why something works the way it does.

## What I write here

Whenever I encounter a concept that makes me stop and think, I write down what I understood.

For example:

- B+ Tree splitting
- B+ Tree deletion
- Indexing
- Concurrency
- Persistence
- Memory layout
- HNSW
- Vector search
- Storage
- Other system-level concepts

The notes may be short or long depending on how much there is to understand.

## Connection to my projects

Most of these learnings come from actually building systems.

For example, while building my Vector DB, I may encounter:

- B+ Tree node becomes full
- Why does it split?
- What happens with a 4KB page?
- What happens to the parent?
- What invariants must be preserved?

Instead of keeping that understanding only inside the project code, I extract the underlying idea here.

The implementation belongs to the project.

The reasoning belongs here.
