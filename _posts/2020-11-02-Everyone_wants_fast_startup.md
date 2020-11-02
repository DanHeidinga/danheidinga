---
layout: post
title: 'Everyone wants fast startup: introducing JVM snapshot+restore'
---

The Eclipse OpenJ9 project has a strong focus on finding the right balance between startup performance, peak throughput and memory usage.  And we've done a pretty good job of it - see any of our [claims](https://www.eclipse.org/openj9/performance/) about fast startup with comparable performance in roughly half the memory.  

In the next major evolution of our work in this space, the project recently created a new `snapshot` branch containing an early prototype of snapshot+restore functionality at the JVM level which allows snapshotting (saving) a running JVM process and restoring it later from the saved point.  

To understand how we arrived at snapshot+restore of the JVM as the best approach for the widest set of workloads, I'd like to share the journey we went on as we investigated this space.

The snapshot work is originally inspired by Linux's [CRIU](https://www.criu.org/Main_Page) (checkpoint restore in user-space) which does the same thing as snapshot+restore but for a full Linux process.  Past [experiments](https://openliberty.io/blog/2020/02/12/faster-startup-Java-applications-criu.html) with CRIU have demonstrated that JVM startup time can be greatly improved by checkpointing the process and then subsequently creating new processes from that checkpointed state.  That's the basic premise of CRIU: live migrating systems is more performant than stopping and restarting.

CRIU, while being an amazing piece of technology, has some drawbacks:

* It defeats address space layout randomization (ASLR).  Restarting a checkpoint results in exactly the same address space layout as previous runs making it easier for attackers to "learn" addresses for later attacks.
* Restoring a CRIU process requires running as `root`. The capabilities needed to restore give the application more privileges than it would otherwise need resulting in a larger attack surface.  Work is being done in this space so this may be less of an issue in the future.
* Secrets, encryption keys, random number generators and more are all reused from the checkpoint.  Similar to the ASLR issue, this makes operations more predicable and easier to attack as well as exposing them to anyone with a copy of the checkpointed process.
* The large footprint overhead of a full process image.  All memory in the process needs to be saved, including the unused space in the Java heap, which can result in larger than necessary images (and restore times).
* Only available on Linux while the JVM runs on a plethora of operating systems and architectures.  

After our CRIU investigation, we took a step back and surveyed the landscape before landing on "Snapshot+Restore" at the JVM level.  For the impatient, jump ahead to the "Snapshot+Restore" section.  Otherwise, follow along through the spectrum of options that helped guide our choice to focus on snapshot+restore at the JVM level.

# Spectrum of options

