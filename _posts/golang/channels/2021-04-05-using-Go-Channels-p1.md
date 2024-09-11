---
title: Using Go Channels - Part 1 - Introduction to Channels
author: g14a
date: 2021-04-06 2:23:00 -0530
categories: [tutorials]
tags: [golang, channels, concurrency]
toc: true
---

## **Introduction**

We know that Goroutines are independently executing user space threads. 

Imagine an office workspace where ten employees are working towards a customer satisfying goal. These ten employees might have similar smaller tasks or issues which they have overcome individually but no one else except them know about. The key point missing here is communication. So to make the best use of their efforts, they might sit in the conference room, communicate with each other to minimize effort and avoid unnecessary wastage of time/resources.

Goroutines could communicate through shared variables, but there's always a question of memory safety and race conditions. That opens up a whole lot of difficulties in the multi threading model. There is an improper solution for that. Each of the shared variables should be locked whenever a goroutine is using it. It should be done either using a ```sync.Mutex``` or using low level ```atomic``` types of Golang. And when businesses scale, the mutexes in the code base become very large and it would be a pain to understand APIs etc. There should be another solution to this i.e **channels**.

So we define a channel in the following way:

**Channels are nothing but a circular queue which implements an inbuilt mutex and they are goroutine-safe. No race conditions occur by design. It can be understood like a pipe where you can read from/write to it. And goroutines communicate through these channels.**

Go's concurrency model says,

> Do not communicate by sharing memory; instead, share memory by communicating.

Which loosely means(from our office space example), 

> Do not save time by working individually; instead, save redundant effort by spending some time communicating.

# **Creating a channel**

Go allows us to create a channel with the ```chan``` keyword. Since Go is a typed language, even channels have types. Once initialized with a type, a channel can only read or write data only of the initialized type.

Let's try creating a channel with no type.

```go
package main

func main() {
    var ch chan
}
```

The above program gives the following error.

```
# command-line-arguments
./main.go:5:1: syntax error: missing channel element type
```

Let's create a channel of type ```int```. We initialize a channel with ```make```.

```go
package main

import "fmt"

func main() {
    var ch chan int
    ch = make(chan int)

    // or just
    c := make(chan int)
}
```

# **Reading and Writing into a channel**

Just how assignment of variables happens from right to left, even passing data to a channel happens from right to left via the `<-` operator.

Writing happens in the following way:

```
channel <- data
```

And Reading happens in the following way:

```
<- channel
```

OR

```
variable <- channel
```

In both cases, the source of any data is on the right hand side, and the destination is on the left.

You can directly initialize the variable via a receive from the channel like this:

```
intVariable := <- channel
```

Go figures out the type of ```intVariable``` based on the type of data coming out from the channel.

# **Communication of Goroutines**

By default, all communication is synchronous and unbuffered. Communication of goroutines using a channel is very much like a human standing with a flower hoping to give it to a loved one. Unless there is someone to receive it, he just exists and blocks everything. 

So, **sends do not complete until there is a reciever to accept data**.

Let's prove this.

```go
package main

func main() {
    ch := make(chan string)

    ch <- "John"
}
```

If we run the above program, we get the following error:

```bash
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /Users/g14a/tutorials/img-proc/main.go:6 +0x50
exit status 2
$ >
```

This happens because there is no goroutine(the main func is also ultimately a goroutine) to read from the channel ```ch``` and the main function has exited. So all goroutines are asleep.

You can understand the above better by the following picture.

<p>
    <img src="../../images/channels/blockinggophers.png" width="60%" alt="">
    <em>Pic Credit: GopherCon UK</em>
</p>

In the above picture, the red buckets are data elements.

* In the first case, the channel is ready with the data, but there's no receiver. So it blocks.
* In the second case, the channel is ready with multiple data elements, but again there's no reciever. So it blocks.
* The third and fourth case represent a receiver(s) sleeping because there is no incoming data. So it blocks. No incoming data means blocking.

Now let's try reading from `ch`.
```go
import "fmt"

func main() {
    ch := make(chan string)

    go greet(ch)
    ch <- "John"
}

func greet(ch chan string) {
    fmt.Println("Hello " + <-ch + "!")
}
```

The above program doesn't print anything. This is not because there is no extra goroutine to read from it. There is `greet` but there is a very little overhead to spawn off a goroutine and by the time `greet` spawns off `main` exits. Now that `main` exits nothing gets printed from the `greet` function. To add that very little overhead we can print a random string with `fmt` or usually `fmt.Scanln()`

```go
import "fmt"

func main() {
    ch := make(chan string)

    go greet(ch)
    ch <- "John"
    fmt.Println("Sent John into channel")
}

func greet(ch chan string) {
    fmt.Println("Hello " + <-ch + "!")
}
```

The above program prints the following:
```bash
Hello John!
Sent John into channel
$ >
```

# **Long Running Goroutines**

Let's simulate a long running goroutine which writes to a channel in a `for` loop and communicate with the main function. We just iterate over the channel like we iterate a slice in Go using `range`

```go
package main

import "fmt"

func main() {
    ch1 := make(chan int)
    go write(ch1)
    for i := range ch1 {
        fmt.Println(i)
    }
}

func write(ch chan int) {
    for i:= 0; i < 10e3; i++ {
        ch <- i
    }
}
```

The above program gives the following output after all the integers upto 10000.

```bash
...
9996
9997
9998
9999
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
$ >
```

This occurs because the `for` loop keeps expecting values from the channel even after it reaches 10000. Just like there's an exit condition for a `for` loop i.e `i < 10e3` there's a `close(channelType)` function which tells Go that no more data will be sent into this channel once it is closed.

A channel is always closed from the sender's end and it is never a good idea to do it on the receiver's end.

Let's try to run this program after adding the `close` condition after the `for` loop.

```go
package main

import "fmt"

func main() {
    ch1 := make(chan int)
    go write(ch1)
    for i := range ch1 {
        fmt.Println(i)
    }
}

func write(ch chan int) {
    for i:= 0; i < 10e3; i++ {
        ch <- i
    }
    close(ch)
}
```

This gives the correct output which is:

```bash
9996
9997
9998
9999
$ > 
```

If data is sent into a closed channel, your goroutine panics. There is no receiver at all, even in the future, to take your data; so your goroutine panics. 

We could easily loop over the channel in a different goroutine like the following as well:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
    ch1 := make(chan int)
    go write(ch1)
    go read(ch1)
    time.Sleep(time.Second)
}
<>
func read(ch chan int) {
    for {
        fmt.Println(<-ch)
    }
}

func write(ch chan int) {
    for i := 0; i < 10e5; i++ {
        ch <- i
    }
    close(ch)
}
```

The above program, prints just `0`s after printing from 1 to 1000000. This happens because once you close the channel and you try to read from it, it keeps giving you empty values for `int` which is basically `0`.

But if you iterate it with `range`, it doesn't print `0`s because `range` internally checks if the channel is closed and breaks the loop.

# **Tricks not covered**

* Reading from a channel returns two values by default out of which:
  * One is the actual data
  * And the other is a boolean value, which represents whether or not the channel is open.

```go
data, ok := <-ch
if !ok {
    fmt.Println("channel closed")
}
```

# **Conclusion**

Hope you enjoyed the first part of the tutorial. In the next one we talk about Buffered Channels.

Check it out at [Part-2 Buffered Channels](https://g14a.github.io/posts/using-Go-Channels-p2/)

Please reach out to me via email(or any social media linked down below) if you think I haven't covered something which you consider important.

Thank you 😁