---
title: How I learnt Go Channels - Part 3 - Read/Write Channels & Select
author: g14a
date: 2021-04-10 2:23:00 -0530
categories: [tutorials]
tags: [golang, channels, concurrency]
toc: true
---

## **A Tiny recap**

In the earlier section [Part -2](https://g14a.github.io/posts/how-I-learnt-Go-Channels-p2/) , we have learnt:

* What buffered channels are
* Reading and writing to a buffered channel
* Communicating with goroutines with buffered channels

Have you ever thought of what would happen if you're on the receiver end of a channel and you accidentally close the channel? Or mistakenly try writing on a receiver end or reading from a sender end?

<img src="https://media.giphy.com/media/wQzqIYHE15zMI/giphy.gif" width="340" height="250"/>

To avoid these accidental errors, channels provide a better syntax to indicate whether a channel is just for reading or writing.

## **Read and Write Channels**

First let's try closing a channel on the receiver's end.

```go
package main

import (
    "fmt"
)

func main() {
    ch := make(chan int, 3)

    go send(ch)

    close(ch)

    for {
        val, ok := <- ch
        if ok {
            fmt.Println(val)
        }
    }
}

func send(ch chan int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
}
```

When we run the above program, we get the following error:

```bash
panic: send on closed channel

goroutine 6 [running]:
main.send(0xc000018080)
        /Users/g14a/tutorials/img-proc/main.go:59 +0x45
created by main.main
        /Users/g14a/tutorials/img-proc/main.go:46 +0x5c
exit status 2
```

The error is self explanatory, we're trying to send into a channel which has been closed by the receiver. Now how do we restrict a receiver from closing a channel but not the sender? By passing the `<-` operator while initializing channels or when passing them. 

Let's pass the channel to the `send` function as `ch chan <- int` and seperate the reading of the channel into another function.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int, 3)

    go send(ch)
    go read(ch)

    time.Sleep(time.Millisecond * 100)
}

func send(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
}

func read(ch <-chan int) {
    for {
        val, ok := <-ch
        if ok {
            fmt.Println(val)
        }
    }
}
```

Notice how the parameters of `send` and `read` have `ch chan<- int` and `ch <-chan int` respectively. The syntax means that `send` only writes to a channel and `read` only reads from a channel.

Now even if you try to close the channel in the `read` function before the `for` loop starts, it throws a syntax error like this.

```bash
# command-line-arguments
./main.go:60:7: invalid operation: close(ch) (cannot close receive-only channel)
$ >
```

Try writing into a channel in `read` now. And then try reading from the channel in the `send` function and see the output for yourself.

```go
func read(ch <-chan int) {
    ch <- 10
    for {
        val, ok := <-ch
        if ok {
            fmt.Println(val)
        }
    }
}
```

It again throws a syntax error:

```bash
# command-line-arguments
./main.go:60:5: invalid operation: ch <- 10 (send to receive-only type <-chan int)
$ >
```

Now that we've understood the read/write channels and how we can avoid pitfalls and errors, lets move forward.

As business and systems scale, there wouldn't be just one goroutine in the application. There might be hundreds of them or even thousands. And these goroutines might be communicating between themselves via multiple channels. And sometimes when one goroutine blocks, the others keep running. So it implies that if one channel is waiting for data to come in, other channels already have data in it and are in need of a receiver i.e they are ready to deliver data. So is there a way we can use whatever data is incoming from whichever channel instead of waiting for whether or not the others have data? Absolutely.

## **Select**

The `select` keyword lets you get values out of simultaneously running goroutines. It is like a `switch` statement but for channel operations. There's also a `default` case in `select` just like in a `switch` statement.

The `select` statement does the following:

* Switches on channel operations to see which one can proceed immediately.
* If multiple channel operations aren't blocking, it choses one of them at random. 
* If all channel operations are blocking, it waits until one of them isn't.
* If none of the channel operations can proceed, and there is a `default` case, it executes the `default` case.

The `select` keyword is generally paired with the `for` loop in Go. We either loop infinitely if we have no clue about how much data we get into our channels, or in a `for range` loop.

Let's see how `select` works in the following program. We initialize two unbuffered channels `helloChan` and `worldChan` and pass data into it in two seperate goroutines. Notice how we didn't use a `for` loop here because we know that only one data element is going to be received by the channel.

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

Upon running several times, we see `Hello` or `World!` in the output. This is because both of the channel operations at line 16 and 18 are non blocking. So any one of them is selected at random.

Now let's try blocking one of them by adding a `time.Sleep` before sending data into `helloChan`. Our `hello` functions becomes this.

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

This can either print out `Hello` or `World!`. Now its upto you, to try adding a blocking call by making one of the goroutines sleep for a while and explore all the combinations.

Let's try adding a default case and we see that it does something interesting. Our `select` statement becomes this:

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

In the third and fourth point of our `select` introduction we've seen that: 

> If all channel operations are blocking, it waits until one of them isn't.

> If none of the channel operations can proceed, and there is a `default` case, it executes the `default` case.

So in the previous example, there is no `default` case. So `select` waited for atleast one of them to happen and executed that case immediately. But now since the `default` case exists, `select` doesn't wait anymore and proceeds to run the default case. Try adding a sleep just before select and see what happens for yourself. That's an exercise for you.

Let's see a real use case of `select`.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
    helloChan := make(chan string, 5)
    worldChan := make(chan string, 5)

    go hello(helloChan)
    go world(worldChan)

    for {
        start := time.Now()
        fmt.Println(<-helloChan)
        fmt.Println(<-worldChan)
        fmt.Println(time.Since(start))
    }
}

func hello(helloChan chan<- string) {
    for {
        time.Sleep(time.Second*1)
        helloChan <- "Hello"
    }
}

func world(worldChan chan<- string) {
    for {
        time.Sleep(time.Millisecond*500)
        worldChan <- "World!"
    }
}
```

