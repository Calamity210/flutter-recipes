---
title: Snapshots
parent: Reverse Engineering
nav_order: 1
---

# Snapshots

The Dart SDK is highly versatile, you can embed Dart code in many different configurations on many different platforms.

The simplest way to run Dart is to use the `dart` executable which just reads dart source files directly like a scripting
language. It includes the primary components we call the front-end (parses Dart code), runtime (provides the environment
for code to run in), and the JIT compiler.

You can also use `dart` to create and execute [snapshots](https://github.com/dart-lang/sdk/wiki/Snapshots), a pre-compiled
form of Dart which is commonly used to speed up frequently used command line tools (like `pub`).

```
#lint shell
ping@debian:~/Desktop$ time dart hello.dart
Hello, World!

real    0m0.656s
user    0m0.920s
sys     0m0.084s

ping@debian:~/Desktop$ dart --snapshot=hello.snapshot hello.dart
ping@debian:~/Desktop$ time dart hello.snapshot
Hello, World!

real    0m0.105s
user    0m0.208s
sys     0m0.016s
```

As you can see, the start-up time is significantly lower when you use snapshots.

The default snapshot format is [kernel](https://github.com/dart-lang/sdk/wiki/Kernel-Documentation), an intermediate
representation of Dart code equivalent to the AST.

When running a Flutter app in debug mode, the flutter tool creates a kernel snapshot and runs it in your android app
with the debug runtime + JIT. This gives you the ability to debug your app and modify code live at runtime with hot
reload.

Unfortunately for us, using your own JIT compiler is frowned upon in the mobile industry due to increased concerns of
RCEs. iOS actually prevents you from executing dynamically generated code like this entirely.

There are two more types of snapshots though, `app-jit` and `app-aot`, these contain compiled machine code that can be
initialized quicker than kernel snapshots but aren't cross-platform.

The final type of snapshot, `app-aot`, contains only machine code and no kernel. These snapshots are generated using the
`gen_snapshots` tool found in `flutter/bin/cache/artifacts/engine/<arch>/<target>/`, more on that later.

They are a little more than just a compiled version of Dart code though, in fact they are a full "snapshot" of the VMs
heap just before main is called. This is a unique feature of Dart and one of the reasons it initializes so quickly
compared to other runtimes.

Flutter uses these AOT snapshots for release builds, you can see the files that contain them in the file tree for an
Android APK built with `flutter build apk`:

```
#lint shell
ping@debian:~/Desktop/app/lib$ tree .
.
├── arm64-v8a
│   ├── libapp.so
│   └── libflutter.so
└── armeabi-v7a
    ├── libapp.so
    └── libflutter.so
```

Here you can see the two libapp.so files which are a64 and a32 snapshots as ELF binaries.

The fact that `gen_snapshots` outputs an ELF / shared object here might be a bit misleading, it does not expose dart
methods as symbols that can be called externally. Instead, these files are containers for the "clustered snapshot" format
but with compiled code in the separate executable section, here is how they are structured:

```
#lint shell
ping@debian:~/Desktop/app/lib/arm64-v8a$ aarch64-linux-gnu-objdump -T libapp.so

libapp.so:     file format elf64-littleaarch64

DYNAMIC SYMBOL TABLE:
0000000000001000 g    DF .text  0000000000004ba0 _kDartVmSnapshotInstructions
0000000000006000 g    DF .text  00000000002d0de0 _kDartIsolateSnapshotInstructions
00000000002d7000 g    DO .rodata        0000000000007f10 _kDartVmSnapshotData
00000000002df000 g    DO .rodata        000000000021ad10 _kDartIsolateSnapshotData
```

The reason why AOT snapshots are in shared object form instead of a regular snapshot file is because machine code
generated by `gen_snapshot` needs to be loaded into executable memory when the app starts and the nicest way to do that
is through an ELF file.

With this shared object, everything in the `.text` section will be loaded into executable memory by the linker allowing
the Dart runtime to call into it at any time.

You may have noticed there are two snapshots: the VM snapshot and the Isolate snapshot.

DartVM has a second isolate that does background tasks called the vm isolate, it is required for `app-aot` snapshots since
the runtime can't dynamically load it in as the `dart` executable would.