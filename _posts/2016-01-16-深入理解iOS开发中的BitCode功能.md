---
layout: post
category: 原创
tagline: "Supporting tagline"
tags: [iOS,BitCode,上架,LLVM,Clang]
---

##前言
做iOS开发的朋友们都知道,目前最新的Xcode7,新建项目默认就打开了bitcode设置.而且大部分开发者都被这个突如其来的bitcode功能给坑过导致项目编译失败,而这些因为bitcode而编译失败的的项目都有一个共同点,就是链接了第三方二进制的库或者框架,而这些框架或者库恰好没有包含bitcode的东西(暂且称为东西),从而导致项目编译不成功.所以每当遇到这个情况时候大部分人都是直接设置Xcode关闭bitcode功能,全部不生成bitcode.也不去深究这一开关背后隐藏的原理.中枪的请点个赞.

LLVM是目前苹果采用的编译器工具链,Bitcode是LLVM编译器的中间代码的一种编码,LLVM的前端可以理解为C/C++/OC/Swift等编程语言,LLVM的后端可以理解为各个芯片平台上的汇编指令或者可执行机器指令数据,那么,BitCode就是位于这两者直接的中间码. LLVM的编译工作原理是前端负责把项目程序源代码翻译成Bitcode中间码,然后再根据不同目标机器芯片平台转换为相应的汇编指令以及翻译为机器码.这样设计就可以让LLVM成为了一个编译器架构,可以轻而易举的在LLVM架构之上发明新的语言(前端),以及在LLVM架构下面支持新的CPU(后端)指令输出,虽然Bitcode仅仅只是一个中间码不能在任何平台上运行,但是它可以转化为任何被支持的CPU架构,包括现在还没被发明的CPU架构,也就是说现在打开Bitcode功能提交一个App到应用商店,以后如果苹果新出了一款手机并CPU也是全新设计的,在苹果后台服务器一样可以从这个App的Bitcode开始编译转化为新CPU上的可执行程序,可供新手机用户下载运行这个App.



##历史回顾
在iPhone出来之前,苹果主要的编译器技术是用经过稍微改进的GCC工具链来把Objective-C语言编写的代码编译出所指定的机器处理器上原生的可执行程序.编译器产生的可执行程序叫做"Fat Binaries"--类似于Windows下PE格式的exe和Linux下的ELF格式的二进制,不同的是,一个"Fat Binary"可以包含同一个程序的很多版本,所以同一个可执行文件可以在不同的处理器上运行.主要就是这个技术让苹果的硬件很容易的从PowerPC迁移到PowerPC64的处理器,以及后来再迁移到Intel和Intel64处理器.这个方案带来的负面影响就是同一个文件中存了多份可执行代码,除了当前机器可执行的那一份之外其他都是无用的,白占空间. 这个在市场上被称为"Universal Binary",在苹果从PowerPC迁移到Intel处理器的事情开始存在的(一个二进制文件既包含一份PowerPC版本和一份Intel版本).慢慢的后来又支持同时包含Intel 32bit和Intel 64bit. 在一个Fat binary中,又操作系统运行时根据处理器类型动态选择正确的二进制版本来运行,但是应用程序要支持不同平台的处理器的话,应用程序本身要多占用一些空间.当然也有一些瘦身的工具,比如lipo,可以用来移除fat binary中那些当前机器中不被支持的或者多余的可执行代码达到瘦身目的,lipo不会改变程序执行逻辑,仅仅只是文件的大小瘦身.

##编译器现状
随着移动设备移动互联网的深入发展,现在移动设备中的程序大小变得越来越重要了,主要是因为移动设备中不会有电脑上那么大的一个硬盘驱动器.还有就是苹果早就从原始的ARM处理器迁移到自家设计的A4,A5,A5X,A6,A7,A8,A8X,A9,A9X以及后续的A10处理器,他们的指令集已经发生了改变和原始ARM设计的有所区别,所有的这些变化都被iOS操作系统底层以及Xcode/LLVM编译工具向上层程序员一定程度的透明了,编译出来的程序会包含很多执行代码版本.当面对这个问题后,苹果投入大量成本迁移到LLVM编译器架构并使用bitcode的必要性越来越大.从最开始的把OPENGL编译为特定的GPU指令到把Clang编译器(LLCM的C/OC编译前端)支持Objective-C的改进并作为Xcode的默认编译器.


LLVM提供了一个虚拟指令集机制,它可以翻译出指定的所支持的处理器架构的执行代码(机器码).这个就使得为iOS应用程序的编译开发一个完全基于LLVM架构的工具链成为可能.而LLVM的这个虚拟的通用的指令集可以用很多种表示格式:

- 叫做IR的文本表示的汇编格式(像汇编语言);
- 转换为二进制数据表示的格式(像目标代码),这个二进制格式就是我们所说的bitcode.

