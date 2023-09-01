---
layout: post
title: Learning Chisel HDL - Resources and Tips
tags: [hardware,chisel,docker]
comments: false
readtime: true
slug: learning-chisel
last_modified_at: September 1, 2023
---

In this short article, I want to share some materials and tips about learning [Chisel](https://www.chisel-lang.org/), a hardware description language (HDL) embedded in Scala. It will be handy for software engineers interested in hardware design. Expect this article to be updated as I progress with my learning journey.

{: .box-note}
It is worth mentioning that I am not a hardware designer expert. I have an experience with Verilog/VHDL and digital design from my university studies.

## Why Chisel?

I learned about the existence of Chisel HDL a while ago when I was learning about compilers for programming languages, [LLVM](https://github.com/llvm/llvm-project) and [MLIR](https://mlir.llvm.org/) projects. MLIR is a compiler framework used to build compilers for different domains, including hardware. To validate how successful MLIR can be used in hardware design, the [CIRCT](https://github.com/llvm/circt) project was created. Chisel HDL uses CIRCT ([FIRTTL](https://circt.llvm.org/docs/Dialects/FIRRTL/) dialect, to be precise) underneath to convert Scala/Chisel code to Verilog. Considering my electrical engineering studies and software development experience, I got excited and tried Chisel.

{: .box-note}
There are debates about Chisel vs. Verilog/VHDL and whatever Chisel is useful for hardware designers. I am not going to discuss those things here. The goal is to learn something new and have fun. But, if you plan to become a hardware designer, I suggest focusing on Verilog or VHDL first. It is the de-facto standard in most companies so that you can get a job more easily.

## Resources

The list of resources I have found useful and recommend to check.

### Chisel Book

I wanted to brush up my knowledge of digital design. I have decided to use [Chisel Book](https://github.com/schoeberl/chisel-book) by Martin Schoeberl. I recommend it a lot for both learning about Chisel3 and digital design. In addition to the book, check [Chisel examples](https://github.com/schoeberl/chisel-examples) repository. It contains many examples from the book in a more complete form.

I have started learning Chisel with a specific project in mind, so after checking a few introduction chapters, I jumped into chapters that were useful to me. If you are generally learning, going through the book from the beginning would probably make more sense.

You can also find some slides from [Digital Electronics 2](https://www2.imm.dtu.dk/courses/02139/) course talking about Chisel.

### Chisel template repository

I like to learn by doing, so I started by running some Chisel examples on my laptop and outputting some Verilog code. I highly suggest checking [chisel-template](https://github.com/freechipsproject/chisel-template) project. It provides all the necessary setup code, and you must install Scala tools.

### Interesting Chisel projects

In this section, you will find a list of interesting projects that you might take an inspiration from.

- [FPGA code samples for different interfaces written in Chisel2](https://github.com/maltanar/fpga-tidbits)
- [FPGA shells with wrappers for different interfaces written in Chisel](https://github.com/sifive/fpga-shells)
- [Collection of different useful small Chisel3 projects](https://github.com/j-marjanovic/chisel-stuff)
- [Project that shows how Chisel and Rust can have a custom peripherals](https://github.com/ekiwi/pynq)

## Tips

Below you can find some tips that I found useful.

### Using Docker to develop in Chisel

A small tip I want to share is that I like to use Docker containers to have an isolated environment instead of installing tools directly on my laptop. For Chisel, I have created a simple Dockerfile that you can use with the `chisel-template` mentioned above:

```dockerfile
FROM openjdk:11

ENV SBT_VERSION 1.9.0
RUN curl -L -o sbt-$SBT_VERSION.zip https://github.com/sbt/sbt/releases/download/v$SBT_VERSION/sbt-$SBT_VERSION.zip && unzip sbt-$SBT_VERSION.zip -d ops
ENV PATH="/ops/sbt/bin/:${PATH}"
WORKDIR /design
```

You can build the container:

```bash
docker build -t scala .
```

And then map it to a folder where chisel-template project is and run it:

```bash
docker run -v <absolute-path>/chisel-template:/design -it scala bash
```

Now you can run the `sbt` command inside the container and modify the code in your favorite editor directly on your laptop. Using Docker is my personal preference, but maybe, you will find it helpful as well.

## Conclusion

Thank you for reading! This is an unusual format for me (publishing short notes), so I hope you will find the material about Chisel HDL useful. I will update this article as I progress with my learning journey.
