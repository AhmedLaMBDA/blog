---
title: Concurrent and Parallel Programming - in Clojure
date: 31-12-2023
---

# Overview
## Elementary Concepts
Concurrent and parallel programming involves a lot of details from all levels of program execution, from the hardware to the operating system to programming
language libraries to the code.
### Managing Multiple Tasks vs. Executing Tasks Simultaneously
**Concurrency**: Managing more than one task at the same time.

> *Note that this doesn't mean we're executing these tasks at the same time,
> but we can manage and switch context between more than one task*

**Parallelism**: Executing more than one task at the same time.

> *Note that Parallelism is a subclass of concurrency,
>  as before you can execute multiple tasks simultaneously, you have to manage them first*

> *You need multiple processors to be able to achieve parallelism*

**Distributed Systems**: They are a special version of parallelism, where processors are distributed across different machines and they communicate throw Network

### Blocking vs Non-blocking
They're mostly referred to in I/O operations (reading a file/waiting for an HTTP request)

**Blocking**: Tasks are executed one after another, they are called *executed synchronously*

**Non-blocking**: Tasks don't wait for other tasks to be executed, they are called "executed asynchronously*

> *Concurrent and Parallel programming are the techniques for splitting a task into multiple subtasks,
> which can executed in parallel and manage the risks that arise from doing it as well*


## Threads
Serial code is a sequence of tasks, you indicate that tasks can be performed ***concurrently*** by putting them on a JVM Thread.

**A thread** is a sub-program, that executes its instructions while sharing access to the whole program's state.

Since Clojure is built on the JVM, The JVM provides its platform-independent thread management functionality.

**Example:**

***Thread A*** consists of multiple instructions **A1 A2 A3 A4** that needs to be executed sequentially

***Thread B*** consists of multiple instructions **B1 B2 B3 B4**

***In a single-core processor executing a single-threaded program A***

The order of execution will be: *A1 -> A2 -> A3 -> A4* - deterministic

***In a single-core processor executing two threads A-B***

The order of execution becomes *non-deterministic*, as you can't know when the processor will switch context from A to be or from B to A
It could be:
- A1 -> A2 --> B1 -> B2 -> B3 --> A3 -> A4 --> B4
- A1 --> B1 -> B2 --> A2 -> A3 -> A4 --> B3 -> B4
- And so on...

***In two-core processor executing a two-threaded program A-B***

The process becomes deterministic again since the two processors will execute the two threads ***simultaneously***


## The Three Challenges
1. Reference Cells
2. Mutual Exclusion
3. Deadlock (Dwarven Berserkers) - The dining philosophers problem

### Instructions for a Program with a Nondeterministic Outcome
ID	Instruction
A1	WRITE X = 0
A2	READ X
A3	WRITE X = X + 1
B1	READ X
B2	WRITE X = X + 1

If the processor follows *the order A1, A2, A3, B1, B2, then X will have a value of 2*, as you’d expect. 
But if it follows *the order A1, A2, B1, A3, B2, X’s value will be 1*.

> *The reference cell problem happens when two threads can read and write to the same location*

### Mutual Exclusion
> *Two threads fight for access to some resource without any way to claim exclusive access to that resource.*
For example, a write to a file, where both threads write for some time, the end result of the file will be corrupted with the function of the two threads

### The dining philosophers problem - Clear. Right?!!!!

# Clojure Concurrency Model

In a serial code execution, we bind these three events together
1. Task definition
2. Task execution
3. Requiring the task's result

```clojure
web-api/get :some-api-call
```
When Clojure sees the above, it executes it and waits for the result, *blocking* until the API call finishes.

> **Clojure tries to decouple these chronological bindings through *futures*, *delays* and *promises*, which allow us to separate task definition, task execution, and requiring the result**

## Futures

## Delays

## Promises

## Queue Time, heeeeh!

