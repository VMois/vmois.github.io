---
layout: post
title: Use locally built standard library in Mojo
tags: [mojo, short]
comments: false
readtime: true
slug: mojo-local-stdlib-build
---

[Mojo standard library (stdlib) was open-sourced yesterday](https://www.modular.com/blog/the-next-big-step-in-mojo-open-source). It is exciting that the community can now contribute directly to the codebase. After spending some time with the stdlib repository, I want to share how you can change the standard library and make Mojo use your version when running programs.

{: .box-note}
The steps below were only tested on macOS and will likely work on Linux.

You will need the [Mojo repository](https://github.com/modularml/mojo) containing stdlib code. After making code changes in your local copy, you should build a library using the `build-stdlib.sh` script located in `stdlib/scripts/build-stdlib.sh`.

The custom stdlib library is saved in the `build/stdlib.mojopkg` folder at the top level of the Mojo repository. Now, we need to replace the standard stdlib with the custom one. But where is the standard library located?

Modular CLI stores all data in the `~/.modular` folder. If we walk it for a bit, we will find the following folder `.modular/pkg/packages.modular.com_mojo/lib/mojo`, and it's content:

```
-rw-r--r--  1 vmois  staff  24449729 Mar 28 00:18 algorithm.mojopkg
-rw-r--r--  1 vmois  staff   1878446 Mar 28 00:18 autotune.mojopkg
-rw-r--r--  1 vmois  staff  12263261 Mar 28 00:18 benchmark.mojopkg
-rw-r--r--  1 vmois  staff   7377122 Mar 28 00:18 buffer.mojopkg
-rw-r--r--  1 vmois  staff   2117290 Mar 28 00:18 compile.mojopkg
-rw-r--r--  1 vmois  staff   4649490 Mar 28 00:18 complex.mojopkg
-rw-r--r--  1 vmois  staff   7598891 Mar 28 00:18 gpu.mojopkg
-rw-r--r--  1 vmois  staff   2773554 Mar 28 00:18 math.mojopkg
-rw-r--r--  1 vmois  staff   2317151 Mar 28 00:18 runtime.mojopkg
-rw-r--r--  1 vmois  staff   3435814 Mar 29 01:11 stdlib.mojopkg <---
-rw-r--r--  1 vmois  staff  48541986 Mar 28 00:18 tensor.mojopkg
```

The `stdlib.mojopkg` file is the standard library package. If we replace it with the one we just built and try to run Mojo programs as usual, our updated code will be used.

{: .box-warning}
Make a copy of the original stdlib package before replacing it.

```bash
$ cp <path-to-mojo-repo>/build/stdlib.mojopkg <path-to-modular-folder>/.modular/pkg/packages.modular.com_mojo/lib/mojo/stdlib.mojopkg
```

{: .box-note}
You can adjust the process so there is no need to copy the library constantly. For example, try a symbolic link.

Enjoy.

