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

Example: URL crawler

```go
// should have a record of the URLs, otherwise keep crawling because pages in URL is cyclic
// Serial Crawler
func Serial(url string, fetcher Fetcher, fetched map[string]bool) {
    // use a pointer of the map, not copy to prevent duplicated search
    if fetched[url] {
        return
    }
    fetched[url] = true
    urls, err := fetcher.Fetch(url)
    if err != nil {
        return
    }
    for _, u := range urls {
        Serial(u, fetcher, fetched)
        // what if change to go Serial(u, fetcher, fetched)?
        // only print one page because the main function will return, not waiting
    }
    return
}

// concurrent crawler with shared state and Mutex
type fetchState struct {
    mu sync.Mutex
    fetched map[string]bool
}
func ConcurrentMutex(url string, fetcher Fetcher, f *fetchState) {
    f.mu.Lock()
    already := f.fetched[url]
    f.fetched[url] = true
    f.mu.Unlock()

    if already {
        return
    }
    urls, err := fetcher.Fetch(url)
    if err != nil {
        return
    }
    var done sync.WaitGroup
    for _, u := range urls {
        // done.Add(1)
        done.Add(1)
        go func(url string) {
            ConcurrentMutex(u, fetcher, f)
            defer done.Done() // decrement the done counter
            // use defer to call done no matter whether ConcurrentMutex will be executed successfully
        }(u)
        // here, the string parameter is a value copy of the outer u, not pointer
        // because string is immutable. Otherwise, the u will change in the outer for loop, and will change the u in the inner function!
    }
    done.Wait() // wait the counter to down to zero
    return
}
```

### Q: If the surrounding function returns, what will happen to the inner function's reference to variables in surrounding function?

In go, the compiler will analyze the inner function(also called closured function) and allocate a heap memory for the variable. So the inner function can still get it and the gabbage collecter is responsible for noticing the last function to refer the little piece of heap and when it returns and releases then.

### Q: race detector in go
It allocates sort of shadow memory, (huge memory) almost for every memory location and keeps track of which thread read/write the memory location. And keep tracking of requiring/releasing locks and doing other synchronization activities/forcing other threads to not run concurrently. If it detects one of the location is written while another thread is reading the memory without a lock notation, it will release error. (Not static analyzing, but watching what happens in a particular part of the program. If this part didn't execute some code, like reading or writing a shared data, the race detector will never know whether there could be a race. So we need to set up some sort of a testing apparatus to make sure all the codes can be executed.) Actually the detector didn't see the interleaving/simultaneous execution, it just use the shadow memory to detect possible simultaneous read/write race without lock.

### Q: How many threads?
In practical, there will be billions of urls, and to create billions of threads(too many memory, not feasible). Solution: pre-create a fixed size of pools of workers and have workers iteratively crawl.

### Use channel for concurrent crawling
```go
// no need to worry about shared memory by using channel
func worker(url string, ch chan []string, fetcher Fetcher) {
    urls, err := fetcher.Fetch(url)
    if err != nil {
        ch <- []string{}
    } else {
        ch <- urls
    }
}

func master(ch chan []string, fetcher Fetcher) {
    // the master that use the channel
    n := 1
    fetched := make(map[string]bool)
    // range: keep waiting until receive a new channel
    // if the channel is not closed, range will keep blocking
    // only condition to return is when n==0
    for urls := range ch {
        for _, u := range urls {
            if fetched[u] == false {
                fetched[u] == true
                // n add 1 for each worker
                n+=1
                go worker(u, ch, fetcher)
            }
        }
        n -= 1
        // n=0, meaning no one is using the channel anymore
        if n == 0 {
            break
        }
    }
}
func ConcurrentChannel(url string, fetcher Fetcher) {
    ch := make(chan []string)
    go func() {
        ch <- []string{url}
    }()
    master(ch, fetcher)
}
```