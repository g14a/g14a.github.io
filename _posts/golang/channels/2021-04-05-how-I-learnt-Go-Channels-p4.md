---
title: How I learnt Go Channels - Part 4 - Image Processing with Channels
author: g14a
date: 2021-04-15 7:30:00 -0530
categories: [tutorials]
tags: [golang, channels, concurrency, exercise]
toc: true
---

## **A Tiny Recap**

In the earlier section [Part - 3](https://g14a.github.io/posts/how-I-learnt-Go-Channels-p3/) , we have learnt:

* What read and write specific channels are
* How we use select in the context of goroutines
* Finally, how we apply timeouts in select

In this section, we apply whatever we have learnt about channels until now and perform a real world task.

## **Exercise**

In this exercise we do a little image manipulation using the following Go library
 
[Bild - Image processing algorithms in pure Go ](https://github.com/anthonynsimon/bild)

to manipulate our images. We take a large dataset of images of size 3k, and apply changes to each image and save them in a different directory. The task is simple, however implementing it is where we are going to be at.

The flow of manipulation is going to be like the following:

* Read an image
* Change the brightness of it.
* Change the contrast of it.
* Save the result in a new file.

## **Image Type**

We create our custom Image:

```go
type Image struct {
    InputPath  string
    OutputPath string
    Img        image.Image
    Wg         *sync.WaitGroup
}
```

We have a `sync.WaitGroup` in our image because each image is going to be processed concurrently, so I thought it is better to embed it in the Image type itself. You are welcome to implement them however you like.

## **Brightness Func**

The function to change the brightness of our image is going to be like the following:

```go
func (i Image) brightness(brightnessDone chan<- bool, contrastDone chan<- bool) {
    i.Img = adjust.Brightness(i.Img, 0.25)

    fmt.Println("Brightness of ", i.InputPath, " changed by 0.25")
    brightnessDone <- true

    go i.contrast(contrastDone)
}
```

The function has two arguments:

* `brightnessDone chan<- bool`
* `contrastDone chan<- bool`

We've declared it as `chan<- bool` because both of them are going to receive data into them.

In line no.5, we send a `true` into the channel once we have adjusted the brightness of the image.

And once we have adjusted the brightness, our next plan is to change the contrast.

## **Contrast Func**

The function to change the contrast of our image is going to be like the following:

```go
func (i Image) contrast(contrastDone chan<- bool) {
    i.Img = adjust.Contrast(i.Img, 0.25)
    fmt.Println("Contrast of ", i.InputPath, " changed by 0.25")
    contrastDone <- true
}
```

The contrast function has only one parameter `contrastDone chan<- bool` because there is nothing more to be done on the image other than saving it. So we just send `true` into it once we're done with it. 

<span style="color:#FF0266"><b>Notice how we spawned off a goroutine in the brightness function to start the modification of contrast immediately.</b></span>

## **Save Func**

We need to save an image only when we are sure that both brightness and contrast are adjusted. How do we keep track of them? Using the `brightnessDone` and `contrastDone` channels.

```go
func (i Image) Save() {
    defer i.Wg.Done()
    brightnessDone, contrastDone := make(chan bool), make(chan bool)

    go i.brightness(brightnessDone, contrastDone)

    <-brightnessDone
    <-contrastDone

    if err := imgio.Save(i.OutputPath, i.Img, imgio.PNGEncoder()); err != nil {
        fmt.Println(err)
        return
    }
}
```

In the `Save` function, we see that we create the two channels required to make sure both brightness and contrast of an image are completed. Here don't explicity call the `contrast` function, because it will be called automatically once `brightness` finishes asynchronously.

Why did we not create any waitgroups for the brightness and contrast functions, you may ask. There are two types of waiting for goroutines.

* Create a waitgroup and pass its address to goroutines and call `defer wg.Done()`.
* Pass data into a channel inside the goroutine and wait for data from the channel in the parent function.

Since we're learning about channels, we prefer the second method.

## **Run**

So our program on the whole becomes the following:

```go
package main

import (
    "fmt"
    "github.com/anthonynsimon/bild/adjust"
    "github.com/anthonynsimon/bild/imgio"
    "github.com/pkg/profile"
    "image"
    "sync"
    "time"
)

func main() {
    var wg sync.WaitGroup

    start := time.Now()
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
            Wg:         &wg,
        }
        wg.Add(1)
        go rImg.Save()
    }
    wg.Wait()
    fmt.Println("Time elapsed", time.Since(start))
}

func (i Image) Save() {
    defer i.Wg.Done()
    brightnessDone, contrastDone := make(chan bool), make(chan bool)

    go i.brightness(brightnessDone, contrastDone)

    <-brightnessDone
    <-contrastDone

    if err := imgio.Save(i.OutputPath, i.Img, imgio.PNGEncoder()); err != nil {
        fmt.Println(err)
        return
    }
}

func (i Image) brightness(brightnessDone chan<- bool, contrastDone chan<- bool) {
    i.Img = adjust.Brightness(i.Img, 0.25)

    fmt.Println("Brightness of ", i.InputPath, " changed by 0.25")
    brightnessDone <- true

    go i.contrast(contrastDone)
}

func (i Image) contrast(contrastDone chan<- bool) {
    i.Img = adjust.Contrast(i.Img, 0.25)
    fmt.Println("Contrast of ", i.InputPath, " changed by 0.25")
    contrastDone <- true
}

type Image struct {
    InputPath  string
    OutputPath string
    Img        image.Image
    Wg         *sync.WaitGroup
}
```

The flow of the program can be better understood by the following illustration:

<p>
    <img src="../../images/channels/goroutines.png" width="100%">
</p>

When you run the above program, we see the following output:

```bash
$ > time ./img-proc
Brightness of  /Users/g14a/Downloads/train/documents/1.png  changed by 0.25
Contrast of  /Users/g14a/Downloads/train/documents/1.png  changed by 0.25
Brightness of  /Users/g14a/Downloads/train/documents/2.png  changed by 0.25
Contrast of  /Users/g14a/Downloads/train/documents/2.png  changed by 0.25
...
...
Brightness of  /Users/g14a/Downloads/train/documents/2998.png  changed by 0.25
Contrast of  /Users/g14a/Downloads/train/documents/2998.png  changed by 0.25
Brightness of  /Users/g14a/Downloads/train/documents/2999.png  changed by 0.25
Contrast of  /Users/g14a/Downloads/train/documents/2999.png  changed by 0.25
Time elapsed 3m34.121020904s
2021/04/15 18:27:23 profile: cpu profiling disabled, cpu.pprof
./img-proc  2235.90s user 25.88s system 1054% cpu 3:34.45 total
$ >
```

The parallel version has taken around 3 and a half minutes long to process 3000 images.

The serial version is pretty simple and let's see how long this has taken:

```go
func main() {
    start := time.Now()
    for i := 1; i < 10000; i++ {
    filePath := fmt.Sprintf("/Users/g14a/Downloads/train/documents/%d.png", i)
    outputFilePath := fmt.Sprintf("/Users/g14a/Downloads/train/documents/output/%d.png", i)

        img, err := imgio.Open(filePath)
        if err != nil {
            fmt.Println(err)
            return
        }
        img = adjust.Brightness(img, 0.25)
        fmt.Println("Brightness of ", filePath, " changed by 0.25")
        img = adjust.Contrast(img, 0.25)
        fmt.Println("Contrast of ", filePath, " changed by 0.25")

        if err := imgio.Save(outputFilePath, img, imgio.PNGEncoder()); err != nil {
            fmt.Println(err)
            return
        }
    }
    fmt.Println("Time elapsed", time.Since(start))
}
```

We see that the serial version takes around 19 and a half minutes to process the 3000 images.

```bash
$ > time ./img-proc
Brightness of  /Users/g14a/Downloads/train/documents/1.png  changed by 0.25
Contrast of  /Users/g14a/Downloads/train/documents/1.png  changed by 0.25
Brightness of  /Users/g14a/Downloads/train/documents/2.png  changed by 0.25
Contrast of  /Users/g14a/Downloads/train/documents/2.png  changed by 0.25
...
...
Brightness of  /Users/g14a/Downloads/train/documents/2998.png  changed by 0.25
Contrast of  /Users/g14a/Downloads/train/documents/2998.png  changed by 0.25
Brightness of  /Users/g14a/Downloads/train/documents/2999.png  changed by 0.25
Contrast of  /Users/g14a/Downloads/train/documents/2999.png  changed by 0.25
Time elapsed 19m39.104002641s
./img-proc  1447.34s user 45.19s system 126% cpu 19:39.62 total
$ >
```

The results may vary on your machine.

I'm using a MacBook Pro, 16 GB 2667 MHz DDR4, 2.6 GHz 6-Core Intel Core i7.

Just by using basic channels and goroutines we've managed to speed up the task by more than 5x.

## **Conclusion**

Hope you enjoyed this exercise. In the next one we see if we can optimise this a little bit more and see what we can come up with.

Check it out at [Part-4 Image Processing with channels - 2](https://g14a.github.io/posts/how-I-learnt-Go-Channels-p5/)

Please reach out to me via email(or any social media linked down below) if you think I haven’t covered something which you consider important.

Thank you 😁