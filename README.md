# How to manually compile and build a tweak (on Linux)
After coming across Kritanta's [guide](https://github.com/KritantaDev/Guides/blob/master/TweakWithoutTheos.md) on how to build a tweak w/o [Theos](https://github.com/theos/theos) on Darwin systems, I figured that it may be helpful to have something similar catered for Linux systems.

**Note:** all credit goes to [Kritanta](https://twitter.com/arm64e) for her work on this guide. I've simply changed the parts that were Darwin-specific, revised some of the wording, and adjusted some stylistic elements.

---

Things that you'll need:
* build-essential, coreutils, curl, dpkg, git, libtinfo5, perl, tar, and unzip
  * *build-essential and libtinfo5 or the equivalents for your distro*
  * *Note: additional libraries may be required for some distros*
* Relevant [headers](https://github.com/theos/headers)
* Relevant [libs](https://github.com/theos/lib)
* [Logos](https://github.com/theos/logos)
* An [iOS Toolchain](https://github.com/sbingner/llvm-project)
* A [patched SDK](https://github.com/theos/sdks)
* A project containing, at a minimum, a Tweak file, a TweakName.plist, and a control file.

---

## 1. Preliminary Setup

Add the following environment variables to your `.profile`:
* `export MBS=~/my-build-system`
* `export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$MBS/toolchain/linux/iphone/lib`
* `export PATH=$PATH:$MBS/toolchain/linux/iphone/bin`

Restart your shell and `echo $MBS` to confirm that it worked.

**Note:** this setup process can be sped up *greatly* if you copy the following in to a shell script, give it execute permissions, and run it instead of copying/typing each command out individually.

    # Dependencies
    sudo apt-get install build-essential coreutils curl git libtinfo5 perl tar unzip

    # Making a centralized directory to hold all of our stuff
    mkdir -p $MBS/{include,lib,logos,toolchain,sdks}

    # Relevant Headers
    git clone https://github.com/theos/headers.git $MBS/include

    # Relevant Libs
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

`$MBS/logos/bin/logos.pl [input] > [output]`

For .x files, the input will be `Tweak.x` and the output will be `Tweak.x.m`.

For .xm files, the input will be `Tweak.xm` and the output will be `Tweak.xm.mm`.

For simplicity's sake, we're going to keep the naming scheme constant across all of the processed (and later linked) files.

---

## 3. Compile with Clang
### The command
`clang++ -target arm64-apple-darwin14-ld -target arm64e-apple-darwin14-ld -arch arm64 -arch arm64e -fobjc-arc -miphoneos-version-min=13.0 -isysroot $MBS/sdks/iPhoneOS14.4.sdk -isystem $MBS/include -Wall -O2 -c -o Tweak.xm.o Tweak.xm.mm`

### Explanation
`clang++` is a symlink to `clang` which is a symlink to `clang-10`, a [compiler](https://releases.llvm.org/10.0.0/tools/clang/docs/index.html#using-clang-as-a-compiler) in the [llvm project](https://github.com/llvm/llvm-project#readme) capable of compiling c++ code.

`-target <target triple>` specifies that we are compiling for Apple's arm64 and arm64e architectures. [From LLVM](https://releases.llvm.org/10.0.0/tools/clang/docs/CrossCompilation.html#target-triple): *"If you don’t specify the target, CPU names won’t match (since Clang assumes the host triple), and the compilation will go ahead, creating code for the host platform, which will break later on when assembling or linking."*

`-arch <archname>` specifies that we are compiling for the arm64 and arm64e architectures.

`-fobjc-arc` tells clang that we are using [ARC](https://clang.llvm.org/docs/AutomaticReferenceCounting.html#general) (Automated Reference Counting).

`-miphoneos-version-min=13.0` specifies that our target deployment version is iOS 13.0.

`-isysroot <directory>` Tells clang where our sdk root directory is.

`-isystem <direictory>` Tells clang to also search this path (not just the root directory above) for files to include.

`-Wall` tells clang to enable (almost) all of its warning modules. Note: you should keep this on. Sure, warnings are annoying, but unexpected behavior is even more so.

`-O2` (Letter 'O') tells the compiler to optimize at level 2. For a list and basic description of all available optimization levels, see [Free BSD's documentation](https://www.freebsd.org/cgi/man.cgi?query=clang++&sektion=1&manpath=FreeBSD+10.0-RELEASE) under "Code Generation Options."

`-c` Tells clang to compile, but not yet link the file.

`-o Tweak.xm.o` specifies the output file.

All other text in the command is evaluated as an input file (e.g., `Tweak.xm.mm`).

---

## 4. Link with Clang
### The command
`clang++ -target arm64-apple-darwin14-ld -target arm64e-apple-darwin14-ld -arch arm64 -arch arm64e -fobjc-arc -miphoneos-version-min=13.0 -isysroot $MBS/sdks/iPhoneOS14.4.sdk -isystem $MBS/include -Wall -O2 -fcolor-diagnostics -framework CoreFoundation -framework CoreGraphics -framework Foundation -framework UIKit -L$MBS/lib -lsubstrate -lobjc -lc++ -dynamiclib -ggdb -lSystem.B -o TweakName.dylib Tweak.xm.o`

#### New flags explained
`-fcolor-diagnostics` adds color to diagnostic information.

`-framework <Framework Name>` tells our linker we're going to be linking against a specific framework.

`-L<directory>` specifies a library search directory.

`-l<library name>` specifies a library we want to link against.

`-dynamiclib` tells clang that we are compiling and linking dynamically.

`-ggdb` tells our linker to produce debugging information for ggdb.

**Note:** If you're compiling for legacy architectures (armv7s and earlier), you'll want to add the following flags to the above command : `-Xlinker -segalign -Xlinker 4000`. For the reason as to why, see [here](https://github.com/theos/theos/blob/1e1c91ba1ff6dc63012fc3deed870787f4c402e5/makefiles/targets/_common/iphone.mk#L81).

---

## 5. Generate debug symbols
`dsymutil TweakName.dylib`

---

## 6. Fake code signature
`ldid -S TweakName.dylib`

---

## 7. Package the tweak
### Create the necessary structure
Dpkg is picky about package structures, so we need to set this up properly in order for our package to install as expected.

`mkdir -p .tmp/Library/MobileSubstrate/DynamicLibraries`

`mkdir .tmp/DEBIAN`

### Copy the files into their respective places
`cp TweakName.{dylib,plist} .tmp/Library/MobileSubstrate/DynamicLibraries/`

`cp control .tmp/DEBIAN/`

### Build the deb
`dpkg-deb -b -Zgzip -z9 .tmp .`

**Note:** Some packages will have to be built as root in order to retain the desired permissions/ownership. You will know if this is the case if something doesn't work as expected or if dpkg/apt throw errors upon the package's installation.

---

And, voilà, you have your package!

~ Kritanta & Lightmann
