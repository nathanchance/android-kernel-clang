# Compiling an Android kernel with Clang


## Background

Google compiles the Pixel 2 kernel with Clang. They shipped the device on Android 8.0 with a 4.4.56 kernel compiled with Clang 4.0 and upgraded to Android 8.1 with a 4.4.88 kernel compiled with Clang 5.0. According to [Google's Clang README](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/#llvm-users), it seems very likely they will ship Android 9.0 with a kernel compiled with Clang 6.0.

Google recently committed to compiling all Chromebook 4.4 kernels with Clang ([commit](https://chromium-review.googlesource.com/809774), [LKML](https://lkml.org/lkml/2018/4/3/567)).

Further information on the motive behind compiling with Clang:

* [Building the kernel with Clang (LWN)](https://lwn.net/Articles/734071/)
* [Compiling Android userspace and Linux kernel with LLVM (YouTube)](https://www.youtube.com/watch?v=6l4DtR5exwo)

TL;DR: Helps find bugs, easier for Google since all of AOSP is compiled with Clang, and better static analysis for higher code quality.


## Requirements

* A compatible kernel (4.4, 4.9, or 4.14 LTS work best)
* arm64 or x86_64
* Patience


## How to compile the kernel with Clang (standalone)

NOTE: I am not going to write this for beginnings. I assume if you are smart enough to pick some commits, you are smart enough to know how to run `git clone` and know the paths of your system.

1. Add the Clang commits to your kernel source (more on that below).
2. Download/build a compatible Clang toolchain (I recommend [AOSP's clang-4053586](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/)) at first).
3. Download/build a compatible GCC toolchain (this is used for assembling and linking - I recommend [AOSP's GCC](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/) at first).
4. Compile the kernel (for arm64, x86_64 is similar - example using AOSP's toolchains):
```bash
make O=out ARCH=arm64 <defconfig>

make -j$(nproc --all) O=out \
                      ARCH=arm64 \
                      CC="<path to clang folder>/bin/clang" \
                      CLANG_TRIPLE=aarch64-linux-gnu- \
                      CROSS_COMPILE="<path to gcc folder>/bin/aarch64-linux-android-"
```

After compiling, you can verify the toolchain used by opening `out/include/generated/compile.h` and looking at the `LINUX_COMPILER` option.


## How to compile the kernel with Clang (inline with a custom ROM)

1. Add the Clang commits to your kernel source (more on that below).
2. Make sure your ROM has [this commit](https://github.com/LineageOS/android_vendor_lineage/commit/da32895b61ef2b3e8899f011110f8eab11da5470)
3. Add the following to your `BoardConfig.mk` file in your device tree: `TARGET_KERNEL_CLANG_COMPILE := true`

To test and verify everything is working:

1. Build a kernel image: `m kernel` or `m bootimage`
2. Open the `out/target/product/*/obj/KERNEL_OBJ/include/generated/compile.h` file and look at the `LINUX_COMPILER` option.


## Getting the Clang patchset

The core Clang patchset is available in two places:

* kernel/common: [4.4](https://android.googlesource.com/kernel/common/+log/f0907aa15ed9f9c7541bb244ed3f52c376ced19c) | [4.9](https://android.googlesource.com/kernel/common/+log/5d15d2e00da4bcb0bcc5e6d27dc18fe1646214f1)
* Chromium: [4.4](https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/sandbox/mka/llvm/v4.4) | [4.9](https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/sandbox/mka/llvm/v4.9)

The other branches in this repository will be dedicated to taking this patchset and enhancing it by fixing/hiding all of the warnings from Clang (from mainline, the Pixel 2, and my own knowledge).

* [msm-4.4](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.4) - based on the latest Oreo branch for the Snapdragon 835 ([kernel.lnx.4.4.r27-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.4/log?h=kernel.lnx.4.4.r27-rel))
    * Compiles with Clang 5.0 (clang-4053586), Clang 6.0 (clang-4691093), and Clang 7.0 (clang-4679922) without any warnings using `arch/arm64/configs/msmcortex-perf_defconfig`

Every time there is a branch update upstream, the branch will be rebased but all commits kept in order.


## Getting help

The preferred method for getting help is either opening an issue on this repository or joining the [Linux kernel newbies chat on Telegram](https://t.me/LinuxKernelNewbies).
