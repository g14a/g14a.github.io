---
title: How I learnt to use Go Channels - Part 2 - Buffered Channels
author: Gowtham Munukutla
date: 2021-04-06 2:23:00 -0530
categories: [tutorials]
tags: [golang, channels, concurrency]
toc: true
---

## A Tiny Recap

In the earlier section [Part -1](https://g14a.github.io/posts/how-I-learnt-to-use-Go-Channels-p1/) , we have learnt:

* What channels are
* How we initialize them
* How we read from and write into a channel
* How we make goroutines communicate with channels
  
Until now we were dealing with unbuffered channels. Meaning, we know that the channel is basically a queue, but we didn't know its size. We didn't know how many data elements it could fit inside it at a time.

But when we tried different approaches and started playing around, we understand that only one data element could fit inside an unbuffered channel at a time. That is the reason they're blocking in nature. They won't be able to take in another data element unless the first one is delivered. And for the first one to be delivered, there needs to be a receiver on the other end.

<span style="color:#FF0266"><b>An unbuffered channel contains only 1 item and it blocks all sends until there is a receiver.</b></span>

## Introduction to Buffered channels

We initialize a buffered channel by giving a second parameter to the holy `make` command. By default it takes in a `capacity` of `0` which is becomes an unbuffered channel.

```go
ch1 := make(chan int, value)
```

Now you may ask:

> So now that we have declared a channel with a size `value`, the sends no longer block the goroutine?

Yes they do, but not until the buffer is full.

<span style="color:#FF0266"><b>Pushing data to a buffered channel doesn't block, unless the capacity of the buffer is completely filled.</b></span>

We can see the difference between both of them here:

| Unbuffered Channel | Buffered Channel |
| ------------- | ------------- |
|  value == 0 | value > 0  |
| Synchronous i.e blocking  | Asynchronous until it reaches value |

