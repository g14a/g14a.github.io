---
title: Channels and Deadlocks in Go with a File crawler
author: g14a
date: 2021-11-27 3:30:00 -0530
categories: [tutorials]
tags: [golang, channels, concurrency, exercise, performance]
toc: true
---

<p>
    <img src="https://media.giphy.com/media/EbRPam1A4jEWkUokL8/giphy.gif" width="250" height="250" alt=""/>
</p>

Welcome back! We've seen in the earlier sections that channels can be used to improve and control concurrency in Golang. But let's look at it again with a fresh mind. Let's unlearn things and re-learn them again.

## **Assumptions**
Going forward I assume that you have basic knowledge of Go syntax and how to write Go programs/build them. Even though we're re-learning things I assume that you haven't literally hit your head and forgotten basic Golang.

## **Creating Channels**

We create channels by either initializing them or assigning them.

```go
intChan := make(chan int)
```

or 

```go
var intChan chan int
intChan = make(chan int)
```

We send data into the channel by using the `<-` operator

```go
intChan <- 10
```

And we receive data using the same operator but like this

```go
<- intChan
//or
a := <-intChan
```

## **Let's write a program**

We're just trying to insert an integer into an int channel and trying to get it out of it. And then print it. This should work right?


```go
var ic chan int
ic = make(chan int)

ic <- 10
a := <- ic

fmt.Println(a)
```

This throws out the famous deadlock error!

```bash
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
        /Users/g14a/main.go:12 +0x37
exit status 2
```

Someone on the internet compared a channel to a portal. And I just never forget that. If something goes inside a portal, it should be able to exit the portal simultaneously. Not before or after, but exactly at the same time.

So for two things to happen at the time, we need two functions running at the same time. Which can be achieved by nothing but goroutines.

The program throws the deadlock error in the following cases:

* When data is going into it, but not exiting it.
* When data is not going into it, but the program tries to extract data from it.
* When data is going in and coming out, but both are happening at different times and are not in sync.
  
So now let's change our code to 

```go
var ic chan int
ic = make(chan int)

go func() {
    ic <- 10
}()
a := <- ic

fmt.Println(a)
```

Now it prints out 10! Voila!

Have you tried this variant though? 

```go
var ic chan int
ic = make(chan int)

a := <- ic

go func() {
    ic <- 10
}()

fmt.Println(a)
```

Spoiler alert! It breaks. Figure it out. Nah, just kidding. It's because line 4 is a blocking operation until it gets a value on the channel. So line 6 never gets executed in the first place raising a deadlock again!

## **Buffered Channels**
 
We've seen the scenario where the program throws a deadlock error. So let's try the following.

```go
func main() {
    var ic chan int
    ic = make(chan int, 2)

    ic <- 1
    ic <- 2

    a := <-ic

    fmt.Println(a)
}
```

Wait a second! How come this prints `1`? It's because we specified the channel length to be `2` in line 3. We call these channels as `Buffered Channels`. In the first scenario we haven't specified a length. So the channel doesn't store any values if there's no one to receive it from the other end. But here it can store a maximum of two values even without a receiver. 

What if want to reproduce the deadlock error even with buffered channels? Is there a way?
Yup, there's more than one way.

* Pushing more data into the channel breaks the program
* Retrieving more data than what's present in the channel also breaks.

Let's have fun with this.

```go
func main() {
    var ic chan int
    ic = make(chan int, 2)

    ic <- 1
    ic <- 2
    ic <- 3
    a := <-ic

    fmt.Println(a)
}
```

The channel length is 2, but you're sending 3 values into it. Nope. Breaks!

```go
func main() {
    var ic chan int
    ic = make(chan int, 2)

    ic <- 1

    a := <-ic
    b := <-ic
    fmt.Println(a)
    fmt.Println(b)
}
```

You've sent only one value into it but you're trying to extract two values from it? Straight to jail! How dare ya?

So we can casually tell the difference between buffered and unbuffered channels.

Unbuffered | Buffered
--- | --- | --- | --- |--- |--- |--- |--- |--- |--- |--- |---
Don't underdo things. | Don't overdo things. 

## **File Crawler**

Let's write a file crawler which counts the number of files(recursively in sub directories as well) and gives us the disk usage of all of them.

