
## 尝试使用其他构建系统构建（TryBuild 模式）

xmake v2.3.1以上版本直接对接了其他第三方构建系统，即使其他项目中没有使用xmake.lua来维护，xmake也可以直接调用其他构建工具来完成编译。

那用户直接调用使用第三方构建工具来编译不就行了，为啥还要用xmake去调用呢？主要有以下好处：

1. 完全的行为一致，简化编译流程，不管用了哪个其他构建系统，都只需要执行xmake这个命令就可以编译，用户不再需要去研究其他工具的不同的编译流程
2. 完全对接`xmake config`的配置环境，复用xmake的平台探测和sdk环境检测，简化平台配置
3. 对接交叉编译环境，即使是用autotools维护的项目，也能通过xmake快速实现交叉编译

目前已支持的构建系统：

* autotools（已完全对接xmake的交叉编译环境）
* xcodebuild
* cmake（已完全对接xmake的交叉编译环境）
* make
* msbuild
* scons
* meson
* bazel
* ndkbuild
* ninja

### 自动探测构建系统并编译

例如，对于一个使用cmake维护的项目，直接在项目根目录执行xmake，就会自动触发探测机制，检测到CMakeLists.txt，然后提示用户是否需要使用cmake来继续完成编译。

```bash
$ xmake
note: CMakeLists.txt found, try building it (pass -y or --confirm=y/n/d to skip confirm)?
please input: y (y/n)
-- Symbol prefix:
-- Configuring done
-- Generating done
-- Build files have been written to: /Users/ruki/Downloads/libpng-1.6.35/build
[  7%] Built target png-fix-itxt
[ 21%] Built target genfiles
[ 81%] Built target png
[ 83%] Built target png_static
...
output to /Users/ruki/Downloads/libpng-1.6.35/build/artifacts
build ok!
```

### 无缝对接xmake命令

目前支持`xmake clean`, `xmake --rebuild`和`xmake config`等常用命令与第三方系统的无缝对接。

我们可以直接清理cmake维护项目的编译输出文件

```bash
$ xmake clean
$ xmake clean --all
```

如果带上`--all`执行清理，会清除autotools/cmake生成的所有文件，不仅仅只清理对象文件。

默认`xmake`对接的是增量构建行为，不过我们也可以强制快速重建：

```bash
$ xmake --rebuild
```

### 手动切换指定构建系统

如果一个项目下有多个构建系统同时在维护，比如libpng项目，自带autotools/cmake/makefile等构建系统维护，xmake默认优先探测使用了autotools，如果想要强制切换其他构建系统，可以执行：

```bash
$ xmake f --trybuild=[autotools|cmake|make|msbuild| ..]
$ xmake
```

另外，配置了`--trybuild=`参数手动指定了默认的构建系统，后续的build过程就不会额外提示用户选择了。

### 实现快速交叉编译

众所周知，cmake/autotools维护的项目虽然很多都支持交叉编译，但是交叉编译的配置过程很复杂，不同的工具链处理方式还有很多的差异，中途会踩到很多的坑。

即使跑通了一个工具链的交叉编译，如果切到另外一个工具链环境，可能又要折腾好久，而如果使用xmake，通常只需要两条简单的命令即可：

!> 目前cmake/autotools都已对接支持了xmake的交叉编译。

#### 交叉编译android平台

```bash
$ xmake f -p android --trybuild=autotools [--ndk=xxx]
$ xmake
```

!> 其中，--ndk参数配置是可选的，如果用户设置了ANDROID_NDK_HOME环境变量，或者ndk放置在~/Library/Android/sdk/ndk-bundle，xmake都能自动检测到。

