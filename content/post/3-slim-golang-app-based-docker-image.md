---
title: "Build a slim Go/Golang app docker image"
date: 2019-02-20T18:33:27+01:00
draft: false
tags: [golang, go, docker, slim]
categories : [ "DevOps" ]
image: "img/resources/golang-docker-slim.jpg"
---

Docker images can sometimes take a lot of space, if one of your concerns is the image size, and that can be for many reasons: not enough disk space, limited bandwidth or just a slow speed connection.

In this post we will go through the process of creating a slim Docker image hosting a web service developed in Golang, some parts are applicable to other languages generating similar executables.<!--more-->

The steps for this will be

- Build a statically compiled binary with no external dependecies
- Compress the executable generated to reduce the size more
- Use a small base docker image, in this case we will go with `scratch`

## Creating a simple Go app

First we should start by creating our app, not being the focus of this post we will create the simplest app we can.

A good fit is a simple web-service with a single endpoint that accepts an http GET request with a parameter `name` and return a string `"Hello $name"` or just `"Hello world"` if none are set.

For example

```bash
curl http://localhost:3000/?name=Ahmed
```

Will return

```text
Hello Ahmed
```

Here is the code that will be using for this

```go
package main

import (
  "fmt"
  "net/http"
)

func main() {
  http.HandleFunc("/", hello)
  http.ListenAndServe(":3000", nil)
}

func hello(resp http.ResponseWriter, req *http.Request) {
  names, ok := req.URL.Query()["name"]
  if !ok || len(names[0]) < 1 {
    fmt.Fprint(resp, "Hello world")
    return
  }
  fmt.Fprintf(resp, "Hello %s", names[0])
}
```

## Building the app

First let's go over the steps that we want to do, as our main target is a linux based docker container we will need to cross compile the app if we are in macos/windows, these are the steps

- build a static binary
- rely on go networking instead of using the system's

Now that we have the app code what we need is to generate a static binary, with `go build` this is an easy task, the only thing you should make sure of is the target platform, if you're on a mac for example, as you know docker containers are based on **linux**, so a build for **darwin** will not run inside the container, but this is easily achievable by setting the environment variable `GOOS` to the value `linux`

```bash
GOOS=linux go build -o app.bin app.go
```

But to make sure that our binary is totally static and doesn't have external dependencies, we would have needed to add some flags to the build, like `netgo` to make sure that the binary is using Go own resolvers instead of relying on the system's, thankfully it's no longer necessary to do as of Go **v1.5** on _Unix_ systems (_windows_, _macos_ and _Plan9_ do still need this), the fact that our main target is linux(docker container) we don't need to add this flag.

What we can do though is set `--ldflags` `-extldflags "-static"` to tell the linker to statically compile the dependencies inside the binary.

Run the command

```bash
GOOS=linux go build --ldflags '-extldflags "-static"' -o app.static.bin app.go
```

## Build a smaller app binary

Now that we have cross compiled the app and generated an executable binary, let's do a quick check to the size of all the generated binaries.

Run the command

```bash
ls -lh
```

On my machine this shows

```output
-rwxr-xr-x  1 ezzarghili  staff   6.2M Feb 20 19:14 app.static.bin
```

Now our main goal it reduce the size of the statically compiled binary, the build flag `--ldflags` accept some really usefull args that will help us achieve this goal, these are `-s -w` which allow for stripping symbols table and dwarf data (debug info, _The end of the article will have more information this_) from the binary file.

Run the command

```bash
GOOS=linux go build --ldflags '-extldflags "-static" -s -w' -o app.static.small.bin app.go
```

Let's now check the size of the new genareted binary file

```bash
ls -lh
```

My output

{{<highlight text "hl_lines=2" >}}
-rwxr-xr-x  1 ezzarghili  staff   6.2M Feb 20 19:14 app.static.bin
-rwxr-xr-x  1 ezzarghili  staff   4.6M Feb 20 19:15 app.static.small.bin
{{</highlight>}}

As you can see we were able to reduce the size of the binary generate by roughly ~25% by just using go build flags.

## Reduce the size even more

If we believe that the results we got from the build are still not super satisfactory we can go further by using a really cool tool `upx` the **Ultimate Packer for eXecutables**.

Download the tool here [github release for upx](https://github.com/upx/upx/releases), or you can build it from source if no binary is available for your platform (_for mac users there is a homebrew formula for upx_)

Run the command bellow (-9 is the best compression level, you can use others)

```bash
upx -9 -o app.static.small.upx.bin app.static.small.bin
```

Output

```text
        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
   4822080 ->   1821876   37.78%   linux/amd64   app.static.small.upx.bin
```

Let's check again the size of the generated binaries.

```bash
ls -lh
```

Ouputs

{{<highlight text "hl_lines=1 3" >}}
-rwxr-xr-x  1 ezzarghili  staff   6.2M Feb 20 19:14 app.static.bin
-rwxr-xr-x  1 ezzarghili  staff   4.6M Feb 20 19:15 app.static.small.bin
-rwxr-xr-x  1 ezzarghili  staff   1.7M Feb 20 19:15 app.static.small.upx.bin
{{</highlight>}}

Now we got the initial size of our app roughly **~72%** down!! we can now move on.

## Create a docker image

Now that we have built and compressed our app binary, the next step is to create our docker image hosting the app.

We will use the smallest docker image available to us `scratch` as our app doesn't have any external dependencies.

Also the fact the app is a web-service we will need to export the app port, so it can be discovered and be available to the host.

Here is the simplest `dockerfile` for the the image

```dockerfile
FROM scratch

COPY [ 'app.static.small.upx.bin', 'app.bin' ]

EXPOSE 3000

ENTRYPOINT [ "./app.bin" ]
```

We can build the docker image as usual

```bash
docker build -t slim-go-image .
```

And run it likewise, make sure to map the container port to a host port

```bash
docker run -p 3000:3000 slim-go-image .
```

Let's check the size of our Docker image

```bash
docker image inspect --format='{{.Size}}' slim-go-image
```

Outputs

```text
1821876
```

Voila, our docker image size is roughly around **1.82MB**

## Last words, considerations

It goes without saying that this process should be automated in CI/CD pipeline, also the use of the ldflags `-s -w` will have some side effect on the binary, although this will not affect how the app is executed.

`-s` will disable symbol table which means that you longer can use external tools to look at binary symbol table or lookup functions, for example `go tool nm` will just output something like

```text
reading app.static.small.bin: no symbol section
reading app.static.small.bin: no symbols
```

`-w` will disable DWARF generation, meaning you no long will be able use debug information with tools like `gdb` to debug the binary or `pprof` to profile it, or any other tool that depends on this information be availble in the binary.

Thank you for reading looking forward to write my next post :)