```go
package main

import (
	"flag"
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"
	"time"
)

func main() {
    flag.Parse()

    roots := flag.Args()
    if len(roots) == 0 {
        roots = []string{"."}
    }

    now := time.Now()

    var totalFiles, totalBytes int64

    for _, root := range roots {
        files, bytes := walkDirectory(root)
        totalFiles += files
        totalBytes += bytes
    }

    fmt.Println("total time: ", time.Since(now))
    printUsage(totalFiles, totalBytes)
}

func printUsage(files, bytes int64) {
	fmt.Printf("%d files %.1f GB\n", files, float64(bytes)/1e9)
}

func walkDirectory(dir string) (noOfFiles, totalSize int64) {
    for _, entry := range getEntries(dir) {
        if entry.IsDir() {
            subDir := filepath.Join(dir, entry.Name())
            files, bytes := walkDirectory(subDir)
            noOfFiles += files
            totalSize += bytes
        } else {
            noOfFiles++
            e, err := entry.Info()
			if err != nil {
				panic(err)
			}
            totalSize += e.Size()
        }
    }
    return
}

func getEntries(dir string) []os.DirEntry {
    entries, err := os.ReadDir(dir)
    if err != nil {
        panic(err)
    }
    return entries
}
```

Now the above program doesn't implement concurrency, its just plain functions and single threaded. The code is straight forward.

* We take one of more directories.
* We loop on each one of them.
* Get entries of each directory, recursively call the `walkDirectory` function if an entry is a directory, else we interpret it as a file and increase the count and size.

Let's test our program on a large directory like the [Linux operating system source tree](https://github.com/torvalds/linux).

The source contains 70k files and a total size of 5GB.

```bash
$ go run two-sum/main.go ~/linux
total time:  2.827069181s
74319 files 5.0 GB
```

Huh! 2 seconds isn't that bad for almost 75k files. But it looks like we can optimise it. The only area of optimisation is we can make each directory operation independent of another and this should get a little better.

## **Optimisation**

Let's refactor our `walkDirectory` function and see how it looks.

```go
func walkDirectory(dir string, wg *sync.WaitGroup, fileSize chan<- int64) {
    defer wg.Done()
        for _, entry := range getEntries(dir) {
            if entry.IsDir() {
                wg.Add(1)
                subdir := filepath.Join(dir, entry.Name())
                go walkDirectory(subdir, wg, fileSize)
            } else {
                info, err := entry.Info()
                if err != nil {
                    panic(err)
                }
                fileSize <- info.Size()
            }
        }
}
```

We will be calling this function as a goroutine in the main function so we need to keep track of all of them. A `sync.WaitGroup` exists just for that purpose. So we add a `defer wg.Done()` in the beginning in case we forget to decrement our counters in the end.

In our serial program we had a counter and it was incremented as and when we encountered a file. But here, we send each file info into a channel of type `int` on line 13. The reason we did this is simple. Every single value that goes into the channel means a file has been come across, because we don't calculate the size of the same file more than once.

I assume we remember the syntax of a write-only channel which is `chan<- int64`

Soon our main function becomes this.

```go
func main() {

    roots := flag.Args()
    if len(roots) == 0 {
        roots = []string{"."}
    }

    now := time.Now()

    fileSizes := make(chan int64)
    var wg sync.WaitGroup

    for _, root := range roots {
        wg.Add(1)
        go walkDirectory(root, &wg, fileSizes)
    }

    wg.Wait()
    close(fileSizes)

    var nFiles, nBytes int64

    for size := range fileSizes {
        nFiles++
        nBytes += size
    }

    fmt.Println("total time: ", time.Since(now))
    printUsage(nFiles, nBytes)
}
```

We wait for all the goroutines to end at line 18 and we close the channel once all the goroutines have completed their work. Now let's see the output of this.

```bash
panic: open /Users/g14a/linux/arch/arc/boot/dts: too many open files

goroutine 1142 [running]:
main.getEntries(...)
	/Users/g14a/main.go:78
main.walkDirectory({0xc000672b70, 0x23}, 0xc0000141c0, 0xc0000240c0, 0xc000024120)
	/Users/g14a/main.go:58 +0x299
created by main.walkDirectory
	/Users/g14a/main.go:62 +0x225
exit status 2
```

<p>
    <img src="https://media.giphy.com/media/e1s8C0YnnfjlRf7mEr/giphy.gif" width="430" height="250" alt=""/>
</p>

The reason this happens in we're concurrently accessing a lot of files which was past our `open file descriptor limit`. So we don't want to do that. Neither do we want to increase our maximum limit to really 75k files. 

We only want our program to access a certain number of files at a time. So we change our `getEntries` function because that's where all the reading is happening.

