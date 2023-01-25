# HOWTO: Package Rust Programs into Containers

## What is a container?

Containers are a way of packaging a program, its dependencies, and its runtime environment in a language-agnostic and portable way. It’s also a sandbox that prevents the program from scribbling all over the filesystem of the system it’s running on, and a way of limiting the compute resources that the program can use. You can also argue that its registry system makes it something of a package manager for arbitrary software artifacts.

The idea was popularized by some folks at a platform-as-a-service company called dotCloud, who needed a lightweight way of isolating its customers’ workloads from one another. They created a container management and construction system called Docker, and went on to form a company of the same name to commercialize the technology. Whether the company did well is questionable, but the technology did incredibly well. We’ll be using Docker to build our containers.

The technological underpinnings of containers (namespaces, cgroups and union mount filesystems) have been part of Linux for a long time. You could even look at containers as the next logical step from things like chroot jails that have been around in Unix-derived operating systems since the early 1980s.

## Why does Hydroplane use containers?

Hydroplane uses containers because virtually every runtime that Hydroplane either implements or would potentially implement supports them. They are, for better or worse, the lowest-common-denominator solution for packaging and distributing software in the cloud.

You might argue that packaging a zip file and pushing it to an object store like S3 or GCS would accomplish the same thing for the Hydro project’s purposes, and you might be right. Once you start thinking through what it would take to actually implement that process end-to-end across a wide variety of runtimes, however, you’ll find that you’re backing into a lot of the same stuff that containers will already give you.

Building containers is kind of a pain in the neck, but I hope that this document and its accompanying artifacts will make that process a little easier.

## How does Docker work?

In order for what I’m about to describe to be less mysterious, it’s important to understand some basics about how container construction works. You can skip this section if you’re already familiar with Docker.

Docker builds container **images**. Container images are just filesystems, packaged in a particular way that allows for reuse and extension. Images consist of **layers**, each of which is a read-only filesystem image that’s transparently overlaid on top of all the layers below it. Since every image is constructed of layers, the author of a container image can re-use the layers that someone else wrote in the past through the use of a base image. When Docker constructs a layer, it also constructs a content-based hash of that layer, which makes it possible to cache layers that have already been built.

This layering principle extends to runtime as well. When a container runs, it adds one additional read-write layer to the top of its image’s read-only stack of layers, and runs a program (called the container’s **entrypoint**) in the resulting read-write filesystem.

A container image is defined in a `Dockerfile`. This file consists of a collection of steps, each of which will create a new layer. It can also define the container’s entrypoint, although you can also specify a custom entrypoint at runtime.

In order for someone to run a Docker image, they need a copy of its constituent layers. When you build a container image on your local system, you’ve now got that image’s layers, and the image is immediately runnable by your Docker installation. If you want to share that image with someone else, or run it somewhere else, you need to send a copy of its layers there. This is where **registries** come in. You can think of a registry like a git repository, except for filesystem layers instead of software revisions.

A registry is just a collection of images, each of which has a name and (optionally) a tag that differentiates it from other versions with the same name. You send images to a registry by pushing them, and retrieve images from a registry by pulling them. In this sense, it's very similar to both git repositories and package registries like `crates.io`.

