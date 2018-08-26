# Compiling an Android kernel with Clang


## Background

Google compiles the Pixel 2 kernel with Clang. They shipped the device on Android 8.0 with a kernel compiled with Clang 4.0 ([build.config commit](https://android.googlesource.com/kernel/msm/+/1282b122796d12f42e650216b40172eae4dc4162) and [prebuilt kernel commit](https://android.googlesource.com/device/google/wahoo-kernel/+/8c65a7e83f8bc602a05f077d221d4648db189ef8)) and upgraded to Android 8.1 with a kernel compiled with Clang 5.0 ([build.config commit](https://android.googlesource.com/kernel/msm/+/1eaefe4575b5c39dacb724344d427e34d12c15df) and [prebuilt kernel commit](https://android.googlesource.com/device/google/wahoo-kernel/+/e03cfae0fa716983ae7af64bf8f1c50003637ffb)). According to [Google's Clang README](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/#llvm-users), it seems very likely they will ship Android 9.0 with a kernel compiled with Clang 6.0.

Google recently started compiling all Chromebook 4.4 kernels with Clang in R67 ([commit](https://chromium-review.googlesource.com/809774), [LKML](https://lkml.org/lkml/2018/4/3/567)).

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

The core Clang patchset comes from mainline and it is backported to 4.4 and 4.9 in two places:

* kernel/common: [4.4](https://android.googlesource.com/kernel/common/+log/f0907aa15ed9f9c7541bb244ed3f52c376ced19c) | [4.9](https://android.googlesource.com/kernel/common/+log/5d15d2e00da4bcb0bcc5e6d27dc18fe1646214f1)
* Chromium: [4.4](https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/sandbox/mka/llvm/v4.4) | [4.9](https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/sandbox/mka/llvm/v4.9)

The branches in this repository will be dedicated to taking this patchset and enhancing it by fixing/hiding all of the warnings from Clang (from mainline, the Pixel 2, and my own knowledge).

All branches will build with `-Werror` and the following toolchains:
* [GCC 4.9.4](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/)
* [Clang 5.0](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+/master/clang-4053586/)
* [Clang 6.0](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+/master/clang-4691093/)
* [Clang 7.0](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+/master/clang-r328903/)

Branch information:

* [msm-3.18](https://github.com/nathanchance/android-kernel-clang/tree/msm-3.18) - based on the latest Oreo branch for the Snapdragon 820/821 ([kernel.lnx.3.18.r33-rel](https://source.codeaurora.org/quic/la/kernel/msm-3.18/log?h=kernel.lnx.3.18.r33-rel)). Uses `msm-perf_defconfig`.

* [msm-4.4](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.4) - based on the latest Oreo branch for the Snapdragon 835 ([kernel.lnx.4.4.r27-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.4/log?h=kernel.lnx.4.4.r27-rel)). Uses `msmcortex-perf_defconfig`.

* [msm-4.9](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.9) - based on the latest Oreo branch for the Snapdragon 845 ([kernel.lnx.4.9.r7-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.9/log?h=kernel.lnx.4.9.r7-rel)). Uses `sdm845-perf_defconfig`.

The general structure of these commits is as follows:

1. The core compilation support
2. Fixing Qualcomm specific drivers to compile with Clang
3. Fixing warnings that come from code in mainline
4. Fixing warnings that come from code outside of mainline

You should pick the commits that I have committed (nathanchance).

Additionally, there are fixes for:

* qcacld-2.0 available in [my Pixel XL kernel](https://github.com/nathanchance/marlin/commits/oreo-m4/drivers/staging/qcacld-2.0).
* qcacld-3.0 available in [my OnePlus 5 kernel](https://github.com/nathanchance/op5/commits/8.1.0-unified/drivers/staging/qcacld-3.0).

Every time there is a branch update upstream, the branch will be rebased, there is no stable history here! Ideally, I will not need to add any commits but I will do my best to keep everything in the same order.

**NOTE:** 3.18 Clang is not supported officially by an OEM. I've merely added it here as I decided to support it with [my Pixel XL kernel](https://github.com/nathanchance/marlin).


## Additional commits

In each stack of patches, I have included two sets of two commits:

* `Revert "scripts: gcc-wrapper: Use wrapper to check compiler warnings"` and `kernel: Add CC_WERROR config to turn warnings into errors`: The first commit removes CAF's crappy gcc-wrapper.py script, removing an unnecessary Python dependency, and the second commit adds the ability to compile with `-Werror` through a Kconfig option, `CONFIG_CC_WERROR`. I highly recommend you enable this after fixing any warnings that arise from OEM/CAF code!

* `UPSTREAM: scripts/mkcompile_h: Remove trailing spaces from compiler version` and `scripts: Support a custom compiler name`: On Oreo, Google tweaks the regex for parsing the kernel version to support no trailing space, which I think looks better, so the first commit removes the trailing space in GCC. The second commit allows you to easily remove the long URLs in the latest Google Clang versions by adding this bit of code to your compilation script (or run it in your shell before compiling):

  ```bash
  export KBUILD_COMPILER_STRING=$(<path_to_clang_folder/bin/clang --version | head -n 1 | perl -pe 's/\(http.*?\)//gs' | sed -e 's/  */ /g' -e 's/[[:space:]]*$//')
  ```


## Getting help

The preferred method for getting help is either opening an issue on this repository or joining the [Linux kernel newbies chat on Telegram](https://t.me/LinuxKernelNewbies).
