# Targets

Builds the XNU kernel 12.5 for x86 and ARM64, both development and release. 

# Requirements

Requires the KDK_12.5_21G72.kdk to build the ARM version. Specify as an argument to make like so:

```sh
KDKROOT=/Library/Developer/KDKs/KDK_12.5_21G72.kdk/ make
```

Otherwise you can just make the x64 target:

```sh
make xnu-x64
```

# Confirmed

Confirmed working on macOS Sonoma 14.5 with `Apple clang version 15.0.0 (clang-1500.3.9.4)`.
