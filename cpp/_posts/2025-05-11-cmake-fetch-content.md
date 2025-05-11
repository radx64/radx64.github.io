---
layout: post
title: "Using CMake FetchContent to handle dependencies"
categories: [cpp, general]
tags: [cmake, libs, dependecies]
---

### What is all about
So, picture this: I'm hopping between a bunch of Debian-based Linux distros-Ubuntu on WSL here, LMDE there, plain old Debian somewhere else, and oh look, another LTS distro lurking in the shadows. Normally, life is good. Your projects compile just fine using dependencies installed via `apt`. But then, BAM! You hit a wall. Different distros, different package versions, and suddenly you're stuck in dependency hell. Updating a package? Forget about it. Enter CMake's FetchContent, the hero you didn't know you needed.

### What is FetchContent?

The `FetchContent` module in CMake is a tool that simplifies dependency management. It allows you to fetch, configure, and integrate external libraries directly into your build system without relying on system-wide package managers. This is particularly useful when working across different environments where package versions may vary.

#### Key Functions

1. **FetchContent_Declare**: This function declares the external dependency, specifying details like the repository URL, tag, and additional options such as shallow cloning and progress display.

2. **FetchContent_MakeAvailable**: After declaring dependencies, this function ensures they are downloaded and made available for use in your project.

#### Benefits

- **Consistency**: Ensures the same version of dependencies across all environments.
- **Ease of Use**: Simplifies the process of integrating libraries into your project.
- **Flexibility**: Supports various repository types like Git and SVN.

For the nitty-gritty details, check out the [official CMake documentation](https://cmake.org/cmake/help/latest/module/FetchContent.html).

### How?
Basically FetchContent can drag your CMake fueled dependencies from GIT/SVN or other repository, described by tag/commit hash, and configures them during project configuration. I was really suprised that it worked that easily.

Here's how I use it:

```cmake
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        v1.17.0
  GIT_SHALLOW ON
  GIT_PROGRESS TRUE
  SYSTEM
)
...

FetchContent_MakeAvailable(googletest sfml fmt)
```

Just give it the name of your package (like "googletest" above), the repository URL, and the tag you want. Add a few optional parameters if you're feeling fancy, and voilà! It downloads, configures, and integrates everything into your build system. No more relying on system packages.

At the end, you just need to tell CMake which packages should be available. Easy peasy.

### Ok but it can't be that easy right?

Of course not. Life's never that simple. In my case, I'm building the SFML library from scratch. And guess what? Now I have to provide all the dependencies of SFML in my system. Right now, I'm installing them with `apt`, but hey, maybe FetchContent can save the day here too?

#### Downsides of FetchContent

While FetchContent is undeniably convenient, it does come with a few caveats:

1. **Increased Build Times**: Fetching and building dependencies during the configuration phase can significantly increase build times, especially for large libraries.

2. **Network Dependency**: Since FetchContent relies on fetching dependencies from remote repositories, it requires a stable internet connection. This can be problematic in offline environments.

3. **Version Management**: While FetchContent allows you to specify versions, managing updates and ensuring compatibility across multiple projects can still be challenging.

4. **Disk Usage**: FetchContent downloads and builds dependencies locally, which can lead to increased disk usage compared to using pre-installed system packages.

Despite these downsides, FetchContent might be a powerful tool for managing dependencies in a consistent and flexible manner.

### My final thoughts
I'm taking all of this with a grain of salt. Sure, it has its upsides and downsides, but that's true for most tools. I've already implemented it in my [pet project](https://github.com/radx64/battle_tanks), and I'm planning to daily drive it for a while to see how it holds up in real-world scenarios. Will it improve my workflow or just add another layer of complexity? IDK.

One thing's for sure: FetchContent is different. It challenges the traditional way of managing dependencies, and that alone makes it worth exploring. Whether it becomes a staple in my toolkit or a passing experiment, I'm glad I gave it a shot. After all, innovation often comes from stepping out of your comfort zone.