Bitcode和传统的可执行指令集不同,他维护的是函数功能的类型和签名,比如,传统可执行指令集中,一系列(<=8)的布尔值可以压缩存储到单个字节中,但是在bitcode中他们是各自独自表示的.此外,逻辑运算操作(比如寄存器清零操作)也由他们对应的逻辑表示方法(`$R=0`);当这些BitCode要转换为特定机器平台的指令集时,他可以用经过针对特定机器平台优化过的汇编指令来代替:`xor eax, eax`.(这个汇编指令同样是寄存器<eax>清零操作).

然而bitcode他也不是完全独立于处理器平台和调用约定的.寄存器的大小在指令集中是一个相当重要的特性,众所周知,64bit寄存器可以比32bit寄存器存储更多的数据,生成64bit平台的bitcode和32bit平台的bitcode是明显不同的,还有,调用约定可以根据函数定义或者函数调用来定义,这些可以确定函数的参数传递是传寄存器值呢还是压栈. 一些编程语言还有一些像sizeof(long)这样的预处理指令,这些将在bitcode生成之前前被翻译.一般情况下,对于支持`fastcc`(fast calling convention)调用的64bit平台会生成与其一致的bitcode代码.

##苹果的要求
到此,让我们思考一下,为什么苹果默认要求watchOS和tvOS的App要上传bitcode? 因为把bitcode上传到他自己的中心服务器后,他可以为目标安装App的设备进行优化二进制,减小安装包的下载大小,当然iOS开发者也可以上传多个版本而不是打包到单个包里,但是这样会占用更多的存储空间. 最重要的是允许苹果可以在后台服务器对应用程序进行签名,而不用导出任何密钥到终端开发者那.

上传到服务器的bitcode给苹果带来更好处是: 以后新设计了新指令集的新CPU,可以继续从这份bitcode开始编译出新CPU上执行的可执行文件,以供用户下载安装.
但是bitcode给开发者带来的不便之处就是: 没用bitcode之前,当应用程序奔溃后,开发者可以根据获取的的奔溃日志再配上上传到苹果服务器的二进制文件的调试符号表信息可以还原程序运行过程到奔溃时后调用栈信息,对问题进行定位排查.但是用了bitcode之后,用户安装的二进制不是开发者这边生成的,而是苹果服务器经过优化后生成的,其对应的调试符号信息丢失了,也就无法进行前面说的还原奔溃现场找原因了.


目前,watchOS和tvOS应用发布必须上传带bitcode版本的包.iOS应用发布对bitcode的要求是可选的,用户可以在Xcode的项目设置中关闭. 相当于在编译的时候加一个标记:embed-bitcode-marker(调试构建) embed-bitcode(打包/真机构建).这个在clang编译器的参数是-fembed-bitcode,swift编译器的参数是-embed-bitcode.

##实践出真知
我们还是应该实际弄两个测试代码进行实践和检验一下比较好.做两次测试,第一次准备两个C语言源代码继续测试;第二次把其中一个转变为汇编语言源代码后再一个C代码和一个汇编代码一起重复之前的测试步骤进行对比校验差异.

- 1 . 如下两个全部是Objective-C代码:

test.m :

```objectivec
#import <Foundation/Foundation.h>
void greeting(void)
{
    NSLog(@"hello world!");
}
```

demo.m :

```objectivec
#import <Foundation/Foundation.h>
void demo(void)
{
    NSLog(@"demo func");
}
```

 用Clang编译成 ARM64 格式且带bitcode的目标文件test.o demo.o:

```bash
wuqiong:~ apple$ xcrun -sdk iphoneos clang -arch arm64 -fembed-bitcode -c test.m demo.m
```

 然后把两个目标文件打包为一个静态库文件:

```bash
wuqiong:~ apple$ xcrun -sdk iphoneos ar  -r libTest.a test.o demo.o
ar: creating archive libTest.a
```

 用Shell命令otool查看目标文件中是否包含bitcode段:

```bash
wuqiong:~ apple$ otool -l test.o |grep bitcode
  sectname __bitcode
  sectname __bitcode
```
 如果看到输出了2行`sectname __bitcode`,就是说明这静态库中的两个目标文件包含了bitcode.


-  2.下面把其中一个demo.m换成汇编语言再参与编译:

 用下面的命令把demo.m的C代码转换为ARM64汇编语言格式demo.s:

```bash
wuqiong:~ apple$ xcrun -sdk iphoneos clang -arch arm64 -S demo.m
wuqiong:~ apple$ cat demo.s
	.section	__TEXT,__text,regular,pure_instructions
	.ios_version_min 9, 2
	.globl	_demo
	.align	2
_demo:                                  ; @demo
	.cfi_startproc
; BB#0:
	stp	x29, x30, [sp, #-16]!
	mov	 x29, sp
Ltmp0:
	.cfi_def_cfa w29, 16
Ltmp1:
	.cfi_offset w30, -8
Ltmp2:
	.cfi_offset w29, -16
	adrp	x0, L__unnamed_cfstring_@PAGE
	add	x0, x0, L__unnamed_cfstring_@PAGEOFF
	bl	_NSLog
	ldp	x29, x30, [sp], #16
	ret
	.cfi_endproc

	.section	__TEXT,__cstring,cstring_literals
L_.str:                                 ; @.str
	.asciz	"demo func"

	.section	__DATA,__cfstring
	.align	4                       ; @_unnamed_cfstring_
L__unnamed_cfstring_:
	.quad	___CFConstantStringClassReference
	.long	1992                    ; 0x7c8
	.space	4
	.quad	L_.str
	.quad	9                       ; 0x9

	.section	__DATA,__objc_imageinfo,regular,no_dead_strip
L_OBJC_IMAGE_INFO:
	.long	0
	.long	0


.subsections_via_symbol
```

 然后删除`demo.m`这个C源代码,仅留下`test.m`和`demo.s`:

```bash
wuqiong:~ apple$ rm demo.m

```

 现在,我们来把`test.m`这个C源代码和`dmeo.s`这个汇编源代码来一起带着`-fembed-bitcode`参数来生成目标代码并打包为一个静态库:

```bash
wuqiong:~ apple$ xcrun -sdk iphoneos clang -arch arm64 -fembed-bitcode -c test.m demo.s
wuqiong:~ apple$ xcrun -sdk iphoneos ar -r libTest.a test.o demo.o

```
 然后我们再运行otool工具来检查这个新的静态库中包含的2个目标文件是否都带有bitcode段:

```bash
wuqiong:~ apple$ ar -t libTest.a
__.SYMDEF SORTED
test.o
demo.o
wuqiong:~ apple$ otool -l libTest.a | grep bitcode
  sectname __bitcode
```

很意外,这一次,只有一行`sectname __bitcode`输出,这就说明这两个目标文件,有一个不带有bitcode段,哪怕我们在编译的时候指定了参数`-fembed-bitcode`也没有用.至于具体是哪一个不带bitcode段,我们肯定知道就是那个从ARM64汇编语言编译过来的目标文件不带.


那么就得出一个结论,bitcode的生成,是由汇编语言以上的上层语言编译而来,和最前面所说的那样,他是上层语言与汇编语言(机器语言)之间的一个中间码.

目前我们日常的iOS应用开发中,一般不会需要用到汇编层面去优化的代码.所以我们主要关注第三方(开源)C代码,尤其是音视频编码解码这些计算密集型项目代码,关键计算的代码针对特定平台都有对应平台的汇编版本实现,当然也有C的实现,但是默认编译一般都是用的汇编版本,这样就会导致我们在编译这个开源代码的时候哪怕你带了`-fembed-bitcode`参数也仅仅只是让项目中的部分C代码的目标文件带了bitcode段,而那小数的汇编代码的目标文件一样不带bitcode段,这样编译出这个库交给上层开发者使用的时候,就会出现在打包上传或者真机调试的时候因为Xcode默认开了bitcode功能而链接失败,导致不能真机调试或者不能上传应用到AppStore.

##此文之初衷
 最近在辅导我戴维营战友们做手机音视频直播的App,调试的时候手机采集音视频,视频用h264编码,音频采用aac编码,通过RTMP协议往斗鱼直播频道发布媒体流,项目需要用`FFMPEG`和`libx264`两个开源项目,在编译为iOS框架库提供给学生用的时候,他们遇到了bitcode的问题,虽然可以采取直接关闭bitcode来避免错误,但是战友的求知欲必须满足,格物致知,必须让其知其究竟.

 `libx264`是VideoLan基金会管理的一个视频编解码的开源项目,其大量使用了各个平台的多媒体汇编指令进行了优化,在编译为不带bitcode的库的时候,完全按官方autotools编译方法是没有任何问题的;编译全带bitcode的库的时候我们不得不关闭汇编优化,在执行`./configure`阶段可以加上`--disable-asm`参数来禁用汇编.但是,这个选项在`configure`脚本中的实现机制有问题.导致其仍然调用了汇编的函数,但是汇编的代码却没有编译进去,从而会导致项目为真机构建和打包的链接阶段会爆出找不到符号的错误,这样就不能做到两全其美.出于轻微程度的强迫症影响,故把之前的`FFMPEG`和`libx264`项目的编译脚本进行了改进和打补丁.目前已经可以做到一键编译出带全部bitcode的FFMPEG和libx264的框架了.
 > `FFmpeg`需要依赖`libx264`.

 自动编译脚本项目位置放在github:
 [https://github.com/Diveinedu-CN/FFmpeg-iOS-build-script.git](https://github.com/Diveinedu-CN/FFmpeg-iOS-build-script.git)

由于时间和篇幅原因,关于其他更多详细的信息就不细细道来了.

戴维营教育Slogan: Dive in education!

更多iOS开发精品文章：[戴维营技术博客](http://io.diveinedu.com)