```go
func getEntries(dir string, sema chan struct{}) []os.DirEntry {
    sema <- struct{}{}
    defer func() { <-sema } ()
    entries, err := os.ReadDir(dir)
    if err != nil {
        panic(err)
    }
    return entries
}
```

What we're doing here is basically send in an empty struct to a channel, do our file operation and then pop the empty struct via defer at the end. Remember semaphores?!

Now we want to keep this configurable at run time so let's add a command line flag called `-p` to get this value via stdin.

The main function becomes:

```go
func main() {

    flag.IntVar(&maxConcurrency, "p", 1, "")
    flag.Parse()

    roots := flag.Args()
    if len(roots) == 0 {
        roots = []string{"."}
    }

    now := time.Now()

    fileSizes := make(chan int64)
    var wg sync.WaitGroup

    var sema = make(chan struct{}, maxConcurrency)

    for _, root := range roots {
        wg.Add(1)
        go walkDirectory(root, &wg, fileSizes, sema)
    }

    wg.Wait()
    close(fileSizes)

    var nFiles, nBytes int64

    for size := range fileSizes {
        nFiles++
        nBytes += size
    }

    fmt.Println("total time: ", time.Since(now))
    printUsage(nFiles, nBytes)
}

```

<span style="color:#FF02c4"><b>The reason we added a size to the channel on `line 16` is to allow `maxConcurrency` number of files to be access by goroutines. It is also the reason why we did not pop empty structs from the channel in the `getEntries` function in a goroutine. Remember the portal example. Buffered channels don't act like portals. Only non-buffered channels do.</b></span>

Let's run this program by passing a value of 20 to the `-p` flag.

```bash
$ go run main.go -p 20 ~/linux
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_Semacquire(0x0)
	/usr/local/go/src/runtime/sema.go:56 +0x25
sync.(*WaitGroup).Wait(0xc0000261e0)
	/usr/local/go/src/sync/waitgroup.go:130 +0x71
main.main()
	/Users/g14a/main.go:37 +0x177

goroutine 6 [chan send]:
main.walkDirectory({0x7ff7bfeff418, 0x11}, 0xc0000141c0, 0xc0000240c0, 0xc000024120)
	/Users/g14a/main.go:68 +0x25c
created by main.main
	/Users/g14a/main.go:33 +0x35f
exit status 2
```

Huh! Any idea why this is happening? 

If we look at our `fileSizes` channel, we know its a non-buffered channel. Our waitgroup on line 23 expects all goroutines to continue and move on. But our non-buffered channel is a portal. For the `walkDirectory` goroutine to move on, it needs someone taking values out of the `fileSizes` channel. But we're reading from the channel after we `wg.Wait()` and `close(fileSizes)`

Two things are happening here:

* The goroutine doesn't move on to the next directory because channel is never read.
* The values from channel are never read since the goroutine doesn't move on and decrement its counter(which is basically what `wg.Wait()` wants).

So we get a deadlock situation.

We just need to put the `wait` and `close` in a goroutine and our job is done.

Let's see how our program performs with `-p` as 1.

```bash
$ go build -o parallel main.go
$ ./parallel -p 1 ~/linux                         
total time:  416.093619ms
74319 files 5.0 GB
```

Our serial version took 2.8 seconds. The parallel version is approximately 7 times faster already. But let's look at what happens when we start increasing the value of `-p`.

```bash
$ ./parallel -p 2 ~/linux 
total time:  268.323046ms
74319 files 5.0 GB
```

```bash
$ ./parallel -p 3 ~/linux
total time:  201.106361ms
74319 files 5.0 GB
```

```bash
$ ./parallel -p 4 ~/linux 
total time:  177.577326ms
74319 files 5.0 GB
```

We made it around 16 times faster than the serial version!

<p>
    <img src="https://media.giphy.com/media/l0amJzVHIAfl7jMDos/giphy.gif" width="430" height="380" alt=""/>
    <em>By now you can tell that I'm an Office fan.</em>
</p>

## **Conclusion**

Pat yourself on the back, its been a hell of a ride!

But at some point the performance gets saturated. I'm going to leave that to you to play around and get the maximum performance on your machine.

You can have a look at my final program here - [Github Gist](https://gist.github.com/g14a/277337b834f3568a4c963c92d4db4c11)

Let me know if I can improve better or any areas where I have gone wrong.

Please reach out to me via email(or any social media linked down below) if you think I haven’t covered something which you consider important.

Thank you 😁