是不是很简单？如果你觉得这没啥，那么可以对比下直接操作`./configure`去配置交叉编译，可以看下这篇文档对比下：[将NDK 与其他编译系统配合使用](https://developer.android.com/ndk/guides/other_build_systems#autoconf)

说白了，你大概得这样，还不一定一次就能搞定：

```bash
$ export TOOLCHAIN=$NDK/toolchains/llvm/prebuilt/$HOST_TAG
$ export AR=$TOOLCHAIN/bin/aarch64-linux-android-ar
$ export AS=$TOOLCHAIN/bin/aarch64-linux-android-as
$ export CC=$TOOLCHAIN/bin/aarch64-linux-android21-clang
$ export CXX=$TOOLCHAIN/bin/aarch64-linux-android21-clang++
$ export LD=$TOOLCHAIN/bin/aarch64-linux-android-ld
$ export RANLIB=$TOOLCHAIN/bin/aarch64-linux-android-ranlib
$ export STRIP=$TOOLCHAIN/bin/aarch64-linux-android-strip
$ ./configure --host aarch64-linux-android
$ make
```

如果是cmake呢，交叉编译也不省事，对于android平台，得这么配置。

```bash
$ cmake \
    -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=$ABI \
    -DANDROID_NATIVE_API_LEVEL=$MINSDKVERSION \
    $OTHER_ARGS
```

而对于 ios 平台，没找到简答的配置方式，就找到个第三方的 ios 工具链配置，很复杂：https://github.com/leetal/ios-cmake/blob/master/ios.toolchain.cmake

对于 mingw 又是另外一种方式，我又折腾了半天环境，很是折腾。

而对接了xmake后，不管是cmake还是autotools，交叉编译都是非常简单的，而且配置方式也完全一样，精简一致。

#### 交叉编译iphoneos平台

```bash
$ xmake f -p iphoneos --trybuild=[cmake|autotools]
$ xmake
```

#### 交叉编译mingw平台

```bash
$ xmake f -p mingw --trybuild=[cmake|autotools] [--mingw=xxx]
$ xmake
```

#### 使用其他交叉编译工具链

```bash
$ xmake f -p cross --trybuild=[cmake|autotools] --sdk=/xxxx
$ xmake
```

关于更多交叉编译的配置细节，请参考文档：[交叉编译](https://xmake.io/#/zh-cn/guide/configuration?id=%e4%ba%a4%e5%8f%89%e7%bc%96%e8%af%91)，除了多了一个`--trybuild=`参数，其他交叉编译配置参数都是完全通用的。

### 传递用户配置参数

我们可以通过`--tryconfigs=`来传递用户额外的配置参数到对应的第三方构建系统，比如：autotools会传递给`./configure`，cmake会传递给`cmake`命令。

```bash
$ xmake f --trybuild=autotools --tryconfigs="--enable-shared=no"
$ xmake
```

比如上述命令，传递`--enable-shared=no`给`./configure`，来禁用动态库编译。

另外，对于`--cflags`, `--includedirs`和`--ldflags`等参数，不需要通过`--tryconfigs`，通过`xmake config --cflags=`等内置参数就可透传过去。

### 编译其他构建系统过程示例

#### 通用编译方式

大多数情况下，每个构建系统对接后的编译方式都是一致的，除了`--trybuild=`配置参数除外。

```bash
$ xmake f --trybuild=[autotools|cmake|meson|ninja|bazel|make|msbuild|xcodebuild]
$ xmake
```

!> 我们还需要确保--trybuild指定的构建工具已经安装能够正常使用。

#### 构建Android jni程序

如果当前项目下存在`jni/Android.mk`，那么xmake可以直接调用ndk-build来构建jni库。

```bash
$ xmake f -p android --trybuild=ndkbuild [--ndk=]
$ xmake
```

如果觉得命令行编译jni比较麻烦，xmake也提供了相关的gradle集成插件[xmake-gradle](https://github.com/xmake-io/xmake-gradle)，可以无缝集成xmake进行jni库的编译集成，具体详情见：[使用xmake-gradle插件构建JNI程序](https://xmake.io/#/zh-cn/plugin/more_plugins?id=gradle%e6%8f%92%e4%bb%b6%ef%bc%88jni%ef%bc%89)

## 自动扫描源码生成xmake.lua

对于一份工程源码，可以不用编写makefile，也不用编写各种make相关的工程描述文件（例如：xmake.lua，makefile.am, cmakelist.txt等）

xmake就可以直接编译他们，这是如何做到的呢，简单来说下实现原理：

1. 首先扫描当前目录下，xmake所以支持的所有源代码文件
2. 分析代码，检测哪些代码拥有main入口函数
3. 所有没有main入口的代码编译成静态库
4. 带有main入口的代码，编译成可执行程序，同时链接其他静态库程序

这种代码扫描和智能编译，非常简单，目前xmake还不支持多级目录扫描，只对单级目录的代码进行扫描编译。。

### 使用场景

1. 临时快速编译和运行一些零散的测试代码
2. 尝试对其他开源库做移植编译
3. 快速基于现有代码创建新xmake工程

### 如何使用

直接在带有源码的目录（没有xmake.lua）下执行xmake，然后根据提示操作：

```bash
$ xmake
note: xmake.lua not found, try generating it (pass -y or --confirm=y/n/d to skip confirm)?
please input: n (y/n)
y
```

另外, 当存在其他构建系统标识性文件的时候(比如 CMakeLists.txt), 不会触发自动生成 xmake.lua 的流程, 而是首先触发 [自动探测构建系统并编译](#自动探测构建系统并编译), 如果要强制触发自动扫描生成 xmake.lua 的流程, 可运行: 

```bash
$ xmake f -y
```

### 开源代码的移植和编译

虽然这种方式，并不是非常智能，限制也不少，但是对于想临时写些代码进行编译运行，或者临时想交叉编译一些简单的开源库代码

这种方式已经足够使用了，下面看下一个实际的例子：

我下载了一份zlib-1.2.10的源码，想要编译它，只需要进入zlib的源码目录执行：

```bash
$ cd zlib-1.2.10
$ xmake
note: xmake.lua not found, try generating it (pass -y or --confirm=y/n/d to skip confirm)?
please input: n (y/n)
y
```

就行了，输入y确认后，输出结果如下：

```
target(zlib-1.2): static
    [+]: ./adler32.c
    [+]: ./compress.c
    [+]: ./crc32.c
    [+]: ./deflate.c
    [+]: ./gzclose.c
    [+]: ./gzlib.c
    [+]: ./gzread.c
    [+]: ./gzwrite.c
    [+]: ./infback.c
    [+]: ./inffast.c
    [+]: ./inflate.c
    [+]: ./inftrees.c
    [+]: ./trees.c
    [+]: ./uncompr.c
    [+]: ./zutil.c
xmake.lua generated, scan ok!👌
checking for the architecture ... x86_64
checking for the Xcode SDK version for macosx ... 10.12
checking for the target minimal version ... 10.12
checking for the c compiler (cc) ... xcrun -sdk macosx clang
checking for the c++ compiler (cxx) ... xcrun -sdk macosx clang
checking for the objc compiler (mm) ... xcrun -sdk macosx clang
checking for the objc++ compiler (mxx) ... xcrun -sdk macosx clang++
checking for the swift compiler (sc) ... xcrun -sdk macosx swiftc
checking for the assember (as) ... xcrun -sdk macosx clang
checking for the linker (ld) ... xcrun -sdk macosx clang++
checking for the static library archiver (ar) ... xcrun -sdk macosx ar
checking for the static library extractor (ex) ... xcrun -sdk macosx ar
checking for the shared library linker (sh) ... xcrun -sdk macosx clang++
checking for the debugger (dd) ... xcrun -sdk macosx lldb
checking for the golang compiler (go) ... go
configure
{
    ex = "xcrun -sdk macosx ar"
,   sh = "xcrun -sdk macosx clang++"
,   host = "macosx"
,   ar = "xcrun -sdk macosx ar"
,   buildir = "build"
,   as = "xcrun -sdk macosx clang"
,   plat = "macosx"
,   xcode_dir = "/Applications/Xcode.app"
,   arch = "x86_64"
,   mxx = "xcrun -sdk macosx clang++"
,   go = "go"
,   target_minver = "10.12"
,   ccache = "ccache"
,   mode = "release"
,   clean = true
,   cxx = "xcrun -sdk macosx clang"
,   cc = "xcrun -sdk macosx clang"
,   dd = "xcrun -sdk macosx lldb"
,   kind = "static"
,   ld = "xcrun -sdk macosx clang++"
,   xcode_sdkver = "10.12"
,   sc = "xcrun -sdk macosx swiftc"
,   mm = "xcrun -sdk macosx clang"
}
configure ok!
clean ok!
[00%]: ccache compiling.release ./adler32.c
[06%]: ccache compiling.release ./compress.c
[13%]: ccache compiling.release ./crc32.c
[20%]: ccache compiling.release ./deflate.c
[26%]: ccache compiling.release ./gzclose.c
[33%]: ccache compiling.release ./gzlib.c
[40%]: ccache compiling.release ./gzread.c
[46%]: ccache compiling.release ./gzwrite.c
[53%]: ccache compiling.release ./infback.c
[60%]: ccache compiling.release ./inffast.c
[66%]: ccache compiling.release ./inflate.c
[73%]: ccache compiling.release ./inftrees.c
[80%]: ccache compiling.release ./trees.c
[86%]: ccache compiling.release ./uncompr.c
[93%]: ccache compiling.release ./zutil.c
[100%]: archiving.release libzlib-1.2.a
build ok!👌
```

通过输出结果，可以看到，xmake会去检测扫描当前目录下的所有.c代码，发现没有main入口，应该是静态库程序，因此执行xmake后，就直接编译成静态库libzlib-1.2.a了

连xmake.lua都没有编写，其实xmake在扫描完成后，会去自动在当前目录下生成一份xmake.lua，下次编译就不需要重新扫描检测了。

自动生成的xmake.lua内容如下：

```lua
-- define target
target("zlib-1.2")

    -- set kind
    set_kind("static")

    -- add files
    add_files("./adler32.c")
    add_files("./compress.c")
    add_files("./crc32.c")
    add_files("./deflate.c")
    add_files("./gzclose.c")
    add_files("./gzlib.c")
    add_files("./gzread.c")
    add_files("./gzwrite.c")
    add_files("./infback.c")
    add_files("./inffast.c")
    add_files("./inflate.c")
    add_files("./inftrees.c")
    add_files("./trees.c")
    add_files("./uncompr.c")
    add_files("./zutil.c")
```

也许你会说，像这种开源库，直接`configure; make`不就好了吗，他们自己也有提供makefile来直接编译的，的确是这样，我这里只是举个例子而已。。

当然，很多开源库在交叉编译的时候，通过自带的`configure`，处理起来还是很繁琐的，用xmake进行交叉编译会更方便些。。

### 即时地代码编写和编译运行

xmake的这个扫描代码编译特性，主要的目的：还是为了让我们在临时想写些测试代码的时候，不用考虑太多东西，直接上手敲代码，然后快速执行`xmake run` 来调试验证结果。。

例如：

我想写了个简单的main.c的测试程序，打印`hello world!`，如果要写makefile或者直接通过gcc命令来，就很繁琐了，你需要：

```bash
gcc ./main.c -o demo
./demo
```

最快速的方式，也需要执行两行命令，而如果用xmake，只需要执行：

```bash
xmake run
```

就行了，它会自动检测到代码后，自动生成对应的xmake.lua，自动编译，自动运行，然后输出：

```
hello world!
```

如果你有十几个代码文件，用手敲gcc的方式，或者写makefile的方式，这个差距就更明显了，用xmake还是只需要一行命令：

```bash
xmake run
```

### 多语言支持

这种代码检测和即时编译，是支持多语言的，不仅支持c/c++，还支持objc/swift，后期还会支持golang（正在开发中）

例如我下载了一份fmdb的ios开源框架代码：

```
.
├── FMDB.h
├── FMDatabase.h
├── FMDatabase.m
├── FMDatabaseAdditions.h
├── FMDatabaseAdditions.m
├── FMDatabasePool.h
├── FMDatabasePool.m
├── FMDatabaseQueue.h
├── FMDatabaseQueue.m
├── FMResultSet.h
└── FMResultSet.m
```

想要把它编译成ios的静态库，但是又不想写xmake.lua，或者makefile，那么只需要使用xmake的这个新特性，直接执行：

```bash
$ xmake f -p iphoneos; xmake
```

就行了，输出结果如下：

```
xmake.lua not found, scanning files ..
target(FMDB): static
    [+]: ./FMDatabase.m
    [+]: ./FMDatabaseAdditions.m
    [+]: ./FMDatabasePool.m
    [+]: ./FMDatabaseQueue.m
    [+]: ./FMResultSet.m
xmake.lua generated, scan ok!👌
checking for the architecture ... armv7
checking for the Xcode SDK version for iphoneos ... 10.1
checking for the target minimal version ... 10.1
checking for the c compiler (cc) ... xcrun -sdk iphoneos clang
checking for the c++ compiler (cxx) ... xcrun -sdk iphoneos clang
checking for the objc compiler (mm) ... xcrun -sdk iphoneos clang
checking for the objc++ compiler (mxx) ... xcrun -sdk iphoneos clang++
checking for the assember (as) ... gas-preprocessor.pl xcrun -sdk iphoneos clang
checking for the linker (ld) ... xcrun -sdk iphoneos clang++
checking for the static library archiver (ar) ... xcrun -sdk iphoneos ar
checking for the static library extractor (ex) ... xcrun -sdk iphoneos ar
checking for the shared library linker (sh) ... xcrun -sdk iphoneos clang++
checking for the swift compiler (sc) ... xcrun -sdk iphoneos swiftc
configure
{
    ex = "xcrun -sdk iphoneos ar"
,   ccache = "ccache"
,   host = "macosx"
,   ar = "xcrun -sdk iphoneos ar"
,   buildir = "build"
,   as = "/usr/local/share/xmake/tools/utils/gas-preprocessor.pl xcrun -sdk iphoneos clang"
,   arch = "armv7"
,   mxx = "xcrun -sdk iphoneos clang++"
,   cxx = "xcrun -sdk iphoneos clang"
,   target_minver = "10.1"
,   xcode_dir = "/Applications/Xcode.app"
,   clean = true
,   sh = "xcrun -sdk iphoneos clang++"
,   cc = "xcrun -sdk iphoneos clang"
,   ld = "xcrun -sdk iphoneos clang++"
,   mode = "release"
,   kind = "static"
,   plat = "iphoneos"
,   xcode_sdkver = "10.1"
,   sc = "xcrun -sdk iphoneos swiftc"
,   mm = "xcrun -sdk iphoneos clang"
}
configure ok!
clean ok!
[00%]: ccache compiling.release ./FMDatabase.m
[20%]: ccache compiling.release ./FMDatabaseAdditions.m
[40%]: ccache compiling.release ./FMDatabasePool.m
[60%]: ccache compiling.release ./FMDatabaseQueue.m
[80%]: ccache compiling.release ./FMResultSet.m
[100%]: archiving.release libFMDB.a
build ok!👌
```

### 同时编译多个可执行文件

输出结果的开头部分，就是对代码的分析结果，虽然目前只支持单级目录结构的代码扫描，但是还是可以同时支持检测和编译多个可执行文件的

我们以libjpeg的开源库为例：

我们进入jpeg-6b目录后，执行：

```bash
$ xmake
```

输出如下：

```
xmake.lua not found, scanning files ..
target(jpeg-6b): static
    [+]: ./cdjpeg.c
    [+]: ./example.c
    [+]: ./jcapimin.c
    [+]: ./jcapistd.c
    [+]: ./jccoefct.c
    [+]: ./jccolor.c
    [+]: ./jcdctmgr.c
    [+]: ./jchuff.c
    [+]: ./jcinit.c
    [+]: ./jcmainct.c
    [+]: ./jcmarker.c
    [+]: ./jcmaster.c
    [+]: ./jcomapi.c
    [+]: ./jcparam.c
    [+]: ./jcphuff.c
    [+]: ./jcprepct.c
    [+]: ./jcsample.c
    [+]: ./jctrans.c
    [+]: ./jdapimin.c
    [+]: ./jdapistd.c
    [+]: ./jdatadst.c
    [+]: ./jdatasrc.c
    [+]: ./jdcoefct.c
    [+]: ./jdcolor.c
    [+]: ./jddctmgr.c
    [+]: ./jdhuff.c
    [+]: ./jdinput.c
    [+]: ./jdmainct.c
    [+]: ./jdmarker.c
    [+]: ./jdmaster.c
    [+]: ./jdmerge.c
    [+]: ./jdphuff.c
    [+]: ./jdpostct.c
    [+]: ./jdsample.c
    [+]: ./jdtrans.c
    [+]: ./jerror.c
    [+]: ./jfdctflt.c
    [+]: ./jfdctfst.c
    [+]: ./jfdctint.c
    [+]: ./jidctflt.c
    [+]: ./jidctfst.c
    [+]: ./jidctint.c
    [+]: ./jidctred.c
    [+]: ./jmemansi.c
    [+]: ./jmemmgr.c
    [+]: ./jmemname.c
    [+]: ./jmemnobs.c
    [+]: ./jquant1.c
    [+]: ./jquant2.c
    [+]: ./jutils.c
    [+]: ./rdbmp.c
    [+]: ./rdcolmap.c
    [+]: ./rdgif.c
    [+]: ./rdppm.c
    [+]: ./rdrle.c
    [+]: ./rdswitch.c
    [+]: ./rdtarga.c
    [+]: ./transupp.c
    [+]: ./wrbmp.c
    [+]: ./wrgif.c
    [+]: ./wrppm.c
    [+]: ./wrrle.c
    [+]: ./wrtarga.c
target(ansi2knr): binary
    [+]: ./ansi2knr.c
target(cjpeg): binary
    [+]: ./cjpeg.c
target(ckconfig): binary
    [+]: ./ckconfig.c
target(djpeg): binary
    [+]: ./djpeg.c
target(jpegtran): binary
    [+]: ./jpegtran.c
target(rdjpgcom): binary
    [+]: ./rdjpgcom.c
target(wrjpgcom): binary
    [+]: ./wrjpgcom.c
xmake.lua generated, scan ok!👌
checking for the architecture ... x86_64
checking for the Xcode SDK version for macosx ... 10.12
checking for the target minimal version ... 10.12
checking for the c compiler (cc) ... xcrun -sdk macosx clang
checking for the c++ compiler (cxx) ... xcrun -sdk macosx clang
checking for the objc compiler (mm) ... xcrun -sdk macosx clang
checking for the objc++ compiler (mxx) ... xcrun -sdk macosx clang++
checking for the swift compiler (sc) ... xcrun -sdk macosx swiftc
checking for the assember (as) ... xcrun -sdk macosx clang
checking for the linker (ld) ... xcrun -sdk macosx clang++
checking for the static library archiver (ar) ... xcrun -sdk macosx ar
checking for the static library extractor (ex) ... xcrun -sdk macosx ar
checking for the shared library linker (sh) ... xcrun -sdk macosx clang++
checking for the debugger (dd) ... xcrun -sdk macosx lldb
checking for the golang compiler (go) ... go
configure
{
    ex = "xcrun -sdk macosx ar"
,   sh = "xcrun -sdk macosx clang++"
,   host = "macosx"
,   ar = "xcrun -sdk macosx ar"
,   buildir = "build"
,   as = "xcrun -sdk macosx clang"
,   plat = "macosx"
,   xcode_dir = "/Applications/Xcode.app"
,   arch = "x86_64"
,   mxx = "xcrun -sdk macosx clang++"
,   go = "go"
,   target_minver = "10.12"
,   ccache = "ccache"
,   mode = "release"
,   clean = true
,   cxx = "xcrun -sdk macosx clang"
,   cc = "xcrun -sdk macosx clang"
,   dd = "xcrun -sdk macosx lldb"
,   kind = "static"
,   ld = "xcrun -sdk macosx clang++"
,   xcode_sdkver = "10.12"
,   sc = "xcrun -sdk macosx swiftc"
,   mm = "xcrun -sdk macosx clang"
}
configure ok!
clean ok!
[00%]: ccache compiling.release ./cdjpeg.c
[00%]: ccache compiling.release ./example.c
[00%]: ccache compiling.release ./jcapimin.c
[00%]: ccache compiling.release ./jcapistd.c
[00%]: ccache compiling.release ./jccoefct.c
[00%]: ccache compiling.release ./jccolor.c
[01%]: ccache compiling.release ./jcdctmgr.c
[01%]: ccache compiling.release ./jchuff.c
[01%]: ccache compiling.release ./jcinit.c
[01%]: ccache compiling.release ./jcmainct.c
[01%]: ccache compiling.release ./jcmarker.c
[02%]: ccache compiling.release ./jcmaster.c
[02%]: ccache compiling.release ./jcomapi.c
[02%]: ccache compiling.release ./jcparam.c
[02%]: ccache compiling.release ./jcphuff.c
[02%]: ccache compiling.release ./jcprepct.c
[03%]: ccache compiling.release ./jcsample.c
[03%]: ccache compiling.release ./jctrans.c
[03%]: ccache compiling.release ./jdapimin.c
[03%]: ccache compiling.release ./jdapistd.c
[03%]: ccache compiling.release ./jdatadst.c
[04%]: ccache compiling.release ./jdatasrc.c
[04%]: ccache compiling.release ./jdcoefct.c
[04%]: ccache compiling.release ./jdcolor.c
[04%]: ccache compiling.release ./jddctmgr.c
[04%]: ccache compiling.release ./jdhuff.c
[05%]: ccache compiling.release ./jdinput.c
[05%]: ccache compiling.release ./jdmainct.c
[05%]: ccache compiling.release ./jdmarker.c
[05%]: ccache compiling.release ./jdmaster.c
[05%]: ccache compiling.release ./jdmerge.c
[06%]: ccache compiling.release ./jdphuff.c
[06%]: ccache compiling.release ./jdpostct.c
[06%]: ccache compiling.release ./jdsample.c
[06%]: ccache compiling.release ./jdtrans.c
[06%]: ccache compiling.release ./jerror.c
[07%]: ccache compiling.release ./jfdctflt.c
[07%]: ccache compiling.release ./jfdctfst.c
[07%]: ccache compiling.release ./jfdctint.c
[07%]: ccache compiling.release ./jidctflt.c
[07%]: ccache compiling.release ./jidctfst.c
[08%]: ccache compiling.release ./jidctint.c
[08%]: ccache compiling.release ./jidctred.c
[08%]: ccache compiling.release ./jmemansi.c
[08%]: ccache compiling.release ./jmemmgr.c
[08%]: ccache compiling.release ./jmemname.c
[09%]: ccache compiling.release ./jmemnobs.c
[09%]: ccache compiling.release ./jquant1.c
[09%]: ccache compiling.release ./jquant2.c
[09%]: ccache compiling.release ./jutils.c
[09%]: ccache compiling.release ./rdbmp.c
[10%]: ccache compiling.release ./rdcolmap.c
[10%]: ccache compiling.release ./rdgif.c
[10%]: ccache compiling.release ./rdppm.c
[10%]: ccache compiling.release ./rdrle.c
[10%]: ccache compiling.release ./rdswitch.c
[11%]: ccache compiling.release ./rdtarga.c
[11%]: ccache compiling.release ./transupp.c
[11%]: ccache compiling.release ./wrbmp.c
[11%]: ccache compiling.release ./wrgif.c
[11%]: ccache compiling.release ./wrppm.c
[12%]: ccache compiling.release ./wrrle.c
[12%]: ccache compiling.release ./wrtarga.c
[12%]: archiving.release libjpeg-6b.a
[12%]: ccache compiling.release ./wrjpgcom.c
[25%]: linking.release wrjpgcom
[25%]: ccache compiling.release ./ansi2knr.c
[37%]: linking.release ansi2knr
[37%]: ccache compiling.release ./jpegtran.c
[50%]: linking.release jpegtran
[50%]: ccache compiling.release ./djpeg.c
[62%]: linking.release djpeg
[62%]: ccache compiling.release ./ckconfig.c
[75%]: linking.release ckconfig
[75%]: ccache compiling.release ./rdjpgcom.c
[87%]: linking.release rdjpgcom
[87%]: ccache compiling.release ./cjpeg.c
[100%]: linking.release cjpeg
build ok!👌
```

可以看到，处理静态库，xmake还分析出了很多可执行的测试程序，剩下的代码统一编译成一个 libjpeg.a 的静态库，供哪些测试程序链接使用。。

```
target(ansi2knr): binary
    [+]: ./ansi2knr.c
target(cjpeg): binary
    [+]: ./cjpeg.c
target(ckconfig): binary
    [+]: ./ckconfig.c
target(djpeg): binary
    [+]: ./djpeg.c
target(jpegtran): binary
    [+]: ./jpegtran.c
target(rdjpgcom): binary
    [+]: ./rdjpgcom.c
target(wrjpgcom): binary
    [+]: ./wrjpgcom.c
```

### 遇到的一些问题和限制

当前xmake的这种自动分析检测还不是非常智能，对于：

1. 需要特殊的编译选项
2. 需要依赖其他目录的头文件搜索
3. 需要分条件编译不同源文件
4. 同目录需要生成多个静态库
5. 需要多级目录支持的源码库

以上这些情况，xmake暂时还没发自动化的智能处理，其中限制1，2还是可以解决的，通过半手动的方式，例如：

```bash
$ xmake f --cxflags="" --ldflags="" --includedirs="" --linkdirs=""; xmake
```

在自动检测编译的时候，手动配置这个源码工程需要的特殊编译选项，就可以直接通过编译了

而限制3，暂时只能通过删源代码来解决了，就像刚才编译jpeg的代码，其实它的目录下面同时存在了：

```
jmemdos.c
jmemmac.c
jmemansi.c
```

其中两个是没法编译过的，需要删掉后才行。。

## Unity Build 编译加速

我们知道，C++ 代码编译速度通常很慢，因为每个代码文件都需要解析引入的头文件。

而通过 Unity Build，我们通过将多个 cpp 文件组合成一个来加速项目的编译，其主要好处是减少了解析和编译包含在多个源文件中的头文件内容的重复工作，头文件的内容通常占预处理后源文件中的大部分代码。

Unity 构建还通过减少编译链创建和处理的目标文件的数量来减轻由于拥有大量小源文件而导致的开销，并允许跨形成统一构建任务的文件进行过程间分析和优化（类似于效果链接时优化）。

它可以极大提升 C/C++ 代码的编译速度，通常会有 30% 的速度提升，不过根据项目的复杂程度不同，其带来的效益还是要根据自身项目情况而定。

xmake 在 v2.5.9 版本中，也已经支持了这种构建模式。相关 issues 见 [#1019](https://github.com/xmake-io/xmake/issues/1019)。

### 如何启用？

我们提供了两个内置规则，分别处理对 C 和 C++ 代码的 Unity Build。

```lua
add_rules("c.unity_build")
add_rules("c++.unity_build")
```

### Batch 模式

默认情况下，只要设置上述规则，就会启用 Batch 模式的 Unity Build，也就是 xmake 自动根据项目代码文件，自动组织合并。

```lua
target("test")
    set_kind("binary")
    add_includedirs("src")
    add_rules("c++.unity_build", {batchsize = 2})
    add_files("src/*.c", "src/*.cpp")
```

我们可以额外通过设置 `{batchsize = 2}` 参数到规则，来指定每个合并 Batch 的大小数量，这里也就是每两个 C++ 文件自动合并编译。

编译效果大概如下：

```console
$ xmake -r
[ 11%]: ccache compiling.release build/.gens/test/unity_build/unity_642A245F.cpp
[ 11%]: ccache compiling.release build/.gens/test/unity_build/unity_bar.cpp
[ 11%]: ccache compiling.release build/.gens/test/unity_build/unity_73161A20.cpp
[ 11%]: ccache compiling.release build/.gens/test/unity_build/unity_F905F036.cpp
[ 11%]: ccache compiling.release build/.gens/test/unity_build/unity_foo.cpp
[ 11%]: ccache compiling.release build/.gens/test/unity_build/main.c
[ 77%]: linking.release test
[100%]: build ok
```

由于我们仅仅启用了 C++ 的 Unity Build，所以 C 代码还是正常挨个编译。另外在 Unity Build 模式下，我们还是可以做到尽可能的并行编译加速，互不冲突。

如果没有设置 `batchsize` 参数，那么默认会吧所有文件合并到一个文件中进行编译。

### Group 模式

如果上面的 Batch 模式自动合并效果不理想，我们也可以使用自定义分组，来手动配置哪些文件合并到一起参与编译，这使得用户更加地灵活可控。

```lua
target("test")
    set_kind("binary")
    add_rules("c++.unity_build", {batchsize = 0}) -- disable batch mode
    add_files("src/*.c", "src/*.cpp")
    add_files("src/foo/*.c", {unity_group = "foo"})
    add_files("src/bar/*.c", {unity_group = "bar"})
```

我们使用 `{unity_group = "foo"}` 来指定每个分组的名字，以及包含了哪些文件，每个分组的文件都会单独被合并到一个代码文件中去。

另外，`batchsize = 0` 也强行禁用了 Batch 模式，也就是说，没有设置 unity_group 分组的代码文件，我们还是会单独编译它们，也不会自动开启自动合并。

### Batch 和 Group 混合模式

我们只要把上面的 `batchsize = 0` 改成非 0 值，就可以让分组模式下，剩余的代码文件继续开启 Batch 模式自动合并编译。

```lua
target("test")
    set_kind("binary")
    add_includedirs("src")
    add_rules("c++.unity_build", {batchsize = 2})
    add_files("src/*.c", "src/*.cpp")
    add_files("src/foo/*.c", {unity_group = "foo"})
    add_files("src/bar/*.c", {unity_group = "bar"})
```

### 忽略指定文件

如果是 Batch 模式下，由于是自动合并操作，所以默认会对所有文件执行合并，但如果有些代码文件我们不想让它参与合并，那么我们也可以通过 `{unity_ignored = true}` 去忽略它们。

```lua
target("test")
    set_kind("binary")
    add_includedirs("src")
    add_rules("c++.unity_build", {batchsize = 2})
    add_files("src/*.c", "src/*.cpp")
    add_files("src/test/*.c", {unity_ignored = true}) -- ignore these files
```

### Unique ID

尽管 Unity Build 带啦的收益不错，但是我们还是会遇到一些意外的情况，比如我们的两个代码文件里面，全局命名空间下，都存在相同名字的全局变量和函数。

那么，合并编译就会带来编译冲突问题，编译器通常会报全局变量重定义错误。

为了解决这个问题，我们需要用户代码上做一些修改，然后配合构建工具来解决。

比如，我们的 foo.cpp 和 bar.cpp 都有全局变量 i。

foo.cpp

```c
namespace {
    int i = 42;
}

int foo()
{
    return i;
}
```

bar.cpp

```c
namespace {
    int i = 42;
}

int bar()
{
    return i;
}
```

那么，我们合并编译就会冲突，我们可以引入一个 Unique ID 来隔离全局的匿名空间。


foo.cpp

```c
namespace MY_UNITY_ID {
    int i = 42;
}

int foo()
{
    return MY_UNITY_ID::i;
}
```

bar.cpp

```c
namespace MY_UNITY_ID {
    int i = 42;
}

int bar()
{
    return MY_UNITY_ID::i;
}
```

接下来，我们还需要保证代码合并后， `MY_UNITY_ID` 在 foo 和 bar 中的定义完全不同，可以按文件名算一个唯一 ID 值出来，互不冲突，也就是实现下面的合并效果：

```c
#define MY_UNITY_ID <hash(foo.cpp)>
#include "foo.c"
#undef MY_UNITY_ID
#define MY_UNITY_ID <hash(bar.cpp)>
#include "bar.c"
#undef MY_UNITY_ID
```

这看上去似乎很麻烦，但是用户不需要关心这些，xmake 会在合并时候自动处理它们，用户只需要指定这个 Unique ID 的名字就行了，例如下面这样：


```lua
target("test")
    set_kind("binary")
    add_includedirs("src")
    add_rules("c++.unity_build", {batchsize = 2, uniqueid = "MY_UNITY_ID"})
    add_files("src/*.c", "src/*.cpp")
```

处理全局变量，还有全局的重名宏定义，函数什么的，都可以采用这种方式来避免冲突。

## 远程编译

2.6.5 版本提供了远程编译支持，我们可以通过它可以远程服务器上编译代码，远程运行和调试。

服务器可以部署在 Linux/MacOS/Windows 上，实现跨平台编译，例如：在 Linux 上编译运行 Windows 程序，在 Windows 上编译运行 macOS/Linux 程序。

相比 ssh 远程登入编译，它更加的稳定，使用更加流畅，不会因为网络不稳定导致 ssh 终端输入卡顿，也可以实现本地快速编辑代码文件。

甚至我们可以在 vs/sublime/vscode/idea 等编辑器和IDE 中无缝实现远程编译，而不需要依赖 IDE 本身对远程编译的支持力度。

### 开启服务

```console
$ xmake service
<remote_build_server>: listening 0.0.0.0:9091 ..
```

我们也可以开启服务的同时，回显详细日志信息。

```console
$ xmake service -vD
<remote_build_server>: listening 0.0.0.0:9091 ..
```

### 以 Daemon 模式开启服务

```console
$ xmake service --start
$ xmake service --restart
$ xmake service --stop
```

### 配置服务端

我们首先，运行 `xmake service` 命令，它会自动生成一个默认的 `server.conf` 配置文件，存储到 `~/.xmake/service/server.conf`。

!> 2.6.5 版本，配置地址在 `~/.xmake/service.conf`，后续版本做了大量改进，分离了配置文件，如果用的是 2.6.6 以上版本，请使用新的配置文件。

然后，我们编辑它，修复服务器的监听端口（可选）。

```bash
$ cat ~/.xmake/service/server.conf
{
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    remote_build = {
        listen = "0.0.0.0:9691",
        workdir = "/Users/ruki/.xmake/service/server/remote_build"
    },
    tokens = {
        "e438d816c95958667747c318f1532c0f"
    }
}
```

### 配置客户端

客户端配置文件在 `~/.xmake/service/client.conf`，我们可以在里面配置客户端需要连接的服务器地址。

!> 2.6.5 版本，配置地址在 `~/.xmake/service.conf`，后续版本做了大量改进，分离了配置文件，如果用的是 2.6.6 以上版本，请使用新的配置文件。

```console
$ cat ~/.xmake/service/client.conf
{
    remote_build = {
        connect = "127.0.0.1:9691",
        token = "e438d816c95958667747c318f1532c0f"
    }
}
```

### 用户认证和授权

!> 2.6.6 以上版本才支持用户认证，2.6.5 版本只能匿名连接。

在实际连接之前，我们简单介绍下 xmake 提供的服务目前提供的几种认证机制。

1. Token 认证
2. 密码认证
3. 可信主机验证

#### Token 认证

这也是我们默认推荐的方式，更加安全，配置和连接也更加方便，每次连接也不用输入密码。

我们在执行 `xmake service` 命令时候，默认就会生成一个服务端和客户端的配置文件，并且自动生成一个默认的 token，因此本地直连是不需要任何配置的。

##### 服务端认证配置

服务端可以配置多个 token 用于对不同用户主机进行授权连接，当然也可以共用一个 token。

```bash
$ cat ~/.xmake/service/server.conf
{
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    remote_build = {
        listen = "0.0.0.0:9691",
        workdir = "/Users/ruki/.xmake/service/server/remote_build"
    },
    tokens = {
        "e438d816c95958667747c318f1532c0f"
    }
}
```

##### 客户端认证配置

客户端只需要添加服务器上的 token 到对应的客户端配置中即可。

```bash
$ cat ~/.xmake/service/client.conf
{
    remote_build = {
        connect = "127.0.0.1:9691",
        token = "e438d816c95958667747c318f1532c0f"
    }
}
```

##### 手动生成新 token

我们也可以执行下面的命令，手动生成一个新的 token，自己添加到服务器配置中。

```bash
$ xmake service --gen-token
New token a7b9fc2d3bfca1472aabc38bb5f5d612 is generated!
```

#### 密码认证

我们也提供密码认证的授权模式，相比 token 认证，它需要用户每次连接的时候，输入密码，验证通过后，才能连接上。

##### 服务端认证配置

密码认证，我们不需要手动配置 token，只需要执行下面的命令，添加用户就行了，添加过程中，会提示用户输入密码。

```bash
$ xmake service --add-user=ruki
Please input user ruki password:
123456
Add user ruki ok!
```

然后，xmake 就会通过用户名，密码生成一个新的 token 添加到服务器配置的 token 列表中去。

```bash
$ cat ~/.xmake/service/server.conf
{
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    remote_build = {
        listen = "0.0.0.0:9691",
        workdir = "/Users/ruki/.xmake/service/server/remote_build"
    },
    tokens = {
        "e438d816c95958667747c318f1532c0f",
        "7889e25402413e93fd37395a636bf942"
    }
}
```

当然，我们也可以删除指定的用户和密码。

```bash
$xmake service --rm-user=ruki
Please input user ruki password:
123456
Remove user ruki ok!
```

##### 客户端认证配置

对于客户端，我们不再需要设置服务器的 token 了，只需要在连接配置中，追加需要连接的用户名即可开启密码认证，格式：`user@address:port`

```bash
$ cat ~/.xmake/service/client.conf
{
    remote_build = {
        connect = "root@127.0.0.1:9691"
  }
}
```

!> 如果去掉用户名，也没配置 token，那就是匿名模式，如果服务器也没配置 token，就是完全禁用认证，直接连接。

#### 可信主机验证

另外，为了更进一步提高安全性，我们还提供了服务端可信主机验证，如果在服务器配置的 known_hosts 列表中，配置了可以连接的客户端主机 ip 地址，
那么只有这些主机可以成功连接上这台服务器，其他主机对它的连接都会被提示为不可信而拒绝连接，即使 token 和密码认证都没问题也不行。

```bash
$ cat ~/.xmake/service/server.conf
{
    logfile = "/Users/ruki/.xmake/service/logs.txt",
    server = {
        tokens = {
            "4b928c7563a0cba10ff4c3f5ca0c8e24"
        },
        known_hosts = { "127.0.0.1", "xx.xx.xx.xx"}
    }
}
```

### 连接远程的服务器

接下来，我们只需要进入需要远程编译的工程根目录，执行 `xmake service --connect` 命令，进行连接。

如果是 token 认证模式，那么不需要的额外的密码输入，直接连接。

```console
$ xmake create test
$ cd test
$ xmake service --connect
<remote_build_client>: connect 192.168.56.110:9091 ..
<remote_build_client>: connected!
<remote_build_client>: sync files in 192.168.56.110:9091 ..
Scanning files ..
Comparing 3 files ..
    [+]: src/main.cpp
    [+]: .gitignore
    [+]: xmake.lua
3 files has been changed!
Archiving files ..
Uploading files with 1372 bytes ..
<remote_build_client>: sync files ok!
```

如果是密码认证，那么会提示用户输入密码，才能继续连接。

```bash
$ xmake service --connect
Please input user root password:
000000
<remote_build_client>: connect 127.0.0.1:9691 ..
<remote_build_client>: connected!
<remote_build_client>: sync files in 127.0.0.1:9691 ..
Scanning files ..
Comparing 3 files ..
    [+]: xmake.lua
    [+]: .gitignore
    [+]: src/main.cpp
3 files has been changed!
Archiving files ..
Uploading files with 1591 bytes ..
<remote_build_client>: sync files ok!
```

如果密码不对，就会提示错误。

```bash
$ xmake service --connect
Please input user root password:
123
<remote_build_client>: connect 127.0.0.1:9691 ..
<remote_build_client>: connect 127.0.0.1:9691 failed, user and password are incorrect!
```

### 远程构建工程

连接成功后，我们就可以像正常本地编译一样，进行远程编译。

```console
$ xmake
<remote_build_client>: run xmake in 192.168.56.110:9091 ..
checking for platform ... macosx
checking for architecture ... x86_64
checking for Xcode directory ... /Applications/Xcode.app
checking for Codesign Identity of Xcode ... Apple Development: waruqi@gmail.com (T3NA4MRVPU)
checking for SDK version of Xcode for macosx (x86_64) ... 11.3
checking for Minimal target version of Xcode for macosx (x86_64) ... 11.4
[ 25%]: ccache compiling.release src/main.cpp
[ 50%]: linking.release test
[100%]: build ok!
<remote_build_client>: run command ok!
```

### 远程运行目标程序

我们也可以像本地运行调试那样，远程运行调试编译的目标程序。

```console
$ xmake run
<remote_build_client>: run xmake run in 192.168.56.110:9091 ..
hello world!
<remote_build_client>: run command ok!
```

### 远程重建工程

```console
$ xmake -rv
<remote_build_client>: run xmake -rv in 192.168.56.110:9091 ..
[ 25%]: ccache compiling.release src/main.cpp
/usr/local/bin/ccache /usr/bin/xcrun -sdk macosx clang -c -Qunused-arguments -arch x86_64 -mmacosx-version-min=11.4 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX11.3.sdk -fvisibility=hidden -fvisibility-inlines-hidden -O3 -DNDEBUG -o build/.objs/test/macosx/x86_64/release/src/main.cpp.o src/main.cpp
[ 50%]: linking.release test
"/usr/bin/xcrun -sdk macosx clang++" -o build/macosx/x86_64/release/test build/.objs/test/macosx/x86_64/release/src/main.cpp.o -arch x86_64 -mmacosx-version-min=11.4 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX11.3.sdk -stdlib=libc++ -Wl,-x -lz
[100%]: build ok!
<remote_build_client>: run command ok!
```

### 远程配置编译参数

```console
$ xmake f --xxx --yy
```

### 手动同步工程文件

连接的时候，会自动同步一次代码，后期代码改动，可以执行此命令来手动同步改动的文件。

```console
$ xmake service --sync
<remote_build_client>: sync files in 192.168.56.110:9091 ..
Scanning files ..
Comparing 3 files ..
    [+]: src/main.cpp
    [+]: .gitignore
    [+]: xmake.lua
3 files has been changed!
Archiving files ..
Uploading files with 1372 bytes ..
<remote_build_client>: sync files ok!
```

### 断开远程连接

针对当前工程，断开连接，这仅仅影响当前工程，其他项目还是可以同时连接和编译。

```console
$ xmake service --disconnect
<remote_build_client>: disconnect 192.168.56.110:9091 ..
<remote_build_client>: disconnected!
```

### 查看服务器日志

```console
$ xmake service --logs
```

### 清理远程服务缓存和构建文件

我们也可以手动清理远程的任何缓存和构建生成的文件。

```console
$ cd projectdir
$ xmake service --clean
```

## 分布式编译

Xmake 提供了内置的分布式编译服务，通常它可以跟 本地编译缓存，远程编译缓存 相互配合，实现最优的编译的加速。

另外，它是完全跨平台支持，我们不仅支持 gcc/clang，也能够很好地支持 Windows 和 msvc。

对于交叉编译，只要交叉工具链支持，我们不要求服务器的系统环境，即使混用 linux, macOS 和 Windows 的服务器资源，也可以很好的实现分布式编译。

### 开启服务

我们可以指定 `--distcc` 参数来开启分布式编译服务，当然如果不指定这个参数，xmake 会默认开启所有服务端配置的服务。

```console
$ xmake service --distcc
<distcc_build_server>: listening 0.0.0.0:9093 ..
```

我们也可以开启服务的同时，回显详细日志信息。

```console
$ xmake service --distcc -vD
<distcc_build_server>: listening 0.0.0.0:9093 ..
```

### 以 Daemon 模式开启服务

```console
$ xmake service --distcc --start
$ xmake service --distcc --restart
$ xmake service --distcc --stop
```

### 配置服务端

我们首先，运行 `xmake service` 命令，它会自动生成一个默认的 `server.conf` 配置文件，存储到 `~/.xmake/service/server.conf`。

```bash
$ xmake service
generating the config file to /Users/ruki/.xmake/service/server.conf ..
an token(590234653af52e91b9e438ed860f1a2b) is generated, we can use this token to connect service.
generating the config file to /Users/ruki/.xmake/service/client.conf ..
<distcc_build_server>: listening 0.0.0.0:9693 ..
```

然后，我们编辑它，修复服务器的监听端口（可选）。

```bash
$ cat ~/.xmake/service/server.conf
{
    distcc_build = {
        listen = "0.0.0.0:9693",
        workdir = "/Users/ruki/.xmake/service/server/distcc_build"
    },
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    tokens = {
        "590234653af52e91b9e438ed860f1a2b"
    }
}
```

### 配置客户端

客户端配置文件在 `~/.xmake/service/client.conf`，我们可以在里面配置客户端需要连接的服务器地址。

我们可以在 hosts 列表里面配置多个服务器地址，以及对应的 token。

!> 分布式编译，推荐使用 token 认证模式，因为密码模式，每台服务器连接时候都要输入一次密码，很繁琐。

```console
$cat ~/.xmake/service/client.conf
{
    distcc_build = {
        hosts = {
            {
                connect = "127.0.0.1:9693",
                token = "590234653af52e91b9e438ed860f1a2b"
            }
        }
    }
}
```

### 用户认证和授权

关于用户认证和授权，可以参考 [远程编译/用户认证和授权](/#/zh-cn/guide/other_features?id=%e7%94%a8%e6%88%b7%e8%ae%a4%e8%af%81%e5%92%8c%e6%8e%88%e6%9d%83) 里面的详细说明，用法是完全一致的。

### 连接服务器

配置完认证和服务器地址后，就可以输入下面的命令，将当前工程连接到配置的服务器上了。

我们需要在连接时候，输入 `--distcc`，指定仅仅连接分布式服务。

```bash
$ cd projectdir
$ xmake service --connect --distcc
<client>: connect 127.0.0.1:9693 ..
<client>: 127.0.0.1:9693 connected!
```

我们也可以同时连接多个服务，比如分布式编译和远程编译缓存服务。

```hash
$ xmake service --connect --distcc --ccache
```

!> 如果不带任何参数，默认连接的是远程编译服务。

### 分布式编译项目

连接上服务器后，我们就可以像正常本地编译那样，进行分布式编译了，例如：

```bash
$ xmake
...
[ 93%]: ccache compiling.release src/demo/network/unix_echo_client.c         ----> local job
[ 93%]: ccache compiling.release src/demo/network/ipv6.c
[ 93%]: ccache compiling.release src/demo/network/ping.c
[ 93%]: distcc compiling.release src/demo/network/unix_echo_server.c.         ----> distcc job
[ 93%]: distcc compiling.release src/demo/network/http.c
[ 93%]: distcc compiling.release src/demo/network/unixaddr.c
[ 93%]: distcc compiling.release src/demo/network/ipv4.c
[ 94%]: distcc compiling.release src/demo/network/ipaddr.c
[ 94%]: distcc compiling.release src/demo/math/fixed.c
[ 94%]: distcc compiling.release src/demo/libm/float.c
[ 95%]: ccache compiling.release src/demo/libm/double.c
[ 95%]: ccache compiling.release src/demo/other/test.cpp
[ 98%]: archiving.release libtbox.a
[ 99%]: linking.release demo
[100%]: build ok!
```

其中，带有 distcc 字样的是远程编译任务，其他的都是本地编译任务，默认 xmake 还开启了本地编译缓存，对分布式编译结果进行缓存，避免频繁请求服务器。

另外，我们也可以开启远程编译缓存，跟其他人共享编译缓存，进一步加速多人协同开发的编译。

### 断开连接

```bash
$ xmake service --disconnect --distcc
```

### 指定并行编译任务数

我们先简单介绍下，目前根据主机 cpu core 数量默认计算的并行任务数：

```lua
local default_njob = math.ceil(ncpu * 3 / 2)
```

因此，如果不开启分布式编译，默认的最大并行编译任务数就是这个 default_njob。

如果开启分布式编译后，默认的并行编译任务数就是：

```lua
local maxjobs = default_njob + server_count * server_default_njob
```

#### 修改本地并行任务数

我们只需要通过 `-jN` 就能指定本地并行任务数，但是它不会影响服务端的并行任务数。

```bash
$ xmake -jN
```

#### 修改服务端并行任务数

如果要修改服务端的并行任务数，需要修改客户端的配置文件。

```bash
$cat ~/.xmake/service/client.conf
{
    distcc_build = {
        hosts = {
            {
                connect = "127.0.0.1:9693",
                token = "590234653af52e91b9e438ed860f1a2b",
                njob = 8   <------- modify here
            },
            {
                connect = "192.168.01:9693",
                token = "590234653af52e91b9e438ed860f1a2b",
                njob = 4
            }
        }
    }
}
```

可以对每个服务器主机，添加 `njob = N` 参数配置，指定这台服务器可以提供的并行任务数。

### 分布式编译 Android 项目

xmake 提供的分布式编译服务是完全跨平台的，并且支持 Windows, Linux, macOS, Android, iOS 甚至交叉编译。

如果要进行 Android 项目编译，只需要在服务端配置中，增加 `toolchains` 工具链配置，提供 NDK 的跟路径即可。

```bash
$ cat ~/.xmake/service/server.conf
{
    distcc_build = {
        listen = "0.0.0.0:9693",
        toolchains = {
            ndk = {
                ndk = "~/files/android-ndk-r21e"   <------------ here
            }
        },
        workdir = "/Users/ruki/.xmake/service/server/distcc_build"
    },
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    tokens = {
        "590234653af52e91b9e438ed860f1a2b"
    }
}
```

然后，我们就可以像正常本地编译那样，分布式编译 Android 项目，甚至可以配置多台 Windows, macOS, Linux 等不同的服务器主机，做为分布式编译服务的资源，来编译它。

只需要下载对应平台的 NDK 就行了。

```bash
$ xmake f -p android --ndk=~/files/xxxx
$ xmake
```

### 分布式编译 iOS 项目

编译 iOS 项目更加简单，因为 Xmake 通常能自动检测到 Xcode，所以只需要像正常本地一样，切一下平台到 ios 即可。

```bash
$ xmake f -p iphoneos
$ xmake
```

### 分布式交叉编译配置

如果要分布式交叉编译，我们需要在服务端配置工具链 sdk 路径，例如：

```bash
$ cat ~/.xmake/service/server.conf
{
    distcc_build = {
        listen = "0.0.0.0:9693",
        toolchains = {
            cross = {
                sdkdir = "~/files/arm-linux-xxx"   <------------ here
            }
        },
        workdir = "/Users/ruki/.xmake/service/server/distcc_build"
    },
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    tokens = {
        "590234653af52e91b9e438ed860f1a2b"
    }
}
```

其中，toolchains 下，每一项对应一个工具链，这里配置为 `cross = {}` 交叉工具链，对应 `toolchain("cross")`。

工具链里面我们可以配置 `sdkdir`, `bindir`, `cross` 等等，对应 `toolchain("cross")` 里面的 `set_sdkdir`, `set_bindir` 和 `set_cross` 等接口配置。

如果交叉工具链比较规范，我们通常只需要配置 `sdkdir`，xmake 就能自动检测到。

而客户端编译也只需要指定 sdk 目录。

```bash
$ xmake f -p cross --sdk=/xxx/arm-linux-xxx
$ xmake
```

### 清理服务器缓存

每个项目在服务端的编译，都会产生一些缓存文件，他们都是按工程粒度分别存储的，我们可以通过下面的命令，对当前工程清理每个服务器对应的缓存。

```bash
$ xmake service --clean --distcc
```

### 一些内部优化

1. 缓存服务器端编译结果，避免重复编译
2. 本地缓存，远程缓存优化，避免不必要的服务端通信
3. 服务器负载均衡调度，合理分配服务器资源
4. 预处理后小文件直接本地编译，通常会更快
5. 大文件实时压缩传输，基于 lz4 快速压缩
6. 内部状态维护，相比 distcc 等独立工具，避免了频繁的独立进程加载耗时，也避免了与守护进程额外的通信

## 本地编译缓存

默认，Xmake 就会开启本地缓存，2.6.5 之前的版本默认使用外置的 ccache，而 2.6.6 之后版本，Xmake 提供了内置的跨平台本地缓存方案。

相比 ccache 等第三方独立进程，xmake 内部状态维护，更加便于优化，也避免了频繁的独立进程加载耗时，也避免了与守护进程额外的通信。

另外，内置的缓存能够更好的支持跨平台，Windows 上 msvc 也能够很好的支持，而 ccache 仅仅支持 gcc/clang。

当然，我们也可以通过下面的命令禁用缓存。

```bash
$ xmake f --ccache=n
```

注：不管是否使用内置本地缓存，配置名都是 `--ccache=`，意思是 c/c++ 构建缓存，而不仅仅是指 ccache 工具的名字。

我们如果想继续使用外置的其他缓存工具，我们也是可以通过下面的方式来配置。

```bash
$ xmake f --ccache=n --cxx="ccache gcc" --cc="ccache gcc"
$ xmake
```

## 远程编译缓存

除了本地缓存，我们也提供了远程缓存服务，类似 mozilla 的 sscache，如果只是个人开发，平常不会用到它。

但是，如果是公司内部多人协同开发一个大型项目，仅仅靠分布式编译和本地缓存，是不够的。我们还需要对编译的对象文件缓存到独立的服务器上进行共享。

这样，其他人即使首次编译，也不需要每次都去分布式编译它，直接从远程拉取缓存来加速编译。

另外，Xmake 提供的远程缓存服务，也是全平台支持的，不仅支持 gcc/clang 还支持 msvc。

### 开启服务

我们可以指定 `--ccache` 参数来开启远程编译缓存服务，当然如果不指定这个参数，xmake 会默认开启所有服务端配置的服务。

```console
$ xmake service --ccache
<remote_cache_server>: listening 0.0.0.0:9092 ..
```

我们也可以开启服务的同时，回显详细日志信息。

```console
$ xmake service --ccache -vD
<remote_cache_server>: listening 0.0.0.0:9092 ..
```

### 以 Daemon 模式开启服务

```console
$ xmake service --ccache --start
$ xmake service --ccache --restart
$ xmake service --ccache --stop
```

### 配置服务端

我们首先，运行 `xmake service` 命令，它会自动生成一个默认的 `server.conf` 配置文件，存储到 `~/.xmake/service/server.conf`。

```bash
$ xmake service
generating the config file to /Users/ruki/.xmake/service/server.conf ..
an token(590234653af52e91b9e438ed860f1a2b) is generated, we can use this token to connect service.
generating the config file to /Users/ruki/.xmake/service/client.conf ..
<remote_cache_server>: listening 0.0.0.0:9692 ..
```

然后，我们编辑它，修复服务器的监听端口（可选）。

```bash
$ cat ~/.xmake/service/server.conf
{
    distcc_build = {
        listen = "0.0.0.0:9692",
        workdir = "/Users/ruki/.xmake/service/server/remote_cache"
    },
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    tokens = {
        "590234653af52e91b9e438ed860f1a2b"
    }
}
```

### 配置客户端

客户端配置文件在 `~/.xmake/service/client.conf`，我们可以在里面配置客户端需要连接的服务器地址。

我们可以在 hosts 列表里面配置多个服务器地址，以及对应的 token。

```console
$cat ~/.xmake/service/client.conf
{
    remote_cache = {
            connect = "127.0.0.1:9692,
            token = "590234653af52e91b9e438ed860f1a2b"
        }
    }
}
```

### 用户认证和授权

关于用户认证和授权，可以参考 [远程编译/用户认证和授权](/#/zh-cn/guide/other_features?id=%e7%94%a8%e6%88%b7%e8%ae%a4%e8%af%81%e5%92%8c%e6%8e%88%e6%9d%83) 里面的详细说明，用法是完全一致的。

### 连接服务器

配置完认证和服务器地址后，就可以输入下面的命令，将当前工程连接到配置的服务器上了。

我们需要在连接时候，输入 `--ccache`，指定仅仅连接远程编译缓存服务。

```bash
$ cd projectdir
$ xmake service --connect --ccache
<client>: connect 127.0.0.1:9692 ..
<client>: 127.0.0.1:9692 connected!
```

我们也可以同时连接多个服务，比如分布式编译和远程编译缓存服务。

```hash
$ xmake service --connect --distcc --ccache
```

!> 如果不带任何参数，默认连接的是远程编译服务。

### 断开连接

```bash
$ xmake service --disconnect --ccache
```

### 清理服务器缓存

我们也可以通过下面的命令，清理当前工程对应的远程服务器上的缓存。

```bash
$ xmake service --clean --ccache
```

而如果我们执行 `xmake clean --all`，在连接了远程服务的状态下，也会去自动清理所有的缓存。

### 一些内部优化

1. 拉取远程缓存的快照，通过 bloom filter + lz4 回传本地后，用于快速判断缓存是否存在，避免频繁的查询服务端缓存信息
2. 配合本地缓存，可以避免频繁地请求远程服务器，拉取缓存。
3. 内部状态维护，相比 sscache 等独立工具，避免了频繁的独立进程加载耗时，也避免了与守护进程额外的通信
