# 为您的古董系统进行交叉编译
![pic1](./CCPS_pictures/pic1)
- 图1 Linux内核2.6.30的首批用户
- ----------------------------------------
## 起因
当你发现被完好封存在“网络琥珀”中的嵌入式设备时，是将其捐赠给最近的博物馆还是尝试破解它？如果该设备实际上是史前设备（也许是恐龙制造的呢，这些家伙粗短的手臂按不到F5和F10，也不会操心为系统提供个GDB什么的），这要怎么搞？  
![pic2](./CCPS_pictures/pic2)
- 图2 化石记录表明，此工具乃尼安德特黑客的最爱，适合未开化时代的优雅工具
- ----------------------------------------
## 发掘古董的相关功能
为了正确的为设备配置工具链，你需要做些研究以查清它许多方面的特征，包括：  
1. 处理器名称/型号；
2. 是否存在FPU；
3. 指令集架构；
4. ABI（若适用）
5. Linux内核版本
6. libc类型及版本
7. GCC版本

此教程将使用一个运行在Linux2.6.30内核版本，ARMv5架构的嵌入式系统作为示例目标。尽管您对于在图坦卡蒙墓葬中发现的任何嵌入式设备找不到上述功能的完全指南 ，但以下常规策略可能会有用：  
1. 首先搞清楚设备名称，型号和制造商。 您应该对设备上的处理器尤其感兴趣。通常，只需检查设备上的所有标签或商标，即可找到此信息。 如果设备具有FCCID，请在[FCCID数据库](https://www.fcc.gov/oet/ea/fccid)中搜索此ID。 尝试Google搜索通过条形码或二维码码找到的所有信息。
2. 如果能够找到设备或其处理器的名称，请查找设备的数据表。 这些通常可以在制造商的网站上找到。数据表可以帮助我们找出设备上正在运行的硬件，并且通常可以告诉我们指令集架构以及该设备是否支持硬件浮点。
3. 其他信息需要访问设备的文件系统或至少您了解运行于设备上的二进制文件。如果您已有权访问文件系统，请使用grep对关键字执行搜索：
```
grep -rn '/root/of/unpacked/filesystem' -e 'keyword'
```
>- 如果uname -a不起作用，您也许还可以在/etc/路径下包含version、release或是issue的文件中找到有关linux版本或是发行版的信息
>- 处理器信息通常可以在/proc/路径下找到。 检查/proc/目录是否包含名为“version”的文件，如果存在，则通常包含有用的信息。
>- 可以通过查看/lib/libc.so.X符号链接找到libc类型和版本。例如，上述ARMv5设备上的/lib/目录包含libuClibc-0.9.30.so，这表明它正在使用uClibc 0.9.30版。其他常见的C库包括glibc、eglibc和musl。
>- 在/lib/libgcc_s.so.X上对“gcc”进行字符串搜索通常不仅可以帮助您找到用于构建二进制文件和文件系统的GCC版本，而且还可以为您提供有关构建时所用工具链的信息。 回到示例设备，我们在/lib/libgcc_s.so.1中发现以下字符串：  
>`/here/workdir/factory/build_armv5l-timesys-linux-uclibcgnueabi/gcc-4.3.3/gcc- 4.3.3/libgcc/../gcc/config/arm/lib1funcs.asm`  
> 这表明设备运行于ARMv5l，使用[Timesys factory](https://linuxlink.timesys.com/factory/latest/)生成工具链，GCC版本为4.3.3，ABI为EABI，而libc为uClibc。
4. 如果您能够访问已知在设备上运行的二进制文件（例如，下载的固件），则可以使用[readelf](https://linux.die.net/man/1/readelf)查找有关设备的信息。
>- 要查看架构的名称，执行：  
> `readelf -h /path/to/binary | grep -i "Machine:"`  
>- 要查看字节序，执行：  
> `readelf -h /path/to/binary | grep -i "Data:"`  
>- 要查看ABI，执行：  
>`readelf -h /path/to/binary | grep -i "Flags:"`  
> 或：  
>`readelf -A /path/to/binary | grep -i "Attribute Section:"`  
>- 要查看架构版本，执行：  
>`readelf -A /path/to/binary | grep -i "Tag_CPU_arch"`  
## 在虚拟机中设置史前Linux
“要想黑掉恐龙，就必须想恐龙一样思考”。自己想一想，“远古巨蜥会使用什么操作系统”？答案是Ubuntu 12.04。有一个好处是，Ubuntu 12.04还是支持[Buildroot 2009.08](https://buildroot.org/downloads/buildroot-2012.08.tar.gz)的Ubuntu的最新版本，这是最后支持较旧内核（> 2.6.X）的Buildroot版本。 如果您的系统不是那么古老，则可以使用较新版本的Buildroot，但是对于本指南，我们将继续使用该版本。 首先，将其安装到虚拟机上。 从[此处](http://releases.ubuntu.com/12.04.5/)获取Ubuntu 12.04的镜像开始。  

配置好虚拟机后，确保运行：
```
sudo apt-get update
sudo apt-get upgrade
```
为这个史前的Ubuntu获取最新的更新至关重要。 哦，它也很有用，这样您可以自动获取到使用wget工具为我们的工具链下载软件包时所需要的证书。
## 为“太太太太太爷爷”安装Buildroot
您可以从此处下载[Buildroot 2009.08](https://buildroot.org/downloads/buildroot-2012.08.tar.gz)。本着追求历史准确性的精神，以下说明将使用适合该时代的技术，也称为CLI。其首字母缩略词的原始含义已无从考证，但学者们猜测它可能代表“ 穴居人语言界面(Caveman Language Interface)”。 无论是哪种情况，这种近乎象形的文字似乎总能完成工作：  
```
wget http://buildroot.org/downloads/buildroot-2009.08.tar.gz
tar xzf buildroot-2009.08.tar.gz
rm -f buildroot-2009.08.tar.gz
cd buildroot-2009.08
```

## 生成交叉编译工具链
### 配置Buildroot
要实际生成ARM交叉编译工具链，请运行以下命令：
```
sudo apt-get install automake bison flex gettext g++ libncurses-dev texinfo
make menuconfig
```
注：automake，bison，flex，gettext，g++，libncurses-dev和texinfo都是Buildroot依赖项。  

如果一切顺利，您现在将看到乍一看似乎是图形界面的内容。别被骗了。这只是一种技巧，旨在使具有各种背景颜色的文本看起来像GUI。在此菜单中，您可以自定义Buildroot将其以各种方式生成工具链。您应该选择哪些选项将取决于您在信息收集阶段发现的内容。对于我们的示例系统，选择了以下选项（其余选项保留为默认值）：  
| ------ | ------ |
| :----- | :-----: |  
| Target Architechure | arm |
| Target Architecture Variant | arm926t |
| Target ABI | EABI |
| Target Options -> Atmel Device Support | Yes |
| Target Options -> Atmel Device Support -> Board Support for the Atmel AT91 Range of Microcessors | Yes |
| Target Options -> Atmel Device Support -> Allow All ARM Targets | No |
| Target Options -> Atmel Device Support -> AT92 Device | Atmel AT91SAM9263 Microprocessor |
| Toolchain -> Kernel Header Options | Linux 2.6.29.X Kernel Headers |
| Toolchain -> uClibc C Library Version | uClibc 0.9.30 |
| Toolchain -> Thread Library Debugging | Yes |
| Toolchain -> Build GDB Debugger for the Target | Yes |
| Toolchain -> Enable Toolchain Locale/i18n Support? | Yes |
| Toolchain -> Use Software Floating Point by Default | Yes |
| Package Selection for the Target -> BusyBox | No |
| Package Selection for the Target -> strace | Yes |
| Package Selection for the Target -> Networking -> tcpdump | Yes |

所选选项的说明：  
- 从信息收集中，我们知道该设备使用的是Atmel AT91SAM9263，一款arm926t芯片。
- 尽管我们知道设备运行的是Linux 2.6.30，但我们还是故意选择稍旧（甚至认为是不可能的版本）的2.6.29.X版本是安全的，因为内核标头是向后兼容的，我们不确定有关Buildroot将生成的次要版本号的信息。
- 我们要确保启用线程库调试，这样我们就可以使用生成的GDB调试多线程程序。
- 在这种情况下，语言环境支持最终成为“陷阱(gotcha)”，因为我们发现在未启用该选项的情况下生成的libc不会生成设备上现有二进制文件所需的某些符号。
- 因为我们知道设备没有FPU，所以选择了“默认情况下使用软件浮点”。 如果您不确定设备的浮点功能，这是一个安全的选择。
- 我们不想构建BusyBox，因为我们不需要它，但是我们确实需要GDB，strace和tcpdump以用于调试。  
从此退出menuconfig并保存您的配置。

### 下载依赖项
对于Buildroot的旧版本，Buildroot将尝试从某些已失效的存储库中下载软件包。因此，在执行make之前，您可能需要手动下载一些Buildroot所需的归档文件，因为某些镜像无法完成自动下载。就我们而言，我们需要下载以下软件包：  
- [gdb-6.8.tar.bz2](http://ftp.lanet.lv/ftp/GNU/gdb/gdb-6.8.tar.bz2)
- [zlib-1.2.3.tar.bz2](https://sourceforge.net/projects/libpng/files/zlib/1.2.3/zlib-1.2.3.tar.bz2/download)
- [strace-4.5.18.tar.bz2](https://sourceforge.net/projects/strace/files/strace/4.5.18/strace-4.5.18.tar.bz2/download)
- [fakeroot_1.9.5.tar.gz](http://snapshot.debian.org/archive/debian/20080427T000000Z/pool/main/f/fakeroot/fakeroot_1.9.5.tar.gz)
- [genext2fs-1.4.tar.gz](https://sourceforge.net/projects/genext2fs/files/genext2fs/1.4/genext2fs-1.4.tar.gz/download)

不可避免地，以上镜像链接也会失效，因此，取决于您的意愿，您可能需要自己查找文件。 如果包含文件的归档文件名为“fossils”，通常是一个好兆头：

![pic3](./CCPS_pictures/pic3)

下载完成后，使用以下命令将所有这些文件移动到Buildroot的下载目录中：
```
mv -t /path/to/buildroot-2009.08/dl/ gdb-6.8.tar.bz2 \
                                     zlib-1.2.3.tar.bz2 \
                                     strace-4.5.18.tar.bz2 \
                                     fakeroot_1.9.5.tar.gz \
                                     genext2fs-1.4.tar.gz
```

### 构建工具链
执行：  
`make`  
您可能需要在此时拿出日晷，因为此步骤需要3到7个月才能完成。  
如果按照我们的示例进行操作，最终将遇到以下构建错误：
```
mkdir /home/buildroot/buildroot-2009.08/build_arm/makedevs-host
cp target/makedevs/makedevs.c /home/buildroot/buildroot-2009.08/build_arm/makedevs-host /usr/bin/gcc -Wall -Werror -O2 /home/buildroot/buildroot-2009.08/build_arm/makedevs- host/makedevs.c -o /home/buildroot/buildroot-2009.08/build_arm/makedevs-host/makedevs /home/buildroot/buildroot-2009.08/build_arm/makedevs-host/makedevs.c: In function ‘main’: /home/buildroot/buildroot-2009.08/build_arm/makedevs-host/makedevs.c:366:6: error: variable ‘ret’ set but not used [-Werror=unused-but-set-variable]
cc1: all warnings being treated as errors
make: *** [/home/buildroot/buildroot-2009.08/build_arm/makedevs-host/makedevs] Error 1
```
看来，我们的祖先以其古老的智慧决定，在编译带有未解决的警告的C文件时，启用-Werror是一个好主意。但是您不介意来一点历史修正，对吧？ 在Buildroot根目录中，使用以下命令打开该文件：  
```
vi build_arm/makedevs-host/makedevs.c
```
注：使用vi编辑文件以获得完整的尼安德特人体验极其重要。  

找到return 0;，在main()函数末尾的语句（位于第534行），并将其更改为return ret;。保存更改并运行：  
`make`  

这次，构建可能停止，并显示以下输出：  
```
makedevs: line 41: regular file '/home/buildroot/buildroot- 2009.08/project_build_arm/uclibc/root/bin/busybox' does not exist: No such file or directory
-rw-rw-r-- 1 buildroot buildroot 6979584 Oct 19 15:55 /home/buildroot/buildroot-2009.08/binaries/uclibc/rootfs.arm.ext2
rm -f /home/buildroot/buildroot-2009.08/project_build_arm/uclibc/.fakeroot*
```
用恐龙的话来说，把这个在rm -f命令中出现的找不到文件的报错翻译为“构建成功”。 相信我们的话。您的目标的二进制文件可以在/ path / to / buildroot-2009.08/project_build_arm/uclibc/root/usr/bin/中找到。

## 使用Make生成工具
如果要在首次运行make之后为目标构建其他软件包，则可以使用make \<name of binary\>进行构建。例如，从buildroot-2009.08目录运行`make strace`将构建strace。

## 使用交叉编译工具链从源码构建二进制程序
如果Buildroot无法自动构建您想要的软件包，或者如果您要构建静态软件包，您将不得不诉诸于从源码手动配置和构建的迷失之术，就像古代民族从被杀死的野兽的皮中制作自己的衣服一样。  
不幸的是，由于不同的程序使用不同的构建方法，因此没有通用的策略来实现这一目标。 话虽这么说，但许多人都使用configure->make->make install工作流程，并且有一些通用步骤可以带您到达大多数地方。 我们将通过一个例子来说明。特别的，我们将从源码开始为ARM系统静态构建GDB。
```
wget http://ftp.lanet.lv/ftp/GNU/gdb/gdb-6.8.tar.bz2 tar xjf gdb-6.8.tar.bz2
rm -f gdb-6.8.tar.bz2
cd gdb-6.8
mkdir build-gdb
cd build-gdb
export PATH=$PATH:/path/to/buildroot-2009.08/build_arm/staging_dir/usr/bin/ export CFLAGS=-static
export CPPFLAGS=-static
export LDFLAGS=-static
../configure --enable-static --disable-shared --host=arm-linux-uclibcgnueabi
make
```
(您应将所有“/path/to”替换为系统中这些目录的实际位置。)  
这些步骤是许多程序的交叉编译过程所共有的。其中包括将交叉编译工具链的位置放在PATH的靠前的位置，通过--enable-static和-disable-shared（配置步骤）和XFLAGS=-static（make步骤）启用静态链接，并将主机设置为生成的工具链的前缀（--host=...）。  
请记住，GDB和其他工具具有自己的依赖关系，如果您的工具链不是使用这些依赖关系构建的，则这种手动构建将失败。 例如，GDB需要ncurses库，默认情况下，Buildroot不会为目标生成该库。如果您只是尝试构建Buildroot支持的静态版本的软件包，则只需在Buildroot的menuconfig中选择该软件包，然后重新运行make即可将其所有依赖项构建到工具链中。如果所需的工具不在Buildroot的软件包列表中，则可以尝试手动查找依赖项，并使用“/”键在menuconfig中搜索它们。  
另一方面，如果只想编译编写的简单C程序（例如，命名为foo.c），则可以运行以下命令：
```
export PATH=$PATH:/path/to/buildroot-2009.08/build_arm/staging_dir/usr/bin/arm-linux-gcc -Wall -g -o foo foo.c
```
在这里，设置了GCC标志以：
1. 使用`-g`为GDB启用调试符号。通常这是一个不错的选择，因为我们可能会测试和调试为目标编写的任何代码。
2. 为了安全起见，请使用-Wall启用输出所有警告。

## 我怎么知道这穴居人的巫术是否有效？
验证通过交叉编译生成的二进制文件的最可靠方法是仅在目标平台上运行它们，但是您也可以使用[file](https://linux.die.net/man/1/file)命令执行快速健全性检查，如下所示：
```
file /path/to/binary
```
例如，在使用上一节概述的方法创建的gdb二进制文件上运行文件会产生以下输出：  
```
gdb: ELF 32-bit LSB executable, ARM, version 1 (SYSV), statically linked, not stripped
```
这是一个好兆头，因为我们的确设定了小端序ARM架构，并且我们想要的是一个静态链接的二进制文件。为了了解的更全面，您可以运行：
```
readelf -hdA /path/to/binary
```
它将打印出整个ELF标头，任何必需的共享库的位置以及详细的体系结构信息。 在相同的GDB二进制文件上运行此命令将产生以下输出：
```
ELF Header:
    Magic: 7f454c46010101000000000000000000
    Class:  ELF32
    Data:   2's complement, little endian
    Version:    1 (current)
    OS/ABI: UNIX - System V
    ABI Version:    0
    Type:   EXEC (Executable file)
    Machine:    ARM
    Version:    0x1
    Entry point address:    0x80d0
    Start of program headers:   52 (bytes into file)
    Start of section headers:   11315256 (bytes into file)
    Flags:  0x4000002, has entry point, Version4 EABI
    Size of this header:    52 (bytes)
    Size of program headers:    32 (bytes)
    Number of program headers:  4
    Size of section headers:    40 (bytes)
    Number of section headers:  29
    Section header string table index:  26

There is no dynamic section in this file.
Attribute Section: aeabi
File Attributes
    Tag_CPU_name: "5T"
    Tag_CPU_arch: v5TE
    Tag_ARM_ISA_use: Yes
    Tag_THUMB_ISA_use: Thumb-1
    Tag_ABI_PCS_wchar_t: 4
    Tag_ABI_FP_denormal: Needed
    Tag_ABI_FP_exceptions: Needed
    Tag_ABI_FP_number_model: IEEE 754
    Tag_ABI_align_needed: 8-byte
    Tag_ABI_align_preserved: 8-byte, except leaf SP
    Tag_ABI_enum_size: int

```
此输出确认该二进制文件是使用EABI ABI为ARMv5TE构建的，并且没有动态节（因为它是静态链接的）。

## 写在最后
很难预料你在操作中会遇到什么问题，确切的过程可能因设备而异。与逆向工程领域中的许多事物一样，坚持是关键。 如果在运行make时遇到一些无法识别的构建错误，不要气馁，这是“正常的”。 [Buildroot邮件列表](http://lists.busybox.net/pipermail/buildroot/)是您的好帮手，很可能有人已经遇到您的问题并在那里记录了解决方案。  
此外，一定要有足够的时间为设备进行提前的信息收集。 您越了解目标的各种软件/硬件规格，就越容易构建工具链，该工具链将生成实际在设备上运行的程序。 这一点特别重要，因为在实际在目标上运行程序之前，您通常不会发现不兼容地方，并且更改单个工具链的设置通常需要完全重新编译，这使得迭代方法非常耗时。  
最后，尽管坚持至关重要，但请注意不要陷在不必要的深坑。着眼于大局(您实际上正在尝试解决的问题)可以帮助您解决这一问题。您真的需要从头开始构建工具链吗？也许有人已经为您的设备创建了一个。您是否真的需要GDB/strace等工具的确切版本？如果构建特定工具的难度太大，请尝试使用其他工具或其他版本。您的二进制文件真的需要静态构建吗？也许您的设备已经具有所需的库。否则，您可以将工具链环境中的文件复制到设备上，以使动态链接的可执行文件正常工作。  
祝您好运，Happy hacking！  

## Origin Link
- https://www.mcafee.com/enterprise/en-us/assets/misc/ms-cross-compiling-prehistoric-systems.pdf  
Origin author: Mark Bereza  
Translated by ghost461  