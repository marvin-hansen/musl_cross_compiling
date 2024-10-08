#  Replicates BAZEL/MUSL ISSUE 

**Important:**

This example code replicates an issue that causes MUSL to build statically linked binaries only on MacOS, but fails to do so on Linux X86. The exact reasons are discussed in [GH issue 2726](https://github.com/bazelbuild/rules_rust/issues/2726).


On MacOS, the build produces statically linked binary, which you can verify:

`file -L target-bzl/out/darwin_arm64-fastbuild/bin/musl_scratch/bin_linux_x86_64_musl/bin`


`target-bzl/out/darwin_arm64-fastbuild/bin/musl_scratch/bin_linux_x86_64_musl/bin: 
ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), static-pie linked, with debug_info, not stripped
`

However, the same build on Linux (Ubuntu 20.04) results in a dynamically linked binary:

`file -L target-bzl/out/linux_x86_64_musl-fastbuild/bin/musl_scratch/bin`

`target-bzl/out/linux_x86_64_musl-fastbuild/bin/musl_scratch/bin:
ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), 
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, 
BuildID[md5/uuid]=272ae7184ca510eb7277f7c195a155d9, with debug_info, not stripped`

**x86_64 Toolchain resolution:**

Host toolchain resolves correctly. For example, on Ubuntu x86_64:

      ToolchainResolution: Recap of selected @@bazel_tools//tools/cpp:toolchain_type toolchains for target platform @@platforms//host:host:

      ToolchainResolution:   Selected @@toolchains_llvm~~llvm~llvm_toolchain//:cc-clang-x86_64-linux to run on execution platform @@platforms//host:host

linux_arm64_musl target toolchain resolves correctly to MUSL. For example, on Ubuntu x86_64:

      INFO: ToolchainResolution: Target platform //build/platforms:linux_arm64_musl: 

      Selected execution platform @@platforms//host:host, type @@rules_rust~//rust:toolchain_type 
      
      -> toolchain @@rules_rust~~rust~rust_linux_x86_64__aarch64-unknown-linux-musl__stable_tools//:rust_toolchain, type @@bazel_tools//tools/cpp:toolchain_type 

      -> toolchain @@toolchains_musl~~toolchains_musl~musl-1_2_3-platform-x86_64-unknown-linux-gnu-target-aarch64-linux-musl//:musl-1_2_3-platform-x86_64-unknown-linux-gnu-target-aarch64-linux-musl

x86_64_musl, however, resolves incorrectly to clang-x86_64-linux

      INFO: ToolchainResolution: Target platform //build/platforms:linux_arm64_musl: Selected execution platform @@platforms//host:host, type @@rules_rust~//rust:toolchain_type

      -> toolchain @@rules_rust~~rust~rust_linux_x86_64__aarch64-unknown-linux-musl__stable_tools//:rust_toolchain, type @@bazel_tools//tools/cpp:toolchain_type 

      -> toolchain @@toolchains_musl~~toolchains_musl~musl-1_2_3-platform-x86_64-unknown-linux-gnu-target-aarch64-linux-musl//:musl-1_2_3-platform-x86_64-unknown-linux-gnu-target-aarch64-linux-musl

**Apple aarch64 (Apple Silicon) Toolchain resolution:**


Host toolchain resolves correctly on Mac:

       Target platform @@platforms//host:host: Selected execution platform @@platforms//host:host, type @@rules_rust~//rust:toolchain_type 
      -> toolchain @@rules_rust~~rust~rust_darwin_aarch64__aarch64-apple

linux_arm64_musl target toolchain resolves correctly to MUSL on Mac:

       ToolchainResolution: Target platform //build/platforms:linux_arm64_musl: Selected execution platform @@platforms//host:host, type @@bazel_tools//tools/cpp:toolchain_type 

      -> toolchain @@toolchains_musl~~toolchains_musl~musl-1_2_3-platform-aarch64-apple-darwin-target-aarch64-linux-musl//:musl-1_2_3-platform-aarch64-apple-darwin-target-aarch64-linux-musl


linux_x86_64_musl, target toolchain resolves correctly to MUSL on Mac: 

      ToolchainResolution: Target platform //build/platforms:linux_x86_64_musl: Selected execution platform @@platforms//host:host, type @@bazel_tools//tools/cpp:toolchain_type 

      -> toolchain @@toolchains_musl~~toolchains_musl~musl-1_2_3-platform-aarch64-apple-darwin-target-x86_64-linux-musl//:musl-1_2_3-platform-aarch64-apple-darwin-target-x86_64-linux-musl


## Requirements

MacOS:

* Install xcode command line tools
* Install git, bazelisk, and docker

Ubuntu 20.04
* install build-essentials, libstdc++6 libtinfo5
* Install git, bazelisk, and [docker](https://phoenixnap.com/kb/install-docker-on-ubuntu-20-04)

You may have to un-minify the official Ubuntu 20.04 image to get all the usual gnu-tools installed. 
To do so, simply run in a terminal and follow the prompt. 

`unminimize`

Test the example code:

`bazel test //...`

These tests pass on MacOs (14), but fail on Ubuntu 18.04 and 20.04. 

* Tests are defined here: [BUILD.bazel](musl_scratch/BUILD.bazel)
* Test script: [test_platform](musl_scratch/test_platform.sh)


Build the example code:

`bazel build //...`

To apply compiler optimization and striping the binary:

`bazel build -c opt //...`

The container supports both, Intel and ARM, architectures via a multi-arch OCI container image. To build the image:

`
bazel build //musl_scratch:image_index
`

To publish the container, you would have to run:

`bazel run //musl_scratch:push`

However, this results in an error because no registry is defined in the example.
You have to define your target registry in the [binary BUILD](musl_scratch/BUILD.bazel) file
before you can publish your image. For details of how to configure a container registry,
please [consult the official documentation.](https://github.com/bazel-contrib/rules_oci/blob/main/docs/push.md)


## MUSL specifics

When you develop applications in Rust that target the MUSL, there are few important considerations:

### 1) Replace SSL dependencies with RustTLS

Because OpenSSL depends on some system libraries, any crate that depends on the default SSL will not compile with MUSL. Luckily, all major crates offer a RustTLS feature flag, so this is resolved by setting 
the correct feature flags in the Rust dependencies. In general, it is easier to disable all default features, and then only enable the features that are actually needed. See the dependencies in the MODULE.bazel file as an example.

### 2) Avoid major C/C++ dependencies whenever possible

This is a non-trivial constraint, but not every C/C++ target compiles with MUSL. Unfortunately, there is no
other way to find out than actually trying to build it. As a rule of thumb, it is best to avoid Rust crates that just wrap an existing C/C++ whenever possible. Many Rust crates have some implicit C/C++ dependencies, but many of these actually build with MUSL. However, the larger the underlying C/C++ library, the higher the chance that something breaks. 

The exact problem isn't C/C++ per se, it's the problem that, just one call in std libc that tries to open a file or something else that's not supported in MUSL binaries simply breaks the build. This is equally true for Rust crates, but many Rust crates use feature flags to mitigate IO related issues. Some code out there actually requires system IO, and that just doesn't compile with MUSL. If somehow one or more major library is needed and cannot be build with MUSL, it's sensible to just move on, a build a dynamically linked binary and use a distroless image instead, as shown in the [OCI example](../07-oci-container).

### 3) MUSL builds best with network only applications

When building network only applications, then MUSL builds most of the time out of the box. However, the only remaining issue comes from DNS resolution. For historical reasons, the DNS resolution implementation in the standard library expects a HOST file with a default DNS server. Because it's a file, this doesnt work with MUSL. However, if you use a custom DNS resolver such as the excellent [Hickory DNS crate](https://github.com/hickory-dns/hickory-dns), you simply switch off the default features in the crate dependency and then construct the hickory resolver with a custom configuration containing the network DNS server. From there, MUSL usually compiles your code. 


### 4) Test cross compiled MUSL Containers on CI or with Docker

There is non-zero chance of a segfault for any number of reasons. It's rare, but Tte compiler cannot catch those corner cases so the best you can do is to pull the actual image, and start it either locally or on your CI and run some basic integration tests. Integration tests are generally considered best practice, 
but in case of MUSL binaries, you really have to double check if the container starts correctly. The section
about configuring a [custom memory allocator](#custom-memory-allocator) elaborates when and how to avoid the segfault pitfall. 

## Setup

In Rust, because of its deep interoperability with C,
a few more steps are required to build a statically linked binary packaged in a scratch container.


The initial setup is similar to the [cross compilation example](../02-hello-cross). 
However, in addition to LLVM and platform support, we also add the MUSL toolchain.

### Rules

Let's start with adding the requires rules to the MODULE.bazel file:

```Starlark
# https://github.com/bazelbuild/rules_rust/releases
bazel_dep(name = "rules_rust", version = "0.46.0")
# Rules for OCI container images
# https://github.com/bazel-contrib/rules_oci/releases
bazel_dep(name = "rules_oci", version = "1.7.6")
# https://github.com/bazelbuild/rules_pkg/releases
bazel_dep(name = "rules_pkg", version = "0.10.1")
#
# Rules for musl cross compilation
# https://github.com/bazel-contrib/musl-toolchain/releases
bazel_dep(name = "toolchains_musl", version = "0.1.16", dev_dependency = True)
# https://github.com/bazelbuild/platforms/releases
bazel_dep(name = "platforms", version = "0.0.10")
# https://github.com/bazel-contrib/toolchains_llvm
bazel_dep(name = "toolchains_llvm", version = "1.0.0")
```


## LLVM Configuration

Next, you have to configure the LLVM toolchain because rules_rust still needs a cpp toolchain for cross compilation and
you have to add the specific platform triplets to the Rust toolchain. Suppose you want to compile a Rust binary that
supports linux on both, X86 and ARM. In that case, you have to setup three LLVM toolchains:

1) LLVM for the host
2) LLVM for X86
3) LLVM for ARM (aarch64)

For the host LLVM, you just specify a LLVM version and then register the toolchain as usual. 
The target LLVM toolchains, however, have dependencies on system libraries for the target platform. 
Therefore, it is required to download a so- called sysroot that contains a root file system 
with all those system libraries for the specific target platform.
To download the sysroot files, please add the following to your MODULE.bazel

```Starlark
http_archive = use_repo_rule("@bazel_tools//:http.bzl", "http_archive")

# Both, cross compilation and MUSL still need a C/C++ toolchain with sysroot.
_BUILD_FILE_CONTENT = """
filegroup(
  name = "{name}",
  srcs = glob(["*/**"]),
  visibility = ["//visibility:public"],
)
"""

# INTEL/AMD64 Sysroot
# LastModified: 2024-04-26T19:15
# https://commondatastorage.googleapis.com/chrome-linux-sysroot/
http_archive(
    name = "org_chromium_sysroot_linux_x64",
    build_file_content = _BUILD_FILE_CONTENT.format(name = "sysroot"),
    sha256 = "5df5be9357b425cdd70d92d4697d07e7d55d7a923f037c22dc80a78e85842d2c",
    urls = ["https://commondatastorage.googleapis.com/chrome-linux-sysroot/toolchain/4f611ec025be98214164d4bf9fbe8843f58533f7/debian_bullseye_amd64_sysroot.tar.xz"],
)

# ARM 64 Sysroot
# LastModified: 2024-04-26T18:33
# https://commondatastorage.googleapis.com/chrome-linux-sysroot/
http_archive(
    name = "org_chromium_sysroot_linux_aarch64",
    build_file_content = _BUILD_FILE_CONTENT.format(name = "sysroot"),
    sha256 = "d303cf3faf7804c9dd24c9b6b167d0345d41d7fe4bfb7d34add3ab342f6a236c",
    urls = ["https://commondatastorage.googleapis.com/chrome-linux-sysroot/toolchain/906cc7c6bf47d4bd969a3221fc0602c6b3153caa/debian_bullseye_arm64_sysroot.tar.xz"],
)
```

Here, we declare to new http downloads that retrieve the sysroot for linux_x64 (Intel/AMD) and linux_aarch64 (ARM/Apple Silicon). Note, these are only
sysroots, that means you have to configure LLVM next to use these files. As mentioned earlier, three LLVM toolchains
needs to be configured and to do that, please add the following to your MODULE.bazel

```Starlark
# LLVM Versions and platforms
# https://github.com/bazel-contrib/toolchains_llvm/blob/master/toolchain/internal/llvm_distributions.bzl
LLVM_VERSIONS = {
    "": "16.0.0",
    "darwin-aarch64": "16.0.3", # Apple Silicon M-series Macs 
    "darwin-x86_64": "15.0.7",  # Intel Macs
}
# Host LLVM toolchain.
llvm.toolchain(
    name = "llvm_toolchain",
    llvm_versions = LLVM_VERSIONS,
)
use_repo(llvm, "llvm_toolchain", "llvm_toolchain_llvm")

# X86 LLVM Toolchain with sysroot.
# https://github.com/bazel-contrib/toolchains_llvm/blob/master/tests/WORKSPACE.bzlmod
llvm.toolchain(
    name = "llvm_toolchain_x86_with_sysroot",
    llvm_versions = LLVM_VERSIONS,
)
llvm.sysroot(
    name = "llvm_toolchain_x86_with_sysroot",
    label = "@org_chromium_sysroot_linux_x64//:sysroot",
    targets = ["linux-x86_64"],
)
use_repo(llvm, "llvm_toolchain_x86_with_sysroot")

#
# ARM (aarch64) LLVM Toolchain with sysroot.
# https://github.com/bazelbuild/rules_rust/blob/main/examples/bzlmod/cross_compile/WORKSPACE.bzlmod
llvm.toolchain(
    name = "llvm_toolchain_aarch64_with_sysroot",
    llvm_versions = LLVM_VERSIONS,
)
llvm.sysroot(
    name = "llvm_toolchain_aarch64_with_sysroot",
    label = "@org_chromium_sysroot_linux_aarch64//:sysroot",
    targets = ["linux-aarch64"],
)
use_repo(llvm, "llvm_toolchain_aarch64_with_sysroot")

# Register all LLVM toolchains
register_toolchains("@llvm_toolchain//:all")
```

The LLVM_VERSIONS defines a default version (16) that applies to every system not defined otherwise (i.e. Linux) 
and two custom versions, one (16.0.3) for Apple Silicon Macs (i.e. M1/M2/M3...) 
and another one (15.0.7) for older Intel based Macs. 
The reason for the differences in LLVM version numbers are two fold;
for once the LLVM toolchain project stopped supporting Intel Macs a while ago, so 15.0.7
is the last available version. Second, the LLVM version is not only depending on the target, 
but also the build host. Many CI systems still run on older, but stable, Linux distros
such as Ubuntu 18.04 or 20.04 and LLVM 15 and 16 are supporting these distros whereas newer LLVM versions
have dropped support for older distros a while ago. 

However, if your build and production system runs on a newer distro and does not need Apple support,
then you can update to a more recent LLVM version, but the same rule applies that
the LLVM version must support both, the host and the target. For a complete [list off all LLVM releases and supported platforms, see the official list.](https://github.com/bazel-contrib/toolchains_llvm/blob/master/toolchain/internal/llvm_distributions.bzl) It is possible to pin different targets to different LLVM
versions, as shown above. [Please see the official documentation for details](https://github.com/bazel-contrib/toolchains_llvm/tree/master?tab=readme-ov-file#per-host-architecture-llvm-version).


### Rust Toolchain Configuration

The Rust toolchain only need to know the additional platform triplets to download the matching toolchains. To do so, add
or or modify your MODULE.bazel with the following entry:

```Starlark
# Rust toolchain
RUST_EDITION = "2021"
RUST_VERSION = "1.79.0"

rust = use_extension("@rules_rust//rust:extensions.bzl", "rust")
rust.toolchain(
    edition = RUST_EDITION,
    versions = [RUST_VERSION],
    extra_target_triples = [
        "x86_64-unknown-linux-musl",
        "aarch64-unknown-linux-musl",
    ],
)
use_repo(rust, "rust_toolchains")
register_toolchains("@rust_toolchains//:all")
```

You find the exact platform triplets in
the [Rust platform support documentation](https://doc.rust-lang.org/nightly/rustc/platform-support.html).
Next, you have to configure the target platform.


### MUSL Toolchain Configuration

The MUSL toolchain needs a target configuration that tells Bazel when to use MUSL. To do so, 
please add the following to your MODULE.bazel.

```Starlark
# MUSL toolchain
toolchains_musl = use_extension("@toolchains_musl//:toolchains_musl.bzl", "toolchains_musl", dev_dependency = True)
toolchains_musl.config(
    extra_target_compatible_with = ["@//build/linker:musl"],
)
```
The target `build/linker/:musl` will be defined next. 


### Platform Configuration

Before the MUSL platform can be configured, we need to add a custom linker configuration to redirect the linker to
MUSL. To do so, add an empty BUILD file in the following path:

`build/linker/BUILD.bazel`

Then add the following content to configure the linker for MUSL.

```Starlark
package(default_visibility = ["//visibility:public"])

constraint_setting(
    name = "linker",
    default_constraint_value = ":unknown",
)

constraint_value(
    name = "musl",
    constraint_setting = ":linker",
)

# Default linker for anyone not setting the linker to `musl`.
# You shouldn't ever need to set this value manually.
constraint_value(
    name = "unknown",
    constraint_setting = ":linker",
)
```

Then, you edit your platform configuration, assumed to be in the following path:

`build/platforms/BUILD.bazel`

Add the following entries to configure MUSL:

```Starlark
package(default_visibility = ["//visibility:public"])

platform(
    name = "linux_x86_64_musl",
    constraint_values = [
        "@//build/linker:musl",
        "@platforms//cpu:x86_64",
        "@platforms//os:linux",
    ],
)

platform(
    name = "linux_arm64_musl",
    constraint_values = [
        "@//build/linker:musl",
        "@platforms//cpu:arm64",
        "@platforms//os:linux",
    ],
)
```

Notice that the path of the linker is set to `//build/linker` so if you chose a different folder, you have to update that path accordingly.

The default visibility at the top of the file means that all targets in this BUILD file will be public by default, which
is sensible because cross-compilation targets are usually used across the entire project.

It is important to recognize that the platform rules use the constraint values to map those constraints to the target
triplets of the Rust toolchain. If you somehow see errors that says some crate couldn't be found with triple xyz, then
one of two things happened.

Either you forgot to add a triple to the Rust toolchain. Unfortunately, the error message
doesn't always tell you the correct triple that is missing. However, in that case you have to double check if for each
specified platform a corresponding Rust extra_target_triples has been added. If one is missing, add it and the error
goes away.

A second source of error is if the platform declaration contains a typo, for example,
cpu:arch64 instead of cpu:aarch64. You have to be meticulous in the platform declaration to make everything work
smoothly.


### Custom Memory allocator.

There is a long-standing multi threading performance issue in MUSL's default memory allocator
that causes a
significant [performance drop of at least 10x or more compared to the default memory allocator in Linux.](https://www.linkedin.com/pulse/testing-alternative-c-memory-allocators-pt-2-musl-mystery-gomes)
The real source of the performance degradation is thread contention is in the malloc implementation of musl.
One known workaround is
to [patch the memory allocator in place](https://www.tweag.io/blog/2023-08-10-rust-static-link-with-mimalloc/)
using a rather lesser known assembly tool.
A unique alternative Rust offers is the global_allocator trait that, once overwritten with
a custom allocator, simply replaces the memory allocator Rust uses.

There are like three alternative memory allocators available for Rust,

* [jemallocator](https://crates.io/crates/jemallocator)
* [mimalloc](https://lib.rs/crates/mimalloc)
* [snmalloc](https://lib.rs/crates/snmalloc-rs)

Notice, Jemalloc has
a[ known segfault issue when you target embedded platforms](https://github.com/clux/muslrust/issues/142#issuecomment-2152638811)
where the memory page size varies.
Specifically, if you compile with Jemalloc on an Apple Silicon for usage on a Raspberry Pi,
Jemalloc may segfault on the Raspberry Pi due to different memory page sizes because
Jemalloc bakes the memory page size into the final binary.
Mimalloc doesn't have this problem, and has performance comparable to Jemalloc.
Therefore, for embedded devices, Mimalloc is the best choice.

However, on x86 (Intel / AMD), this issue does not exists, and any of the memory allocators listed above works just
fine.[A benchmarks show that both](https://github.com/rust-lang/rust-analyzer/issues/1441), Jemalloc and Mimalloc
demonstrate comparable performance so for X86, you can pick any of the two.

For this example, I chose Jemalloc from the Free/NetBSD distro because
it is a robust and battle tested memory allocators out there that still delivers excellent performance under heavy
multi-threading workload.

Make sure jemallocator is declared a dependency in your MODULE.bazelmod file:

```Starlark
###############################################################################
# R U S T  C R A T E S
###############################################################################
crate = use_extension("@rules_rust//crate_universe:extension.bzl", "crate")
#
# Custom Memory Allocator
crate.spec(package = "jemallocator", version = "0.5.4")
# ... other crate dependencies.
```

Next, you add a new memory allocator by adding the following lines to your main.rs file:

```Rust
use jemallocator::Jemalloc;

// Jemalloc overwrites the default memory allocator. This fixes a performance issue in the MUSL.
// https://www.linkedin.com/pulse/testing-alternative-c-memory-allocators-pt-2-musl-mystery-gomes
#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;

#[tokio::main]
async fn main() {
  // ...
}
```


At this point, you want to run a full build and check for any errors.

`bazel build //...`

Also run a full release build to double check that the optimization settings work:

`bazel build -c opt //...`


## Scratch image

The new rules_oci build container images in Bazel without Docker. Before you build a container,
you have to add base image.
Previous examples have used the lightweight Distroless containers,
but since the binary has been compiled statically, all you need is a scratch image.
To declare a scratch image, add the following to your MODULE.bazel file:

```Starlark
###############################################################################
#  O C I  B A S E  I M A G E
###############################################################################
oci = use_extension("@rules_oci//oci:extensions.bzl", "oci")
#
# https://hub.docker.com/r/hansenmarvin/rust-scratch/tags
oci.pull(
    name = "scratch",
    digest = "sha256:c6d1c2b62a454d6c5606645b5adfa026516e3aa9213a6f7648b8e9b3cc520f76",
    image = "index.docker.io/hansenmarvin/rust-scratch",
    platforms = ["linux/amd64", "linux/arm64"],
)
use_repo(oci, "scratch")
```

In this example, a custom scratch image is used. You can inspect the Docker build file on
[its public repository](https://github.com/marvin-hansen/rust-scratch). As you can the in
the [Dockerfile](https://github.com/marvin-hansen/rust-scratch/blob/main/Dockerfile),
SSL certificates are copied from the base image to ensure encrypted connections
work as expected. However, this is also a prime example of how an
attacker could sneak in bogus certificates via sloppy supply chain security.

Therefore, it is generally recommended to build and use your own scratch image
instead of relying on unknown third parties.

The process to build a multi_arch scratch image to hold your statically linked binary takes a few steps:

1) Compress the Rust binary as tar
2) Build container image from the tar
3) Build a multi_arch image for the designated platform(s)
4) Generate a oci_image_index
5) Tag the final multi_arch image

Building a multi_arch image, however, requires a platform transition. Without much ado,
just create new empty BUILD file in a folder containing all your custom BAZEL rules and toolchains, say:

`build/transition.bzl`

And then add the following content:

```Starlark
"a rule transitioning an oci_image to multiple platforms"

def _multiarch_transition(settings, attr):
    return [
        {"//command_line_option:platforms": str(platform)}
        for platform in attr.platforms
    ]

multiarch_transition = transition(
    implementation = _multiarch_transition,
    inputs = [],
    outputs = ["//command_line_option:platforms"],
)

def _impl(ctx):
    return DefaultInfo(files = depset(ctx.files.image))

multi_arch = rule(
    implementation = _impl,
    attrs = {
        "image": attr.label(cfg = multiarch_transition),
        "platforms": attr.label_list(),
        "_allowlist_function_transition": attr.label(
            default = "@bazel_tools//tools/allowlists/function_transition_allowlist",
        ),
    },
)
```

Next, you need a custom rule to tag your container. In a hermetic build, you can't rely on timestamps because these
changes regardless of whether the build has changed. Strictly speaking, timestamps as tags could be made possible in
Bazel, but it is commonly discouraged. Also, immutable container tags are increasingly encouraged to prevent accidental
pulling of a different image that has the same tag as the previous one but contains breaking changes relative to the
previous image. Instead, you want unique tags that only change when the underlying artifact has changed. Turned out,
rules_oci already generates a sha256 for each OCI image so a simple tag rule would be to extract this has and trim to,
say 7 characters and use this short hash as unique and immutable tag.

To crate this rule, crate new file, say,

`build/container.bzl`

Then add the following rule:

```Starlark
def _build_sha265_tag_impl(ctx):

    # Both the input and output files are specified by the BUILD file.
    in_file = ctx.file.input
    out_file = ctx.outputs.output

    # No need to return anything telling Bazel to build `out_file` when
    # building this target -- It's implied because the output is declared
    # as an attribute rather than with `declare_file()`.
    ctx.actions.run_shell(
        inputs = [in_file],
        outputs = [out_file],
        arguments = [in_file.path, out_file.path],
        command = "sed -n 's/.*sha256:\\([[:alnum:]]\\{7\\}\\).*/\\1/p' < \"$1\" > \"$2\"",
    )

build_sha265_tag = rule(
    doc = "Extracts a 7 characters long short hash from the image digest.",
    implementation = _build_sha265_tag_impl,
    attrs = {
        "image": attr.label(
            allow_single_file = True,
            mandatory = True,
        ),
        "input": attr.label(
            allow_single_file = True,
            mandatory = True,
            doc = "The image digest file. Usually called image.json.sha256",
        ),
        "output": attr.output(
            doc = "The generated tag file. Usually named _tag.txt"
        ),
    },
)

```

Then, you import this rule together with the multi_arch and some others rules
to build a container for your binary target.

```Starlark
load("@rules_rust//rust:defs.bzl", "rust_binary", "rust_doc", "rust_doc_test")
# OCI Container Rules
load("@rules_pkg//pkg:tar.bzl", "pkg_tar")
load("@rules_oci//oci:defs.bzl", "oci_image", "oci_push",  "oci_image_index")
# Custom container macro
load("//:build/container.bzl", "build_sha265_tag")
# Custom platform transition macro
load("//:build/transition.bzl", "multi_arch")
```

Remember, the steps to build a multi_arch image are the following:

1) Compress the Rust binary as tar
2) Build container image from the tar
3) Build a multi_arch image for the designated platform(s)
4) Generate a oci_image_index
5) Tag the final multi_arch image

Let's start with the first three steps. Add the following to your binary target:

```Starlark
# Compress binary to a layer using pkg_tar
pkg_tar(
    name = "tar",
    srcs = [":bin"],
)

# Build container image
# https://github.com/bazel-contrib/rules_oci/blob/main/docs/image.md
oci_image(
    name = "image",
    base = "@scratch",
    tars = [":tar"],
    entrypoint = ["/bin"],
    exposed_ports = ["3232"],
    visibility = ["//visibility:public"],
)

# Build multi-arch images
multi_arch(
    name = "multi_arch_images",
    image = ":image",
    platforms = [
        "//build/platforms:linux_x86_64_musl",
        "//build/platforms:linux_arm64_musl",
    ],
)
```

**A few notes:**

1) Make sure the tar package references the binary.
2) Make sure the container image exposes the exact same ports as the binary uses.
3) The base image, scratch, of the container.
4) Make sure the path and labels used of the platforms in the multi_arch match exactly the folder structure you have
   defined in the previous steps.

