# What is a container?

Containers are a way of packaging a program, its dependencies, and its runtime environment in a language-agnostic and portable way. It’s also a sandbox that prevents the program from scribbling all over the filesystem of the machine it’s running on, and a way of limiting the compute resources that the program can use. You can also argue that its registry system makes it something of a package manager for arbitrary software artifacts.

The idea was popularized by some folks at a platform-as-a-service company called dotCloud, who needed a lightweight way of isolating its customers’ workloads from one another. They created a container management and construction system called Docker, and went on to form a company of the same name to commercialize the technology. Whether the company did well is questionable, but the technology did incredibly well. We’ll be using Docker to build our containers.

The technological underpinnings of containers (namespaces, cgroups and union mount filesystems) have been part of Linux for a long time. You could even look at containers as the next logical step from things like chroot jails that have been around in Unix-derived operating systems since the early 1980s.

# Why does Hydroplane use containers?

Hydroplane uses containers because virtually every runtime that Hydroplane either implements or would potentially implement supports them. They are, for better or worse, the lowest-common-denominator solution for packaging and distributing software in the cloud.

You might argue that packaging a zip file and pushing it to an object store like S3 or GCS would accomplish the same thing for the Hydro project’s purposes, and you might be right. Once you start thinking through what it would take to actually implement that process end-to-end across a wide variety of runtimes, however, you’ll find that you’re backing into a lot of the same stuff that containers will already give you.

Building containers is kind of a pain in the neck, but I hope that this document and its accompanying artifacts will make that process a little easier.

# How does Docker work?

In order for what I’m about to describe to be less mysterious, it’s important to understand some basics about how container construction works. You can skip this section if you’re already familiar with Docker.

Docker builds container images. Container images are just filesystems, packaged in a particular way that allows for reuse and extension. Images consist of layers, each of which is a read-only filesystem image that’s transparently overlaid on top of all the layers below it. Since every image is constructed of layers, the author of a container image can re-use the layers that someone else wrote in the past through the use of a base image. When Docker constructs a layer, it also constructs a content-based hash of that layer, which makes it relatively straightforward to cache layers that have already been built.

This layering principle extends to runtime as well. When a container runs, it adds one additional read-write layer to the top of its image’s read-only stack of layers, and runs a program (called the container’s entrypoint) in the resulting read-write filesystem.

A container image is defined in a Dockerfile. This file consists of a collection of steps, each of which will create a new layer. It also describes the container’s entrypoint.

In order for someone to run a Docker image, they need a copy of its constituent layers. When you build a container image on your local machine, you’ve now got that image’s layers, and the image is immediately runnable by your Docker installation. If you want to share that image with someone else, or run it somewhere else, you need to send a copy of its layers there. This is where registries come in. You can think of a registry like a git repository, except for filesystem layers instead of software revisions.

A registry is a collection of images, each of which has a name and (optionally) a tag that differentiates it from other versions with the same name. You send images to a registry by pushing them, and retrieve images from a registry by pulling them.

# How Do I Containerize My Rust Program?

To containerize your Rust program, we’ll want to build a Dockerfile for it. This repo contains a couple examples of how to build a Docker image containing a "hello world" Rust program. We'll look at each of these examples in turn.


## The Simplest Solution

