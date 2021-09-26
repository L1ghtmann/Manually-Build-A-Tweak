# How to manually compile and build a tweak (on Linux)

After coming across Kritanta's [guide](https://github.com/KritantaDev/Guides/blob/master/TweakWithoutTheos.md) on how to build a tweak w/o [Theos](https://github.com/theos/theos) (or similar tools) on Darwin systems, I figured that it may be helpful to have something similar catered for Linux systems.

**Note:** huge thanks to [Kritanta](https://twitter.com/arm64e) for her initial work on this guide. In my rendition, I've changed the parts that were Darwin-specific, added relevant documentation links, elaborated various bits, revised some of the wording, and adjusted some stylistic elements.

---

Things that you'll need:
* build-essential, coreutils, curl, dpkg, git, libtinfo5, perl, tar, and unzip
  * *build-essential and libtinfo5 or the equivalents for your distro*
  * *Note: additional libraries may be required for some distros*
* Relevant [headers](https://github.com/theos/headers)
* Relevant [libraries](https://github.com/theos/lib)
* [Logos](https://github.com/theos/logos)
* An [iOS Toolchain](https://github.com/sbingner/llvm-project)
  * *To build your own toolchain, see [here](https://github.com/usrlightmann/building-an-ios-toolchain)*
* A [patched SDK](https://github.com/theos/sdks)
* A project containing, at a minimum, a Tweak file, a TweakName.plist, and a control file.

---

## 1. Preliminary Setup

Add the following environment variables to your `.profile` (bash) or `.zshenv` (zsh):
* `export MBS=~/my-build-system`
* `export LD_LIBRARY_PATH=$MBS/toolchain/linux/iphone/lib:$LD_LIBRARY_PATH`
* `export PATH=$MBS/toolchain/linux/iphone/bin:$PATH`

Restart your shell and `echo $MBS` to confirm that it worked.

**Note:** this setup process can be sped up *greatly* if you copy the following into a shell script, give it execute permissions, and run it instead of copying/typing each command out individually.

    # Dependencies
    sudo apt-get install build-essential coreutils curl git libtinfo5 perl tar unzip

    # Making a centralized directory to hold all of our stuff
    mkdir -p $MBS/{include,lib,logos,toolchain,sdks}

    # Relevant Headers
    git clone https://github.com/theos/headers.git $MBS/include

    # Relevant Libraries
    git clone https://github.com/theos/lib.git $MBS/lib

    # Logos
    git clone https://github.com/theos/logos.git $MBS/logos

    # Toolchain (x86|not Swift compatible)
    curl -LO https://github.com/sbingner/llvm-project/releases/latest/download/linux-ios-arm64e-clang-toolchain.tar.lzma
    TMP=$(mktemp -d)
    tar -xvf linux-ios-arm64e-clang-toolchain.tar.lzma -C $TMP
    mkdir -p $MBS/toolchain/linux/iphone
    mv $TMP/ios-arm64e-clang-toolchain/* $MBS/toolchain/linux/iphone/
    rm -r linux-ios-arm64e-clang-toolchain.tar.lzma $TMP

    # Patched sdks
    curl -LO https://github.com/theos/sdks/archive/master.zip
    TMP=$(mktemp -d)
    unzip master.zip -d $TMP
    mv $TMP/sdks-master/*.sdk $MBS/sdks
    rm -r master.zip $TMP

---

## The working directory

For this guide, the project directory will contain the following: `Tweak.xm`, `TweakName.plist`, and `control`.

This is kept simple for demonstration purposes, but just know that it can be scaled for larger projects with multiple tweak files, subprojects, etc.

---

## 2. Preprocess with Logos

[Logos](https://github.com/theos/logos) is a preprocessor created by the [Theos dev team](https://theos.dev/) to make writing tweaks less arduous. To learn more about what it is capable of/provides, see the iPhoneDevWiki's [documentation](https://iphonedev.wiki/index.php/Logos).

Using Logos, you can convert .xm and .x files to .m and .mm files respectively like so:

    $MBS/logos/bin/logos.pl [input] > [output]

For .x files, the input will be `Tweak.x` and the output will be `Tweak.x.m`.

For .xm files, the input will be `Tweak.xm` and the output will be `Tweak.xm.mm`.

For simplicity's sake, we're going to keep the naming scheme constant across all of the processed (and later linked) files.

---

## 3. Compile via Clang
### The command:

    clang++ -target arm64-apple-ios13.0 -target arm64e-apple-ios13.0 -arch arm64 -arch arm64e -fobjc-arc -isysroot $MBS/sdks/iPhoneOS14.4.sdk -isystem $MBS/include -Wall -O2 -c -o Tweak.xm.o Tweak.xm.mm

### Explanation:

`clang` is a [compiler front-end](https://clang.llvm.org/) in the [LLVM project](https://github.com/llvm/llvm-project#readme) capable of compiling C-based code (C, C++, Objective-C, etc). We're using `clang++` here because, by default, `clang` only links against the C standard library whereas `clang++` links against both the C and C++ standard libraries.

`-target <target triple>` specifies which platform(s) we want to build for and, in our case, the minimum iOS version our project is targeting. [From LLVM](https://releases.llvm.org/10.0.0/tools/clang/docs/CrossCompilation.html#target-triple): *"If you don’t specify the target, CPU names won’t match (since Clang assumes the host triple), and the compilation will go ahead, creating code for the host platform, which will break later on when assembling or linking."*

`-arch <architecture>` specifies which architecture(s) we want to build for.

`-fobjc-arc` tells clang to enable [ARC](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#general) (Automatic Reference Counting).

`-isysroot <directory>` specifies our target sdk as the system root directory.

`-isystem <directory>` specifies an additional system include search path.

`-Wall` tells clang to enable (almost) all of its warning modules.

`-O<level>` (Letter 'O') tells clang to optimize at the specified level. For a list and basic description of all available optimization levels, see [Free BSD's documentation](https://www.freebsd.org/cgi/man.cgi?query=clang++&sektion=1&manpath=FreeBSD+10.0-RELEASE) under "Code Generation Options."

`-c` tells clang to compile, but not link the file.

`-o <file>` tells clang to write the command's output to a file.

The rest of the command is evaluated as an input file (e.g., `Tweak.xm.mm`).

---

## 4. Link via Clang
### The command:

    clang++ -target arm64-apple-ios13.0 -target arm64e-apple-ios13.0 -arch arm64 -arch arm64e -fobjc-arc -isysroot $MBS/sdks/iPhoneOS14.4.sdk -isystem $MBS/include -Wall -O2 -fcolor-diagnostics -F$MBS/sdks/iPhoneOS14.4.sdk/System/Library/PrivateFrameworks -framework CoreFoundation -framework CoreGraphics -framework Foundation -framework UIKit -L$MBS/lib -lsubstrate -lobjc -lSystem.B -dynamiclib -ggdb -o TweakName.dylib Tweak.xm.o

### New flags explained:

`-fcolor-diagnostics` tells clang to add color to diagnostic information.

`-F<directory>` specifies an additional framework search path (required when linking against [private frameworks](https://github.com/SparkDev97/iOS14-Runtime-Headers/tree/master/PrivateFrameworks) included in a patched sdk).

`-framework <framework name>` specifies which framework(s) we want to link against.

`-L<directory>` specifies an additional library search path.

`-l<library name>` specifies which library/libraries we want to link against.

`-dynamiclib` tells the linker to link [dynamically](https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/OverviewOfDynamicLibraries.html) (as opposed to statically).

`-ggdb` tells the linker to produce debugging information for ggdb.

**Note:** by default, clang uses [GNU ld](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_mono/ld.html) as its linker. To select an [alternate linker](https://clang.llvm.org/docs/Toolchain.html#linker), add the following flag to the command above: `-fuse-ld=<linker name>`.

**Note 2:** if you're compiling for legacy architectures (armv7s and earlier), you'll want to add the following flags to the above command : `-Xlinker -segalign -Xlinker 4000`. For the reason as to why, see [here](https://github.com/theos/theos/blob/1e1c91ba1ff6dc63012fc3deed870787f4c402e5/makefiles/targets/_common/iphone.mk#L81).

---

## 5. Generate debug symbols

    dsymutil TweakName.dylib

---

## 6. Fake code signature

    ldid -S TweakName.dylib

---

## 7. Package the tweak
### Create the necessary structure:

Dpkg is picky about package structures, so we need to set this up properly in order for our package to install as expected.

    mkdir -p .tmp/DEBIAN .tmp/Library/MobileSubstrate/DynamicLibraries

### Copy the files into their respective places:

    cp TweakName.{dylib,plist} .tmp/Library/MobileSubstrate/DynamicLibraries/
    cp control .tmp/DEBIAN/

### Build the deb:

    dpkg-deb -b -Zgzip -z9 .tmp .

**Note:** some packages will have to be built as root in order to retain the desired permissions/ownership. You will know if this is the case if something doesn't work as expected or if dpkg/apt throw errors upon the package's installation.

**Note 2:** we are explicitly using gzip compression here to ensure compatibility with various dpkg versions (dpkg switched its default compression type to xz in v1.17.0).

---

And, voilà, you have your package!

~ Kritanta & Lightmann
