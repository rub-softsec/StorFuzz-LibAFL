# StorFuzz

This folder contains an example StorFuzz fuzzer tailored for fuzzbench.
It uses the best possible setting, with the exception of a SimpleRestartingEventManager instead of an LlmpEventManager - since fuzzbench is single threaded.
Real fuzz campaigns should consider using multithreaded LlmpEventManager, see the other examples.

## Build

To build this example, run `cargo build --release`.
This will build the fuzzer compilers (`libafl_cc` and `libafl_cpp`) with `src/lib.rs` as fuzzer.
The fuzzer uses the libfuzzer compatibility layer and the SanitizerCoverage runtime functions for coverage feedback.

These can then be used to build libfuzzer harnesses in the software project of your choice.
Finally, just run the resulting binary with `out_dir`, `in_dir`.

In any real-world scenario, you should use `taskset` to pin each client to an empty CPU core, the fuzzer does not pick an empty core automatically.

## How-To Fuzz with StorFuzz

To instrument and fuzz arbitrary C/C++ projects, simply build the fuzzer and use the resulting compiler wrapper with the compile flag `--libafl`.
The fuzzer assumes that the function `LLVMFuzzerTestOneInput` is defined as a harness and that there exists no `main` function.

This could look like this:
```bash
cd ./fuzzers/storfuzz_fuzzbench_in_process
CFLAGS="" CXXFLAGS="" LIBAFL_EDGES_MAP_SIZE="" STORFUZZ_MAP_SIZE="" cargo build --release

./target/release/libafl_cc --libafl <target_without_main.c>

./target --help
```

If there is a `main` function, you might want to compile the fuzzer with `--features no_link_main` and link in the stub runtime provided as `stub_rt.c`.
This may however require changes to the build scripts.

```bash
cd ./fuzzers/storfuzz_fuzzbench_in_process
CFLAGS="" CXXFLAGS="" LIBAFL_EDGES_MAP_SIZE="" STORFUZZ_MAP_SIZE="" cargo build --release --features no_link_main

clang -c stub_rt.c && ar r stub_rt.a stub_rt.o

./target/release/libafl_cc --libafl stub_rt.a <target_with_main.c>

./target --help

# For the original target behavior (e.g., if needed during build process)
CONFIGURE=1 ./target
```
