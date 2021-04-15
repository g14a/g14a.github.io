---
title: How I learnt Go Channels - Part 4 - Image Processing with Channels - Part 2
author: g14a
date: 2021-04-15 7:30:00 -0530
categories: [tutorials]
tags: [golang, channels, concurrency, exercise]
toc: true
---

## **A Tiny Recap**

In the earlier section [Part - 4](https://g14a.github.io/posts/how-I-learnt-Go-Channels-p4/), we have seen the serial and parallel versions of our image processing program.

Now let us see if we can make our parallel version even better.

<p>
    <img src="https://media.giphy.com/media/11zU4IDJeg75Sg/giphy.gif" width="430" height="250"/>
    <em>Hope you're as excited as Kevin!</em>
</p>

## **Worker Pool**

If we take a look at our previous main function, we see that we read the image and then pass it to the `Save` function. So until we read the image and prepare the `Image` type, the `Save` function blocks and the next image isn't prepared until then.

What if we do this asynchronously in a goroutine, and have a channel which gives a steady input of images? Let's see what happens.

Let's create another function `inputImages` which just takes in a `sync.WaitGroup`, assigns it to all the images we create and returns a `<-chan Image`. 

If we haven't discussed about channels being initialized with custom structs, then this is a good example. We can.

```go
func inputImages(wg *sync.WaitGroup) <-chan Image {
    images := make(chan Image)

    go func(wg *sync.WaitGroup) {
        for i := 1; i < 3000; i++ {
            filePath := fmt.Sprintf("/Users/g14a/Downloads/train/documents/%d.png", i)
            outputFilePath := fmt.Sprintf("/Users/g14a/Downloads/train/documents/output/%d.png", i)

            img, err := imgio.Open(filePath)
            if err != nil {
                fmt.Println(err)
            }

            rImg := Image{
                InputPath:  filePath,
                OutputPath: outputFilePath,
                Img:        img,
                Wg:         wg,
            }
            images <- rImg
        }
        close(images)
    }(wg)

    return images
}
```

Notice we `close()` the `images` channel after the `for` loop. 

We now implement a worker pool of 100 workers, which take images returned by the `inputImages` function and start saving images.

```go
func worker(imgChan <-chan Image, wg *sync.WaitGroup, workerWg *sync.WaitGroup) {
	defer workerWg.Done()
	for img := range imgChan {
		wg.Add(1)
		go img.save()
	}
}
```

Now our main function becomes much simpler.

```go
func main() {
    var wg, workerWg sync.WaitGroup

    images := inputImages(&wg)

    for i := 0; i < 100; i++ {
        workerWg.Add(1)
        go worker(images, &wg, &workerWg)
    }

    workerWg.Wait()

    wg.Wait()
    fmt.Println("Time elapsed", time.Since(start))
}
```

What happens here is, `inputImages` creates a goroutine, creates a channel of images just ready to be processed. Now, there is nothing blocking a second image to be initialized until the first image is prepared. 

Let's see how this program performs.

```bash
$ > time ./img-proc
2021/04/15 22:32:36 profile: memory profiling enabled (rate 1), mem.pprof
Time elapsed 2m38.151913114s
2021/04/15 22:35:14 profile: memory profiling disabled, mem.pprof
./img-proc  1666.75s user 15.45s system 1058% cpu 2:38.94 total
```

This looks a lot better. We have come down from `3m34s` from the earlier section to `2m38s` now.

## **Memory Profile**

Let's add `pprof` to our Go program and see the memory profile by running it again and check the flame graph.

<p>
    <img src="../../images/channels/memprof.png" width="100%">
</p>

We observe the following from the graph and our code in general:

* `imgio.PNGEncoder()` is being allocated again and again.
* `filePath`, `outputFilePath` are initialized for every image.
* `img`(line no 9) and `rImg`(line no 14) are initialized for every image.
* Printing known logs i.e `fmt.Println()` increased I/O overhead to print on the terminal when data set is as large 3k images.
* We don't need `InputPath` in our `Image` type unless it is logged.

We make modifications to decrease memory allocation and then our final program is updated as per the next heading.

## **Final Parallel Version**

This is how our final program looks like:

```go
package main

import (
    "fmt"
    "github.com/anthonynsimon/bild/adjust"
    "github.com/anthonynsimon/bild/imgio"
    "image"
    "sync"
    "time"
)

func main() {
    start := time.Now()
    var wg sync.WaitGroup
    images := inputImages(&wg)

    var workerWg sync.WaitGroup
    encoder := imgio.PNGEncoder()

    for i := 0; i < 100; i++ {
        workerWg.Add(1)
        go worker(images, &wg, &workerWg, encoder)
    }

    workerWg.Wait()
    wg.Wait()
    fmt.Println("Time elapsed", time.Since(start))
}

func worker(imgChan <-chan Image, wg *sync.WaitGroup, workerWg *sync.WaitGroup, encoder imgio.Encoder) {
    defer workerWg.Done()
    for img := range imgChan {
        wg.Add(1)
        go img.save(encoder)
    }
}

func inputImages(wg *sync.WaitGroup) <-chan Image {
    images := make(chan Image)

    var filePath, outputFilePath string
    var img image.Image
    var rImg Image
    var err error

    go func(wg *sync.WaitGroup) {
        for i := 1; i < 3000; i++ {
            filePath = fmt.Sprintf("/Users/g14a/Downloads/train/documents/%d.png", i)
            outputFilePath = fmt.Sprintf("/Users/g14a/Downloads/train/documents/output/%d.png", i)

            img, err = imgio.Open(filePath)
            if err != nil {
                fmt.Println(err)
                return
            }

            rImg = Image{
                OutputPath: outputFilePath,
                Img:        img,
                Wg:         wg,
            }
            images <- rImg
        }
        close(images)
    }(wg)

    return images
    }

    func (i Image) save(encoder imgio.Encoder) {
    defer i.Wg.Done()
    brightnessDone, contrastDone := make(chan bool), make(chan bool)

    go i.brightness(brightnessDone, contrastDone)

    <-brightnessDone
    <-contrastDone

    if err := imgio.Save(i.OutputPath, i.Img, encoder); err != nil {
        fmt.Println(err)
        return
    }
}

func (i Image) brightness(brightnessDone chan<- bool, contrastDone chan<- bool) {
    i.Img = adjust.Brightness(i.Img, 0.25)
    brightnessDone <- true

    go i.contrast(contrastDone)
}

func (i Image) contrast(contrastDone chan<- bool) {
    i.Img = adjust.Contrast(i.Img, 0.25)
    contrastDone <- true
}

type Image struct {
    OutputPath string
    Img        image.Image
    Wg         *sync.WaitGroup
}
```

We get the following output when we run this:

```bash
$ > time ./img-proc
Time elapsed 2m37.824414114s
./img-proc  1584.74s user 11.74s system 1008% cpu 2:38.37 total
```

Oops, it didn't get any better! But we can see that the CPU time came down from 1058% to 1008% so that counts I guess?