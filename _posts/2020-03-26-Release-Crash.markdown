---
layout: post
title: 记录一次 debug 正常，release 崩溃的解决方案
date: 2020-03-26 13:32:24.000000000 +09:00
---
### 今天小伙伴给测试打包，发现一个异常情况，首页点击直接闪退。但是用真机和模拟器调试一切正常。
* 第一反应是，今天刚更新了xcode 11.4,首页的代码又没有改动，怀疑是apple的bug.
*  为了验证具体是哪里代码出问题，看了下bugly后台崩溃日志，以方便具体定位,bugly记录如下:
```c
0 libswiftCore.dylib	swift_getObjectType + 60
1 PicaDo	init (<compiler-generated>:0)
2 PicaDo	init (<compiler-generated>:0)
3 PicaDo	-[HomeComponentRoute control] (HomeComponentRoute.m:238)
4 PicaDo	-[HomeComponentRoute handModeCode:parmars:] (HomeComponentRoute.m:103)
5 PicaDo	-[HomeComponentRoute routeModeCode:parmars:] (HomeComponentRoute.m:46)
6 PicaDo	_$s6PicaDo25HomeNewComponentViewModelC05tableE10DidTapItem9component9indexPathyAA07ZYTableE0_p_10Foundation05IndexN0VtFTf4enn_nAA012ZYCollectionfE0C_Tg5Tm + 684
7 PicaDo	collectionView (<compiler-generated>:0)
8 PicaDo	collectionView (<compiler-generated>:0)
9 UIKitCore	0x00000001b609f000 + 2508024
10 UIKitCore	0x00000001b609f000 + 2668436
11 UIKitCore	0x00000001b609f000 + 10850772
12 UIKitCore	0x00000001b609f000 + 10851008
13 PicaDo	_$sSo6UIViewC6PicaDoE12touchesBegan_4withyShySo7UITouchCG_So7UIEventCSgtFToTm + 200
14 UIKitCore	0x00000001b609f000 + 10850772
15 UIKitCore	0x00000001b609f000 + 10851008
16 PicaDo	_$sSo6UIViewC6PicaDoE12touchesBegan_4withyShySo7UITouchCG_So7UIEventCSgtFToTm + 200
17 UIKitCore	0x00000001b609f000 + 10850772
18 UIKitCore	0x00000001b609f000 + 10851008
19 PicaDo	_$sSo6UIViewC6PicaDoE12touchesBegan_4withyShySo7UITouchCG_So7UIEventCSgtFToTm + 200
20 UIKitCore	0x00000001b609f000 + 6294532
21 UIKitCore	0x00000001b609f000 + 6287368
22 UIKitCore	0x00000001b609f000 + 6286804
23 UIKitCore	0x00000001b609f000 + 10914300
24 UIKitCore	0x00000001b609f000 + 10765984
25 UIKitCore	0x00000001b609f000 + 11292036
26 UIKitCore	0x00000001b609f000 + 11303100
27 UIKitCore	0x00000001b609f000 + 11270352
28 CoreFoundation	0x00000001b28f6000 + 691044
29 CoreFoundation	0x00000001b28f6000 + 690876
30 CoreFoundation	0x00000001b28f6000 + 688708
31 CoreFoundation	0x00000001b28f6000 + 668276
32 CoreFoundation	CFRunLoopRunSpecific + 424
33 GraphicsServices	GSEventRunModal + 160
34 UIKitCore	UIApplicationMain + 1932
35 PicaDo	main (main.m:14)
36 libdyld.dylib	0x00000001b281f000 + 6144
```
* 有了具体代码，那就找到对应代码位置去review下，看看到底是什么异常

```OC
- (GzWebViewControl *)control {
    if (!_control) {
        _web = [WKWebView new];
        _control= [[GzWebViewControl alloc] initWithWebView:_web delegate:nil];
        CommonWebHandle *commonHandle = [[CommonWebHandle alloc] initWithController:self];
        [_control registerHandler:commonHandle];
    }
    
    return _control;
}
```

> 没什么异常啊。这是什么问题呢？

* 首先，Debug模式下正常运行，所以，代码肯定是没有问题的，便排除了代码有问题的假设

* 其中，Debug通常称为调试版本，通过一系列编译选项的配合，编译的结果通常包含调试信息，而且不做任何优化，以为开发人员提供强大的应用程序调试能力。

* 而Release通常称为发布版本，是为用户使用的，一般客户不允许在发布版本上进行调试。所以不保存调试信息，同时，它往往进行了各种优化，以期达到代码最小和速度最优。为用户的使用提供便利。

* 也就是说，Debug模式和Release模式上的区别，就是在xcode开发环境里的编译选项的区别。在明确了这一点之后，就不得不回头重新思考自己的程序了。

* 于是又查各种资料，发现debug 与release 编译器优化的原因导致的

> 解决方案 ；Build Setting -> 搜索debug ,修改optimization Level ,release改为none.这个只是临时解决方案，看xcode更新或者代码修改才是最终解决方案。

* 这个编译策略呢，就是编译器的优化程度。早期因为硬件资源不够强大，编译器在编译过程中会对代码进行优化，提高代码的效率。不过这个优化过程，因为比较靠近硬件一层，所以可能导致一些不兼容的问题。

*  optimization Level 是什么？

> `-O0'

* None[-O0]: 不优化。在这种设置下， 编译器的目标是降低编译消耗，保证调试时输出期望的结果。程序的语句之间是独立的：如果在程序的停在某一行的断点出，我们可以给任何变量赋新值抑或是将程序计数器指向方法中的任何一个语句，并且能得到一个和源码完全一致的运行结果。

> `-O1'

* Fast[-O1]: 大函数所需的编译时间和内存消耗都会稍微增加。在这种设置下，编译器会尝试减小代码文件的大小，减少执行时间，但并不执行需要大量编译时间的优化。在苹果的编译器中，在优化过程中，严格别名，块重排和块间的调度都会被默认禁止掉。


> `-O2'

* Faster[-O2]: 编译器执行所有不涉及时间空间交换的所有的支持的优化选项。在这种设置下，编译器不会进行循环展开、函数内联或寄存器重命名。和‘Fast[-O1]’项相比，此设置会增加编译时间和生成代码的性能。


> `-O3'  

* Fastest[-O3]: 在开启‘Fast[-O1]’项支持的所有优化项的同时，开启函数内联和寄存器重命名选项。这个设置有可能会导致二进制文件变大。

> `-Os'

* Fastest, Smallest[-Os]: 优化大小。这个设置开启了‘Fast[-O1]’项中的所有不增加代码大小的优化选项，并会进一步的执行可以减小代码大小的优化。
 
* Fastest, Aggressive Optimizations[-Ofast]: 这个设置开启了“Fastest[-O3]”中的所有优化选项，同时也开启了可能会打破严格编译标准的积极优化，但并不会影响运行良好的代码。


[demo可以参考](https://github.com/wangkai598/C-review)