While our investigation started with CRIU and focused on how to make OpenJ9 start even faster than it already does ([check out SharedClasses and dynamic AOT if you haven't already!](https://developer.ibm.com/tutorials/j-class-sharing-openj9/)), it would be foolish to ignore Graal's SubstrateVM technology when looking at this space.

SubstrateVM starts applications fast through native image creation.  Quarkus shows off the startup benefits of SubstrateVM on its [main page](https://quarkus.io), demonstrating that incredibly fast startup time (aka time to first request) when the application is compiled to native.  The results are impressive but there are trade-offs required of the developer, the framework, and the platform to get there - see SubstrateVM's [limitations](https://github.com/oracle/graal/blob/master/substratevm/Limitations.md) for a description of some of these tradeoffs.

From the investigations we've done into OpenJ9 startup improvements, CRIU, and SubstrateVM, a spectrum of options emerges, each targeted towards different use cases.  This spectrum is anchored at its extremes by the full Java experience at one end, and the closed world native compiled image at the other.

### Full Java experience

We all know what the full Java experience provides as that's what we develop for every day.  Using dynamic class loading to load classes as they're needed, using reflection to access methods and fields that aren't known at compile time, serializing data, calling natives or running JVMTI-based monitoring agents are all part of what makes the full Java experience so rich and dynamic.  

But nothing comes for free so our applications pay in slower startup (alleviated by AOT and SharedClasses technology) and higher memory use.  Normally, these are costs we gladly pay.

### Closed world native compiled image

The closed world native compiled image also has a lot of benefits.  It allows very fast startup time and extremely small memory footprint, both at runtime and on disk.  The single executable packaging format tends to fit well with container tools.  There's a lot to like here.  But all this doesn't come for free either.  

Opting into the benefits of native image means coding to the subset of Java that native image can compile and leaves a lot of existing applications unable to benefit as they can't reasonably change to fit the restrictions.  

And even for those that can benefit, it could mean a significant drop in peak performance compared to a JVM or to get nearer to JVM comparable peak throughput, increasing to a comparable memory footprint. ([Performance details](https://quarkus.io/blog/runtime-performance/)) (Note, always do your own measurements as workload behaviour may vary).  

Doing development on a JVM while deploying on a different technology exposes you to differences in runtime behaviour between the two implementations and may lead to issues at deploy time.

# Snapshot+Restore in the JVM

We needed a solution that allows applications to pick different points on this spectrum rather than being limited to one of the extremes: full JVM or closed world native image.  

Taking CRIU's checkpoints and re-implementing them as a JVM snapshot+restore mechanism provides a bridge across the extremes of the spectrum.  Similar to CRIU, a JVM snapshot captures the state of the running application including the current object heap, metadata for loaded classes, jit compiled code, and execution state of threads.  Basically all the state require to allow restoring the application.

Since a snapshot can be taken at any point during runtime - whether that's after running many class initializers to get an initial heap and avoid class loading, or just before loading the application code into a framework to avoid the framework initialization, or even after a number of requests to save a fully initialized system - it allows different tradeoffs than available at the extremes of the spectrum.

One of the advantages of this approach is it doesn't require a closed-world.  Snapshot+Restore can be combined with a closed-world but it also allows the world to be left open at the restore, allowing further dynamic class loads and new jit compilations to continue to occur.

A JVM implementation brings the capability to a wider set of platforms (not just Linux!) and avoids tying application behaviour to a particular operating system.  And it has the potential to produce smaller images than CRIU while still gaining the fast startup benefits.

Because the Snapshot+Restore is integrated into the JVM, it can avoid many of the security concerns raised with the CRIU approach by allowing the application to help control which data is not captured - ie: encryption keys - and which state needs to be re-initialized on each run - ie: random number generators.

The new OpenJ9 `snapshot` project explores snapshot+restore at the JVM level and is pursuing the following major goals:

1. Produce a restartable snapshot from the JVM state
1. Ensure as much of the snapshot memory as possible is share-able across restored instances
1. Optionally, allow closing the world at the snapshot through static code analysis to minimize loaded classes / methods / fields and allow complete native compilation of all methods

To meet these goals, the project focuses on providing a basic set of primitive operations that can be viewed as building blocks.  Combining those primitives in different ways hits different points on the spectrum and can address a wide variety of use cases.  These primitives let the JVM provide the tools to help with more than just the small footprint, fast start use cases.  Snapshots also help with process migration, debugging use cases (snapshot just before the problem occurs and then debug it over and over again avoiding the setup costs), problem determination (snapshot the production workload when an issues occurs and investigate offline), and keep existing tools like JVMTI monitoring agents working.

# Current state of the prototype

The current OpenJ9 `snapshot` branch is an early prototype providing the right primitives.  It allows snapshotting the heap, and restoring a limited set of application state using a Lifecycle API that provides pre and post snapshot hooks.  It doesn't yet work with ASLR enabled but work is being done to address the limitations.  See OpenJ9 issue [#10680](https://github.com/eclipse/openj9/issues/10680) for details on how to snapshot+restore the current state

The elephant in the room is package size.  Which like an elephant, is still too large for the containers we want to put it in.  To address this "elephant", we've introduced a static analysis tool called ["jarmin"](https://github.com/eclipse/openj9-utils/tree/master/jarmin) that can be run on the jar files and JDK libraries to remove unneeded classes, methods and fields.  This helps to shrink the package size, especially for applications with small heaps where class metadata is a larger portion of the footprint.


# Common operations of all approaches

If you step back and look at the solutions in this space - whether SubstrateVM's native image, OpenJ9's snapshot+restore, or even CRIU - they have a lot in common and do many of the same operations.

All solutions:

* capture the state of the Java heap and are able to restore it later.  In some cases, this is primarily for class initializers (`<clinit>`) while others preserve additional objects.
* capture all or enough of the Class metadata to be able to describe the objects on the heap and to support some of the capabilities of the Java programming language.
* capture the compiled code, whether AOT or JIT compiled, and reuse it later to avoid recompiling for each execution.

And they typically need one more thing: a way to allow the application to help determine which state gets reset and recreated rather than persisted.  In native image case, this can be as simple as deciding which `<clinit>`'s get run at build time vs those run at runtime.  For a snapshot+restore, this may be hooks, a "lifecycle api", to allow an application to close sockets or files at snapshot and reopen them at restore.

# Broader ecosystem

There's a lot going on in this space.  Already mentioned are CRIU and Graal's SubstrateVM.  Recent additions are OpenJDK's announcement of [Project Leyden](https://mail.openjdk.java.net/pipermail/discuss/2020-April/005429.html) focused on static images and Azul's [CRaC](https://mail.openjdk.java.net/pipermail/discuss/2020-September/005594.html) which builds coordination on top of CRIU's checkpointing.

Our goals are to continue to develop the snapshot+restore technology and to share our experience and learn from others experience working in this space.  In the end, we all want the best APIs and capabilities for Java and the ecosystem.

Sound interesting?  Then join in with the snapshot+restore efforts by checking out the [`snapshot`](https://github.com/eclipse/openj9/tree/snapshot) branch, joining the #snapshot channel on the [OpenJ9 slack](https://join.slack.com/t/openj9/shared_invite/enQtNDU4MDI4Mjk0MTk2LWVhNTMzMGY1N2JkODQ1OWE0NTNmZjM4ZDcxOTBiMjk3NGFjM2U0ZDNhMmY0MDZlNzU0ZjAyNzQ1ODlmYjg3MjA), or engaging with us on GitHub.

----

(Many thanks to reviewers for comments and suggestions: Andrew Craik, Ashutosh Mehra, Daryl Maier, Mark Stoodley, Theresa Mammarella, Vijay Sundaresan)
