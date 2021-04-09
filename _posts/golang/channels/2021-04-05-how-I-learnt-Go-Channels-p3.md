---
title: How I learnt Go Channels - Part 3 - Select & Types of Channels
author: Gowtham Munukutla
date: 2021-04-06 2:23:00 -0530
categories: [tutorials]
tags: [golang, channels, concurrency]
toc: true
---

## **A Tiny recap**

In the earlier section [Part -2](https://g14a.github.io/posts/how-I-learnt-Go-Channels-p2/) , we have learnt:

* What buffered channels are
* Reading and writing to a buffered channel
* Communicating with goroutines with buffered channels

As business and systems scale, there wouldn't be just one goroutine in the application. There might be hundreds of them or even thousands. And these goroutines might be communicating between themselves via multiple channels. And sometimes when one goroutine blocks, the others keep running. So it implies that if one channel is waiting for data to come in, other channels already have data in it and are in need of a receiver i.e they are ready to deliver data. So is there a way we can use whatever data is incoming from whichever channel instead of waiting for whether or not it has data? Absolutely.

# **Select**

The `select` keyword lets you get values out of simultaneously running goroutines. It is like a `switch` statement but for channel operations. There's also a `default` case in `select` just like in a `switch` statement.

The `select` statement does the following:

* Switches on channel operations to see which one can proceed immediately.
* If multiple channel operations aren't blocking, it choses one of them at random. 
* If all channel operations are blocking, it waits until one of them isn't.
* If none of the channel operations can proceed, and there is a `default` case, it executes the `default` case.

The `select` keyword is generally paired with the `for` loop in Go. We either loop infinitely if we have no clue about how much data we get into our channels, or in a `for range` loop.

Let's see how `select` works in the following program. We initialize two unbuffered channels `helloChan` and `worldChan` and pass data into it in two seperate goroutines.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    helloChan := make(chan string)
    worldChan := make(chan string)

    go hello(helloChan)
    go world(worldChan)

    select {
        case msg := <- helloChan:
            fmt.Println(msg)
        case msg := <- worldChan:
            fmt.Println(msg)
    }

    time.Sleep(time.Millisecond*100)
}

func hello(helloChan chan string) {
    helloChan <- "Hello"
}

func world(worldChan chan string) {
    worldChan <- "World!"
}
```

Upon running several times, we see with `Hello` or `World!` in the output. This is because both of the channel operations at line 16 and 18 are non blocking. So any one of them is selected at random.

Now let's try blocking one of them by adding a `time.Sleep` before sending data into `helloChan`. Out `hello` functions becomes this.

```go
func hello(helloChan chan string) {
    time.Sleep(time.Millisecond*100)
    helloChan <- "Hello"
}
```

Now the program always prints `World!` because the first case is blocking. We're sleeping for 100ms before sending data into `helloChan` and by then, `worldChan` already has data in it ready to be delivered. So the `select` statement always executes the second case.

Let's try a send operation on a channel and see what happens:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    helloChan := make(chan string)
    worldChan := make(chan string)

    go hello(helloChan)
    go world(worldChan)

    select {
        case msg := <- helloChan:
            fmt.Println(msg)
        case worldChan <- "World":
    }

    time.Sleep(time.Millisecond*100)
}

func hello(helloChan chan string) {
    helloChan <- "Hello"
}

func world(worldChan chan string) {
    msg := <- worldChan
    fmt.Println(msg)
}
```

This can either print out `Hello` or `World!`. Now its upto you to try adding a blocking call by making one of the goroutines sleep for a while and explore all the combinations.

Let's try adding a default case and see that it does something interesting. Our `select` statement becomes this:

```go
select {
    case msg := <- helloChan:
        fmt.Println(msg)
    case worldChan <- "World":
    default:
        fmt.Println("default case")
}
```

Now we see that the program always prints `default` case. But we've seen that default case gets executed only when all the cases cannot proceed. But they were proceeding in the previous example when the `default` case wasn't present. What is different now?

In the third and fourth point when we introduced `select` we've seen that: 

> If all channel operations are blocking, it waits until one of them isn't.

> If none of the channel operations can proceed, and there is a `default` case, it executes the `default` case.

So in the previous example, there is no `default` case. So `select` waited for atleast one of them to happen and executed that case immediately. But now since the `default` case exists, `select` doesn't wait anymore and proceeds to run the default case. Try adding a sleep just before select and see what happens for yourself.