The most popular registry by far is [DockerHub](https://hub.docker.com/). Every major cloud provider also has their own private registry solutions: AWS has [Elastic Container Registry](https://aws.amazon.com/ecr/), GCP has [Google Cloud Container Registry](https://cloud.google.com/container-registry), and Azure has [Azure Container Registry](https://azure.microsoft.com/en-us/products/container-registry). GitHub also provides both public and private registries via [GitHub Packages](https://github.com/features/packages).

## How Do I Containerize My Rust Program?

To containerize your Rust program, we’ll want to build a Dockerfile for it. This repo contains a couple examples of how to build a Docker image containing a "hello world" Rust program. We'll look at each of these examples in turn.

The example project can be found in [`my-project/`](/tree/main/my-project). That project contains a few `Dockerfile`s, in [`my-project/dockerfiles`](/tree/main/my-project/dockerfiles). These `Dockerfile`s all accomplish the same thing -- they build a container that will run the `my-project` binary -- but they do so in different ways.

### The Simplest Solution

[`Dockerfile.simple`](/blob/main/my-project/dockerfiles/Dockerfile.simple) is a `Dockerfile` that illustrates the most straightforward way to build a Rust program into a container. For convenience, I've pasted it below:

```dockerfile
FROM rust:slim-bullseye

WORKDIR /usr/src/myapp
COPY . .

RUN cargo build --release

ENTRYPOINT ["/usr/src/myapp/target/release/my-project"]
```

All the words in all-caps are **commands**. Some commands write a new layer on top of the previous command's layer, while others change the image's metadata, and still others modify the build's state in convenient ways. A comprehensive list of available Docker commands can be found in [the `Dockerfile` command reference](https://docs.docker.com/engine/reference/builder/), but we'll only look at a handful of them here.

Let's study each command in turn to understand what it's doing.

```dockerfile
FROM rust:slim-bullseye
```

The `FROM` command tells Docker which base image we're building on top of. In this case, we're using the `rust` image, specifically the image version tagged with the `slim-bullseye` tag (which is based off a slimmed-down version of [Debian Bullseye](https://www.debian.org/releases/bullseye/)). The official `rust` image has a _lot_ of tags. You can see those tags described on [its DockerHub page](https://hub.docker.com/_/rust). As you can see, you can get just about any version of Rust, on top of several Linux distributions. If you look at [the complete list of tags for the `rust` image](https://hub.docker.com/_/rust/tags), you'll notice that each tag has builds for 32-bit and 64-bit X86 and ARM. Docker will automatically pick the image that matches the architecture of the system you're building on. You can build Docker images for platforms other than your own, but that's an advanced topic that we'll cover later in this document.

```dockerfile
WORKDIR /usr/src/myapp
```

This command switches your working directory within the container image's filesystem to `/usr/src/myapp`. Henceforth, any files you copy into the image will be copied relative to this directory.

```dockerfile
COPY . .
```

When you run `docker build` inside a directory on your system, the filesystem rooted at that directory becomes the build's **context**, and the `Dockerfile` can copy any file from that filesystem into the image that it's building. This command copies _everything_ from the build's context into the image.

We'll be building your image from within `my-project/`. Since you previously set your `WORKDIR` to `/usr/src/myapp`, files will be copied from `my-project/` on your system to `/usr/src/myapp/` in the container image.

```dockerfile
RUN cargo build --release
```

This command runs `cargo build --release`. All the files that `cargo build` writes will be a part of this command's associated layer.

```dockerfile
ENTRYPOINT ["/usr/src/myapp/target/release/my-project"]
```

This command sets the image's entrypoint to the binary that was built in the previous step.

### Let's Try It!

To test this `Dockerfile` out, enter the `my-project/` directory and run

```bash
docker build -f dockerfiles/Dockerfile.simple -t my-project:simple .
```

This command uses `dockerfiles/Dockerfile.simple` as the Dockerfile that will guide the build process, and will call the resulting image `my-project` and tag it with the `simple` tag. If you didn't specify a tag here (by using `-t my-project` instead of `-t my-project:simple`), Docker would tag the resulting image with the `latest` tag.

Once the build completes, run:

```bash
docker run --rm my-project:simple
```

This will run a container based on the `my-project` image. You should see it print "Hello, world!" and exit.

The `--rm` flag tells Docker to delete the container's filesystem after the container completes. It's a good idea to get in the habit of doing this, since most runtimes will not save your container's state between runs, and you don't want random container filesystems cluttering up your system's disk.

### Drawbacks of This Approach

This image is very simple to construct, but it won't be very smart about caching layers, and it's also carrying around a lot of files it doesn't need.

To understand the cache inefficiency of this approach, we need to look at the `COPY . .` command. Every time any file in the context changes (e.g. you edit any file in `src/`, or `Cargo.toml`, or even the `Dockerfile` itself), the layer that results from the `COPY` command will change. This means that all subsequent layers will have to be rebuilt on top of that layer, and `cargo build --release` will run every time you re-build.

For small projects, this isn't the end of the world, and it may be easiest to go with this solution. If you find that your container builds are taking a really long time, you may want to consider something a little less straightforward that's more effective at caching.

Another problem with this container image is that it's carrying around a lot more files than it needs. The layer that results from `RUN cargo build --release` is capturing not just the binaries in `target/`, but all intermediate build files and dependencies that went into producing it. For more complicated projects, this can be hundreds of MiB of unnecessary files that need to be pushed and pulled. Again, this isn't always the end of the world, but for larger projects it can start to be a problem.

## Multi-Stage Builds for Smaller Images

Here's a slightly more complicated `Dockerfile` that will make the resulting image significantly smaller:

```dockerfile
FROM rust:slim-bullseye as builder

WORKDIR /usr/src/myapp
COPY . .

RUN cargo build --release

FROM debian:bullseye-slim

COPY --from=builder /usr/src/myapp/target/release/my-project /usr/local/bin/my-project

ENTRYPOINT ["/usr/local/bin/my-project"]
```

There are two notable differences from `Dockerfile.simple` here.

We now have two `FROM` commands:

```dockerfile
FROM rust:slim-bullseye as builder
```

and

```dockerfile
FROM debian:bullseye-slim
```

One thing I didn't mention before about `FROM` is that it defines a build **stage**. Each stage is its own distinct image, and one stage can copy files from another stage's topmost layer, but only the last stage in a `Dockerfile` actually gets saved. This allows us to do building and preparation of binaries in one stage, and copy only those files that we need into another, smaller stage.

In this case, we're creating a temporary stage called `builder` that is based on `rust:slim-bullseye` and actually does our build. We're then creating our final stage from `debian:bullseye-slim`, and using a `COPY --from=builder` command to copy the `my-project` binary from the `builder` stage to the final stage.

The difference in size between the two images is huge. To see for yourself, enter the `my-project/` directory and run

```bash
docker build -f dockerfiles/Dockerfile.multistage -t my-project:multistage .
```

Now, let's look at the different images we have for `my-project` by running

```bash
docker images my-project
```

You'll see something that looks roughly like this:

```
REPOSITORY   TAG          IMAGE ID       CREATED              SIZE
my-project   simple       101fc304040a   About a minute ago   983MB
my-project   multistage   0623c2758fa8   About a minute ago   78.6MB
```

Note that the `simple` tag's image is 983 MB, but the `multistage` version is only 78.6 MB!