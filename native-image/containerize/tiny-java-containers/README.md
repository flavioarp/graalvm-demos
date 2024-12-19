# Tiny Java Containers

This demo shows how a simple Java application and a simple web
server can be compiled to produce very small Docker container images. 

The smallest container images contains just an executable. But since there's
nothing in the container image except the executable, including no `libc` or other
shared libraries, an executable has to be fully statically linked with all
needed libraries and resources.

To support static linking of `libc`, GraalVM Native Image supports using the
"lightweight, fast, simple, free" [musl](https://musl.libc.org/) `libc`
implementation.

You can watch a [Devoxx 2022](https://devoxx.be/) session that walks through
an earlier version of this example on YouTube. 

[![A 1.5MB Java Container
App](images/youtube.png)](https://youtu.be/6wYrAtngIVo)

## Prerequisites

* x86 Linux (but the few binary dependencies could easily be changed for aarch64)
* Docker installed and running. It should work fine with
  [podman](https://podman.io/) but it has not been tested.

> NOTE: These instructions have only been tested on Linux x64.

## Setup

Clone this Git repo. Everything runs in Docker so no need to install anything
on your machine.

## Hello World

Let's start with a simple Hello World example.

Change the directory to `helloworld`.

![](images/keyboard.jpg) `cd helloworld`

Use the `build.sh` script to run a Docker build that:
1. compiles a simple single Java class Hello World application with `javac`
2. compiles the generated .class file with GraalVM Native Image into a fully
statically linked native Linux executable named `hello`
3. compresses the executable with [upx](https://upx.github.io/) to create the
executable `hello.upx`
4. packages the compressed static `hello.upx` executable into a `scratch` Docker
container image

In a terminal, run:

![](images/keyboard.jpg) `./build.sh`

### Native Executables

1. The `hello` executable generated by GraalVM Native Image in the Dockerfile
   using the `--static --libc=musl` options is a fully self-contained
   executable. This means that it does not rely on any libraries in the host
   operating system environment. This makes it easier to package in a variety of
   container images.

2. You can see in the output of the Dockerfile build that `ls -lh` reports the
   `hello` executable is ~4.9MB. There's no JVM, no JARs, no JIT compiler and
   none of the overhead it imposes. It starts extremely fast as there is minimal
   startup cost.

3. The `upx` compressed `hello.upx` executable is over 70% smaller, 1.3MB vs.
   4.9MB! A `upx` compressed application self-extracts quickly but does incur a
   cost of about 100ms for decompression. See this blog for a deep dive on
   [GraalVM Native Image and
   UPX](https://medium.com/graalvm/compressed-graalvm-native-images-4d233766a214).  

### Container Images

The size of the `scratch`-based container image is about the same size as the
`hello.upx` executable since it adds little overhead.

![](images/keyboard.jpg) `docker images hello`

```shell
REPOSITORY       TAG         IMAGE ID      CREATED             SIZE
hello            upx         b69a5d79e8dc  1 second ago        1.3MB
```

This is a tiny container image and yet it contains a fully functional and
deployable (although fairly useless 😉) application.  

Running the executable in the container image is straight forward:

![](images/keyboard.jpg) `docker run --rm hello:upx`

```shell
Hello World
```

Amazingly, it works!

## A Simple Web Server

Containerizing Hello World is not that interesting so let's move on to something
you could actually deploy as a service. We'll take the [Simple Web
Server](https://blogs.oracle.com/javamagazine/post/java-18-simple-web-server)
introduced in JDK 18 and build a containerized executable that serves up web
pages.

How small can a containerized Java web server be? Would you believe a measly
3.9 MB? Let's see.

Let's move from the `helloworld` folder over to the `jwebserver` folder. 

![](images/keyboard.jpg) `cd ../jwebserver`

There are a number of different GraalVM Native Image [linking
options](https://www.graalvm.org/22.0/reference-manual/native-image/StaticImages/)
that are suitable for different container images.

![](images/linkingoptions.png)

The `build-all.sh` script will generate a number of container images that
illustrate various linking and packaging options as well as a `jlink` generated
custome runtime image for comparison.

![](images/keyboard.jpg) `./build-all.sh`

The various Dockerfiles simply copy the compiled executable or `jlink` generated
custom runtime image folder into the deployment container image along with an
`index.html` file to serve, and set the `ENTRYPOINT`.

The Distroless Java, Eclipse Temurin, and Debian container images include a full
JDK, which includes jwebserver.

When complete you can see the sizes of the various variants:

![](images/keyboard.jpg) `$ docker images jwebserver`

```shell
REPOSITORY          TAG                            IMAGE ID            CREATED             SIZE
jwebserver          distroless-java                de7f7efb6df4        4 minutes ago       192MB
jwebserver          temurin                        643203bf8168        4 minutes ago       451MB
jwebserver          debian                         fa5bfa4b2e5e        4 minutes ago       932MB
jwebserver          distroless-java-base.jlink     c3113c2400ea        5 minutes ago       122MB
jwebserver          scratch.static-upx             75b3bb3249f3        5 minutes ago       3.9MB
jwebserver          alpine.static                  178081760470        6 minutes ago       21.6MB
jwebserver          distroless-static.static       84053f6323c1        6 minutes ago       15.8MB
jwebserver          scratch.static                 98061f48037c        6 minutes ago       13.8MB
jwebserver          distroless-base.mostly         b33fc99fbe2a        7 minutes ago       34.3MB
jwebserver          distroless-java-base.dynamic   1aceeabbb329        7 minutes ago       46.9MB
```

Sorting by size, it's clear that the fully statically linked GraalVM Native
Image generated executable that's compressed and packaged on `scratch`
(`scratch.static-upx`) is the smallest at just 3.9 MB, less than 4% of the size
of the `jlink` version (`distroless-java-base.jlink`) running on the JVM.

| Base Image           | App Version                        | Size (MB) |
| -------------------- | ---------------------------------- | --------- |
| Debian Slim + JDK    | jwebserver included in the JDK     |    932.00 |
| Eclipse Temurin      | jwebserver included in the JDK     |    451.00 |
| Distroless Java      | jwebserver included in the JDK     |    192.00 |
| Distroless Java Base | jlink                              |    122.00 |
| Distroless Java Base | native *dynamic* linked            |     46.90 |
| Distroless Base      | native *mostly* static linked      |     34.30 |
| Alpine               | native *fully* static linked       |     21.60 |
| Distroless Static    | native *fully* static linked       |     15.80 |
| Scratch              | native *fully* static linked       |     13.80 |
| Scratch              | *compressed* native *fully* static |      3.90 |

Running a container image is once again straight forward, just remember to map
the server port, e.g.:

![](images/keyboard.jpg) `docker run --init --rm -p8000:8000 jwebserver:scratch.static`

or

![](images/keyboard.jpg) `docker run --init --rm -p8000:8000 jwebserver:scratch.static-upx`

Using `curl` or your favourite tool you can hit `http://localhost:8000` to fetch
the _index.html_ file.

## Wrapping Up

A fully functional, albeit minimal, Java "microservice" was compiled into a
native Linux executable and packaged into Distroless, Alpine, and
`scratch`-based container images thanks to GraalVM Native Image's support for
various linking options including fully static linking with `musl libc`.

To learn more about linking options check out [Static and Mostly Static Images](https://www.graalvm.org/latest/reference-manual/native-image/guides/build-static-executables/) in the GraalVM docs.