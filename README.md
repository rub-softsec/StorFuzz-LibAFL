# StorFuzz

This repository contains the fuzzer source code for the ICSE 2026 paper ["StorFuzz: Using Data Diversity to Overcome Fuzzing Plateaus"](https://doi.org/10.1145/3744916.3773179). It is based on LibAFL version 0.13.1

The complete artifacts can be found at [rub-softsec/StorFuzz](https://github.com/rub-softsec/StorFuzz).

The original README can be found at the bottom.

The fuzzer implementation used for our experiments can be found in [./fuzzers/storfuzz_fuzzbench_in_process](./fuzzers/storfuzz_fuzzbench_in_process). It shows by example how to use the LLVM coverage instrumentation pass and the newly added components.

The instrumentation pass is located in [./libafl_cc/src/storfuzz-coverage-pass.cc](./libafl_cc/src/storfuzz-coverage-pass.cc).

If you use StorFuzz in your work, please cite our paper:

```biblatex
@inproceedings{icse2026-storfuzz,
  title     = {{StorFuzz: Using Data Diversity to Overcome Fuzzing Plateaus}},
  author    = {Weiß, Leon and Holl, Tobias and Borgolte, Kevin},
  booktitle = {Proceedings of the 48th IEEE/ACM International Conference on Software Engineering (ICSE)},
  date      = {2026-04},
  editor    = {Mezini, Mira and Zimmermann, Thomas},
  location  = {Rio de Janeiro, Brazil},
  publisher = {Association for Computing Machinery (ACM)/Institute of Electrical and Electronics Engineers (IEEE)},
  doi       = {10.1145/3744916.3773179}
}
```
<details>
  <summary>How to <code>ß</code></summary>

  LaTeX natively supports unicode and with that the `ß` in Weiß. You may need to use `\usepackage[utf8]{inputenc}`. You can also use `{\ss}` instead of `ß`.

  The correct internationalization of `ß` is `ss`.
</details>

## Software Requirements

LibAFL requires a fairly recent Rust version (1.90.0 is known to work). The StorFuzz pass is built to work with LLVM 17, but should also work with more recent versions up until LLVM 19.

## How-To Fuzz with StorFuzz

To instrument and fuzz arbitrary C/C++ projects, simply build the fuzzer and use the resulting compiler wrapper with the compile flag `--libafl`.
The fuzzer assumes that the function `LLVMFuzzerTestOneInput` is defined as a harness and that there exists no `main` function.

This could look like this:
```bash
cd ./fuzzers/storfuzz_fuzzbench_in_process
unset LIBAFL_EDGES_MAP_SIZE_MAX
unset STORFUZZ_MAP_SIZE
CFLAGS="" CXXFLAGS="" cargo build --release

./target/release/libafl_cc --libafl <target_without_main.c> -o target_without_main

./target_without_main --help
```

If there is a `main` function, you might want to compile the fuzzer with `--features no_link_main` and link in the stub runtime provided as `stub_rt.c`.
This may however require changes to the build scripts.

```bash
cd ./fuzzers/storfuzz_fuzzbench_in_process
unset LIBAFL_EDGES_MAP_SIZE_MAX
unset STORFUZZ_MAP_SIZE
CFLAGS="" CXXFLAGS="" cargo build --release --features no_link_main

clang -c stub_rt.c && ar r stub_rt.a stub_rt.o

./target/release/libafl_cc --libafl stub_rt.a <target_with_main.c> -o target_with_main

./target_with_main --help

# For the original target behavior (e.g., if needed during build process)
CONFIGURE=1 ./target_with_main
```

### Useful Environment Variables
There are several environment variables that can be used to configure various aspects of StorFuzz and LibAFL.

#### Fuzzer Compile Time
- `LIBAFL_EDGES_MAP_SIZE_MAX`: The maximum size of the edges map used by the default LibAFL edge-coverage instrumentation in byte (1 edge = 1 byte)
- `STORFUZZ_MAP_SIZE`: Size in bytes of the StorFuzz map in memory (the space occupied by one instrumented store depends on the value reduction that is chosen). Must be a power of 2

#### Target Compile Time
- `AFL_LLVM_DICT2FILE`: Use the LibAFL `AutoTokensPass` to write a fuzzing dictionary to file while compiling the target. If set, it must be set to an absolute path
- `VALUE_REDUCTION_WIDTH`: Bitwidth of the reduced values, the used algorithms are explained at [rub-softsec/StorFuzz](https://github.com/rub-softsec/StorFuzz/blob/main/README.md#reduction-functions). Possible values are 4, 8 (default), 12, 16. `STORFUZZ_MAP_SIZE` must be large enough to accomodate at least 4096 entries
- `MAX_STORES_PER_BB`: Maximum number of stores that are instrumented in a basic block. If there are more stores in the basic block, it is skipped entirely (default: 9)
- `STORFUZZ_INSTRUMENT_MEM2MEM_COPY`: If set, instrument copies from memory to memory (by default these are not instrumented)
- `STORFUZZ_VERBOSE`: If set, the instrumentation pass prints additional information
- `CONFIGURE_MODE`: Flag to turn off StorFuzz instrumentation. Useful for configuring certain build systems that cannot cope with instrumented binaries during configuration
- `LIBAFL_INSTRUMENT`: If set, the call to `libafl_cc`/`libafl_cxx` does not need to include `--libafl` to instrument the target

#### Target Run Time
- `LIBAFL_FUZZBENCH_DEBUG`: If NOT set (default), the fuzzer discards all output from the target to ensure clean fuzzing logs and improve fuzzing speed
- `RUST_LOG`: Configure logging with [env_logger](https://docs.rs/env_logger/latest/env_logger/)
- `CONFIGURE`: Flag to execute default `main` function instead of running the fuzzer. Useful for dealing with build systems that cannot cope with instrumented binaries. If there was no original `main` function, the target will exit with `42`

## How-To Include StorFuzz's Data Coverage in your Fuzzer

For LibAFL-based fuzzers it is fairly straightforward to integrate StorFuzz's data coverage:

1. To instrument the target compile the target binary with `libafl_cc::LLVMPasses::StorFuzzCoverage`
2. The coverage data is available via `libafl_targets::std_storfuzz_map_observer`, which creates a `libafl::observers::StdMapObserver`
3. Feedback from the observer is created with `libafl::feedbacks::AflMapFeedback`, which has the policy `DifferentIsNovel` and combines the historic data with new data using an `OrReducer`. To display correct stats this feedback has to be interpreted as a bitmap using `set_is_bitmap(true)`
4. The resulting feedback can then be incorporated in the overall fuzzer feedback
5. The observer must be added to the executor to obtain correct results

---

# LibAFL, the fuzzer library.

 <img align="right" src="https://raw.githubusercontent.com/AFLplusplus/Website/main/static/libafl_logo.svg" alt="LibAFL logo" width="250" heigh="250">

Advanced Fuzzing Library - Slot your own fuzzers together and extend their features using Rust.

LibAFL is written and maintained by

 * [Andrea Fioraldi](https://twitter.com/andreafioraldi) <andrea@aflplus.plus>
 * [Dominik Maier](https://twitter.com/domenuk) <dominik@aflplus.plus>
 * [s1341](https://twitter.com/srubenst1341) <github@shmarya.net>
 * [Dongjia Zhang](https://github.com/tokatoka) <toka@aflplus.plus>
 * [Addison Crump](https://github.com/addisoncrump) <me@addisoncrump.info>
 * [Romain Malmain](https://github.com/rmalmain) <rmalmain@pm.me>

## Why LibAFL?

LibAFL gives you many of the benefits of an off-the-shelf fuzzer, while being completely customizable.
Some highlight features currently include:
- `fast`: We do everything we can at compile time, keeping runtime overhead minimal. Users reach 120k execs/sec in frida-mode on a phone (using all cores).
- `scalable`: `Low Level Message Passing`, `LLMP` for short, allows LibAFL to scale almost linearly over cores, and via TCP to multiple machines.
- `adaptable`: You can replace each part of LibAFL. For example, `BytesInput` is just one potential form input:
feel free to add an AST-based input for structured fuzzing, and more.
- `multi platform`: LibAFL was confirmed to work on *Windows*, *MacOS*, *Linux*, and *Android* on *x86_64* and *aarch64*. `LibAFL` can be built in `no_std` mode to inject LibAFL into obscure targets like embedded devices and hypervisors.
- `bring your own target`: We support binary-only modes, like Frida-Mode, as well as multiple compilation passes for sourced-based instrumentation. Of course it's easy to add custom instrumentation backends.

## Overview

LibAFL is a collection of reusable pieces of fuzzers, written in Rust.
It is fast, multi-platform, no_std compatible, and scales over cores and machines.

It offers a main crate that provide building blocks for custom fuzzers, [libafl](./libafl), a library containing common code that can be used for targets instrumentation, [libafl_targets](./libafl_targets), and a library providing facilities to wrap compilers, [libafl_cc](./libafl_cc).

LibAFL offers integrations with popular instrumentation frameworks. At the moment, the supported backends are:

+ SanitizerCoverage, in [libafl_targets](./libafl_targets)
+ Frida, in [libafl_frida](./libafl_frida)
+ QEMU user-mode and system mode, including hooks for emulation, in [libafl_qemu](./libafl_qemu)
+ TinyInst, in [libafl_tinyinst](./libafl_tinyinst) by [elbiazo](https://github.com/elbiazo)

## Getting started

1. Install the Dependecies
- The Rust development language.  
We highly recommend *not* to use e.g. your Linux distribition package as this is likely outdated. So rather install
Rust directly, instructions can be found [here](https://www.rust-lang.org/tools/install).

- LLVM tools  
The LLVM tools (including clang, clang++) are needed (newer than LLVM 15.0.0 up to LLVM 18.1.3)
If you are using Debian/Ubuntu, again, we highly recommmend that you install the package from [here](https://apt.llvm.org/)

(In `libafl_concolic`, we only support LLVM version newer than 18)

- Cargo-make  
We use cargo-make to build the fuzzers in `fuzzers/` directory. You can install it with

```sh
cargo install cargo-make
```

2. Clone the LibAFL repository with

```sh
git clone https://github.com/AFLplusplus/LibAFL
```

3. Build the library using

```sh
cargo build --release
```

4. Build the API documentation with

```sh
cargo doc
```

5. Browse the LibAFL book (WIP!) with (requires [mdbook](https://rust-lang.github.io/mdBook/index.html))

```sh
cd docs && mdbook serve
```

We collect all example fuzzers in [`./fuzzers`](./fuzzers/).
Be sure to read their documentation (and source), this is *the natural way to get started!*

You can run each example fuzzer with

```sh
cargo make run
```

as long as the fuzzer directory has `Makefile.toml` file.

The best-tested fuzzer is [`./fuzzers/libfuzzer_libpng`](./fuzzers/libfuzzer_libpng), a multicore libfuzzer-like fuzzer using LibAFL for a libpng harness.

## Resources

+ [Installation guide](./docs/src/getting_started/setup.md)

+ [Online API documentation](https://docs.rs/libafl/)

+ The LibAFL book (WIP) [online](https://aflplus.plus/libafl-book) or in the [repo](./docs/src/)

+ Our research [paper](https://www.s3.eurecom.fr/docs/ccs22_fioraldi.pdf)

+ Our RC3 [talk](http://www.youtube.com/watch?v=3RWkT1Q5IV0 "Fuzzers Like LEGO") explaining the core concepts

+ Our Fuzzcon Europe [talk](https://www.youtube.com/watch?v=PWB8GIhFAaI "LibAFL: The Advanced Fuzzing Library") with a (a bit but not so much outdated) step-by-step discussion on how to build some example fuzzers

+ The Fuzzing101 [solutions](https://github.com/epi052/fuzzing-101-solutions) & series of [blog posts](https://epi052.gitlab.io/notes-to-self/blog/2021-11-01-fuzzing-101-with-libafl/) by [epi](https://github.com/epi052)

+ Blogpost on binary-only fuzzing lib libaf_qemu, [Hacking TMNF - Fuzzing the game server](https://blog.bricked.tech/posts/tmnf/part1/), by [RickdeJager](https://github.com/RickdeJager).

+ [A LibAFL Introductory Workshop](https://www.atredis.com/blog/2023/12/4/a-libafl-introductory-workshop), by [Jordan Whitehead](https://github.com/jordan9001)

## Contributing

Please check out [CONTRIBUTING.md](CONTRIBUTING.md) for the contributing guideline.

## Cite

If you use LibAFL for your academic work, please cite the following paper:

```bibtex
@inproceedings{libafl,
 author       = {Andrea Fioraldi and Dominik Maier and Dongjia Zhang and Davide Balzarotti},
 title        = {{LibAFL: A Framework to Build Modular and Reusable Fuzzers}},
 booktitle    = {Proceedings of the 29th ACM conference on Computer and communications security (CCS)},
 series       = {CCS '22},
 year         = {2022},
 month        = {November},
 location     = {Los Angeles, U.S.A.},
 publisher    = {ACM},
}
```

#### License

<sup>
Licensed under either of <a href="LICENSE-APACHE">Apache License, Version
2.0</a> or <a href="LICENSE-MIT">MIT license</a> at your option.
</sup>

<br>

<sub>
Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this crate by you, as defined in the Apache-2.0 license, shall
be dual licensed as above, without any additional terms or conditions.
</sub>