When we run this we get the following output:

```bash
Hello
World!
1.002727246s
Hello
World!
1.002881771s
Hello
World!
1.003041016s
Hello
World!
1.004952019s
...
$ >
```

We see that every 1 second, we get both `Hello` and `World!` gets printed. But that's not what we need. We need `World!` to be printed twice every one second because we're waiting only 500ms before sending data into `worldChan`.

So finally, by the end of 1 sec, we need `Hello` to be printed once, and `World!` to be printed twice. But this doesn't happen because, the read at line no. 17 i.e `<-helloChan` is a blocking one. It blocks until the one second gets over at line no. 25, and then it proceeds. Since it blocks, `<-worldChan` doesn't get called.

Now lets replace the `for` loop with a for-select loop.

```go
for {
    select {
    case msg := <-helloChan:
        fmt.Println(msg)
    case msg := <-worldChan:
        fmt.Println(msg)
    }
}
```

Now when we run this, we get the desired output:

```bash
World!
Hello
World!
World!
Hello
World!
World!
Hello
World!
...
$ >
```

The control flow can be understood better by the following picture.

<p>
    <img src="../../images/channels/select.png" width="60%">
</p>

## **Timeouts in channels**

In a HTTP environment, we talk about request timeouts. Where, if a request takes more than a certain period of time, we return a `408`. Similarly we have something in channels where if a channel operation takes more than a certain period, we execute something.

You can imagine the usage of a timeout in a channel, where after time `t`, we send a message onto the channel saying "Hey, its time we did something else instead of waiting."

A basic timeout implementation can be done by the following

```go
package main

import (
	"fmt"
	"time"
)

func main() {
    helloChan := make(chan string, 5)
    worldChan := make(chan string, 5)
    timeoutChan := make(chan bool, 1)

    go hello(helloChan)
    go world(worldChan)

    go func() {
        time.Sleep(time.Second*3)
        timeoutChan <- true
    }()

    for {
        select {
        case msg := <-helloChan:
            fmt.Println(msg)
        case msg := <-worldChan:
            fmt.Println(msg)
        case <-timeoutChan:
            fmt.Println("It has been 3 seconds. Timeout!")
        }
    }
}
```

When you run this, the program prints `It has been 3 seconds. Timeout!` every three seconds.

Try blocking the `hello` and `world` functions for more than 3 seconds each, and add a return in the timeout case.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    helloChan := make(chan string, 5)
    worldChan := make(chan string, 5)
    timeoutChan := make(chan bool, 1)

    go hello(helloChan)
    go world(worldChan)

    go func() {
        time.Sleep(time.Second*3)
        timeoutChan <- true
    }()

    for {
        select {
        case msg := <-helloChan:
            fmt.Println(msg)
        case msg := <-worldChan:
            fmt.Println(msg)
        case <-timeoutChan:
            fmt.Println("It has been 3 seconds. Timeout!")
            return
        }
    }
}

func hello(helloChan chan<- string) {
    for {
        time.Sleep(time.Second*5)
        helloChan <- "Hello"
    }
}

func world(worldChan chan<- string) {
    for {
        time.Sleep(time.Second*5)
        worldChan <- "World!"
    }
}
```
Try running this and see the output for yourself.

We used the `timeoutChan` for demonstration purposes but Go has something neat exactly for this functionality where we don't have to make our own timeout channels. It's the `time.After()` function that returns a channel and sends on it, after a certain period of time.

[https://golang.org/pkg/time/#After](https://golang.org/pkg/time/#After)

The above example can be converted to the following and that's all we have to do.

```go
for {
    select {
    case msg := <-helloChan:
        fmt.Println(msg)
    case msg := <-worldChan:
        fmt.Println(msg)
    case <-time.After(time.Second*3):
        fmt.Println("It has been 3 seconds. Timeout!")
        return
    }
}
```

## **Conclusion**

Hope you enjoyed the third part of the tutorial.
In the next one we see a real world example of how we use channels.

Check it out at [Part-4 Image Processing with channels](https://g14a.github.io/posts/how-I-learnt-Go-Channels-p3/)


Please reach out to me via email(or any social media linked down below) if you think I haven't covered something which you consider important.

Thank you 😁