Next, lets add the remaining two steps plus a declaration to push the final image to a container registry.

```Starlark
# Build a container image index.
oci_image_index(
    name = "image_index",
    images = [
        ":multi_arch_images",
    ],
    visibility = ["//visibility:public"],
)

# Build an unique and immutable image tag based on the image SHA265 digest.
build_sha265_tag(
    name = "tags",
    image = ":image_index",
    input = "image.json.sha256",
    output = "_tag.txt",
)

# Publish multi-arch with image index to registry
oci_push(
    name = "push",
    image = ":image_index",
    repository = "my.registry.com/musl",
    remote_tags = ":tags",
    visibility = ["//visibility:public"],
)
```

**Important details:**

1) The oci_image_index always references the multi_arch rule even if you only compile for one platform.
2) The oci_image_index is public because that target is what you call when you build the container without publishing
   it.
3) The build_sha265_tag rule uses the image.json.sha256 file from the original image. This is on purpose because the
   sha265 is only generated for images during the build, but not for the index file.
4) The oci_push references the image_index to ensure a multi arch image will be published.
5) oci_push is public because that is the target you call to publish you container.

For details of how to configure a container registry,
please [consult the official documentation.](https://github.com/bazel-contrib/rules_oci/blob/main/docs/push.md)

### Custom Container Macro

The scratch image configuration feels quite verbose and this configuration becomes quickly tedious
when you build a large number of containers that roughly follow the same blueprint and only differ
by a handful of parameters such as exposed ports, the specific platform(s) and similar.
In that case, it is advisable to write a custom macro that reduces the boilerplate code to a
bare minimum.

In short, open or add a file in

`build/container.bzl`

And add the following content:

```Starlark
load("@rules_pkg//pkg:tar.bzl", "pkg_tar")
load("@rules_oci//oci:defs.bzl", "oci_image", "oci_image_index")
load("//:build/transition.bzl", "multi_arch")

# Build a Bazel Macro
# https://belov.nz/posts/bazel-rules-macros/

def build_multi_arch_image(
    name,
    base,
    srcs,
    exposed_ports = [],
    platforms = [],
    visibility=None
    ):

    # https://codilime.com/blog/bazel-build-system-build-containerized-applications/
    entry_point = "bin"
    layer_name = "tar_layer"

    # Compress binary to a layer using pkg_tar
    pkg_tar(
        name = layer_name,
        srcs = srcs,
    )

    # Build container image
    oci_image(
        name = "image",
        base = base,
        tars = [layer_name],
        entrypoint = ["/{}".format(entry_point)],
        exposed_ports = exposed_ports,
    )

    # Build multi-arch images for the provided platforms
    multi_arch(
        name = "multi_arch_images",
        image = ":image",
        platforms = platforms,
    )
    
    # Build a container image index.
    oci_image_index(
        name = name,
        images = [
            ":multi_arch_images",
        ],
        visibility = visibility,
    )
```

This macro rule turns the previous boilerplate into a template you can import and use
to build your custom MUSL scratch image for your binary targets. This usually simplifies maintenance
because the bulk of changes can be made in the macro instead of each targets. Note, if you want
to enforce a specific base image, say for security reasons, you can declare it in the macro instead of using a
parameter. You still need the tag rule from before because the tags apply to the push rule.
With the new macro in place, you import the macro and the tag rule in your target BUILD:

```Starlark
# Normal Rust rules 
load("@rules_rust//rust:defs.bzl", "rust_binary", "rust_doc", "rust_doc_test")
# OCI Push rule
load("@rules_oci//oci:defs.bzl",  "oci_push")
# Custom container macro
load("//:build/container.bzl", "build_multi_arch_image", "build_sha265_tag")
```

With these imports in place, you then use the rules as shown below:

```Starlark
# Build normal Rust binary
rust_binary(
    name = "bin",
    # ...
)

# 1) Build musl multi arch container image
build_multi_arch_image(
    name = "image_index",
    base = "@scratch",
    srcs = [":bin"],
    exposed_ports = ["3232"],
    platforms = [
        "//build/platforms:linux_x86_64_musl",
        "//build/platforms:linux_arm64_musl",
    ],
    visibility = ["//visibility:public"],
)

# 2) Tag image based on the image SHA265 digest.
build_sha265_tag(
    name = "remote_tag",
    image = ":image_index",
    input = "image.json.sha256",
    output = "_tag.txt",
)

# 3) Publish multi-arch with image index to registry
oci_push(
    name = "push",
    image = ":image_index",
    repository = "my.registry.com/musl",
    remote_tags = ":tags",
    visibility = ["//visibility:public"],
)
```

### Discussion

With the macro, building a multi-arch container is a three step process, build, tag, and push.
As stated before, the macro only makes sense when you have either a larger number
of very similar container to build or you have to enforce a number of (security) polices across the entire project.

On the other hand, if you have to build very different or complex multi-layer containers,
than the previous approach of defining each stage and container layer manually gives much more fine grained control
and is therefore the preferred process. 

Lastly, the custom macros for image tagging or building multi-arch containers  
only serve as examples. In general, it is recommended to write custom macros only
to support custom requirements that are too specific for inclusion in the default rules. 
