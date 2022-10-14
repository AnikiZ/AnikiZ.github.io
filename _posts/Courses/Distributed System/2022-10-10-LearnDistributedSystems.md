---
title: Distributed Systems Notes
date: 2022-10-10 02:28:00 +0400
categories: [Courses, Distributed System]
tags: [Go Instant messaging System Programming]
img_path: /assets/img/Courses/Distributed System
---

# 1. RPC and Multi-Threads
reason for multi-threads:
1. I/O concurrency
2. Parallelism(multiple cores, true parallelism, multiple cpu cycles per second)
3. Convenience(fire something every some seconds in loop waiting for separate work; notice to exit every routine gracefully! otherwise accumulating in background!)

what if not use multi-threads while still enabling multiple servers to communicate with various clients?
<font color=Blue>Async programming/Event-Driven programming</font>
single thread, single loop waiting for any input/event/timer fire
interesting: use multiple cores, each responsible for an event-driven loop, can also act like multi-threads

process vs thread
Process: a single programming you are running. In a go program, it will create one UNIX process and one memory area. When you create many go routines, these threads will run sitting in this process within same memory space.