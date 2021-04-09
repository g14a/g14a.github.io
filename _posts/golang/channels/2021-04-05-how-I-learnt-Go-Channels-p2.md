---
title: How I learnt Go Channels - Part 2 - Buffered Channels
author: Gowtham Munukutla
date: 2021-04-06 2:23:00 -0530
categories: [tutorials]
tags: [golang, channels, concurrency]
toc: true
---

## **A Tiny Recap**

In the earlier section [Part -1](https://g14a.github.io/posts/how-I-learnt-Go-Channels-p1/) , we have learnt:

* What channels are
* How we initialize them
* How we read from and write into a channel
* How we make goroutines communicate with channels
  
Until now we were dealing with unbuffered channels. Meaning, we know that the channel is basically a queue, but we didn't know its size. We didn't know how many data elements it could fit inside it at a time.

But when we tried different approaches and started playing around, we understand that only one data element could fit inside an unbuffered channel at a time. That is the reason they're blocking in nature. They won't be able to take in another data element unless the first one is delivered. And for the first one to be delivered, there needs to be a receiver on the other end.

<span style="color:#FF0266"><b>An unbuffered channel contains only 1 item and it blocks all sends until there is a receiver.</b></span>

## **Introduction to Buffered channels**

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

Let us create a buffered channel and try to communicate with the goroutine.

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    ch := make(chan float64, 5)
    go write(ch)

    ch <- 10
    ch <- 20
    ch <- 30
    ch <- 40
    ch <- 50

    fmt.Println("main exits")
}

func write(ch chan float64) {
    for i := 0; i <= 5; i++ {
        float := <- ch
        fmt.Println(math.Log(float))
    }
}
```

The above program gives the following output:

```bash
main exits
$ >
```

This is because we exactly gave the channel 5 values and it equals its capacity precisely.

Now let's see what happens if we give it one more data element. Your main function would look like this:

```go
func main() {
    ch := make(chan float64, 5)
    go write(ch)

    ch <- 10
    ch <- 20
    ch <- 30
    ch <- 40
    ch <- 50
    ch <- 60

    fmt.Println("main exits")
}
```

This time you might get a variety of outputs like the following:

```bash
2.302585092994046
2.995732273553991
3.4011973816621555
main exits
$ >
```

```bash
2.302585092994046
2.995732273553991
3.4011973816621555
3.6888794541139363
3.912023005428146
4.0943445622221
main exits
$ >
```

```bash
2.302585092994046
main exits
$ >
```

This happens because computation of `log` takes some time and it can sometimes depend
on other applications running on your machine which take up CPU. For best results, the workaround is to apply a `time.Sleep(time.Second)` after sending in all data to the channel.

Although we did not get the best results, we proved that goroutines don't block when the buffered channel gets more data than its capacity.

Interestingly, running the above program with `GOMAXPROCS=1` gives all the results 99% of the time. This can happen because Go selects a processor in which the main function is already running and immediately assign the new goroutine to the same thread. When `GOMAXPROCS` isn't enabled, it CAN happen that the new goroutine might be assigned to some other system thread on another core. This can cause a little scheduling latency and might give out varied results.

## **Something Interesting**

Let us see if the goroutine and the main function run on the same system thread always.

```go
package main

import (
    "fmt"
    "math"
    "runtime"
    "syscall"
)

func main() {

    fmt.Printf("Main:System Thread Id:%d\n", syscall.Gettid())
    fmt.Printf("GOMAXPROCS=%v\n", runtime.GOMAXPROCS(8))
    ch := make(chan float64, 5)
    go write(ch)

    ch <- 10
    ch <- 20
    ch <- 30
    ch <- 40
    ch <- 50
    ch <- 60

    fmt.Printf("Main:System Thread Id:%d\n", syscall.Gettid())
}

func write(ch chan float64) {
    fmt.Printf("Write:System Thread Id:%d\n", syscall.Gettid())
    for i := 0; i < 6; i++ {
        float := <- ch
        fmt.Println(math.Log(float))
    }
}
```

We can get varied outputs again.

```bash
Main:System Thread Id:12
GOMAXPROCS=8
Write:System Thread Id:12
2.302585092994046
2.995732273553991
3.4011973816621555
3.6888794541139363
3.912023005428146
4.0943445622221
Main:System Thread Id:14
```

We notice that the `write` goroutine is mapped to a thread with an id 12. And the main system thread runs on 14. And there is a chance that both of them run on the same thread too.

## **Reading from a buffered channel**

Buffered channels are readable even if their capacity isn't filled. Delivery of data is blocked until the capacity is full but not when you read from it.

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    ch := make(chan float64, 3)

    fmt.Println(len(ch), cap(ch))

    ch <- 10
    ch <- 20

    fmt.Println(<-ch)
    fmt.Println(<-ch)

    fmt.Println(len(ch), cap(ch))
}
```

This gives the output:

```bash
2 3 
10
20
0 3
$ >
```

We see that the buffer did not even reach its capacity, but it lets us read from it. We also try to understand the difference between the length and capacity of the channel here.

The capacity of the channel doesn't change once declared in the `make` function. The `len` keeps changing as values are added and read from the buffer. We see that after we read both the values from the channel, the `len` becomes `0`.

## **Fun stuff**

Until now we've tried sending data into a channel in the main function, and reading it in a goroutine. Can we do the opposite? Absolutely.

When we run the following program,

```go
package main

import (
    "fmt"
)

func main() {
    ch := make(chan int, 3)
    go send(ch)
    for val := range ch {
        fmt.Println(val, len(ch), cap(ch))
    }
}

func send(ch chan int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}
```

we get this:

```bash
0 3 3
1 3 3
2 2 3
3 1 3
4 0 3
$ >
```

Notice how the length of the channel is `3` even after reading the first element `0`. This is because the element `3` is inserted immediately into the channel in the 4th iteration of the for loop. 

## **Key points learnt**
* An unbuffered channel contains only 1 item and it blocks all sends until there is a receiver.
* Pushing data to a buffered channel doesn't block, unless the capacity of the buffer is completely filled.
* You can read from buffered channels even if their capacity isn't filled.

## **Conclusion**

Hope you enjoyed the second part of the tutorial. In the next one we talk about Channel types.

Check it out at [Part-3 Channel Types](https://g14a.github.io/posts/how-I-learnt-Go-Channels-p3/)

Please reach out to me via email(or any social media linked down below) if you think I haven't covered something which you consider important.

Thank you 😁