![](https://upload-images.jianshu.io/upload_images/24396273-45185a538ffbdeec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 一、HOOK概述

#### 1.HOOK定义

`HOOK`翻译成中文为“挂钩”、“钩子”，在iOS逆向领域中指的是改变程序运行流程的一种技术，通过`HOOK`可以让别人的程序执行自己所写的代码

下列示意图就是对HOOK功能的形象诠释：

*   注入恶意代码让用户误以为打卡成功，实际并没有完成打卡（只修改中间流程）
*   注入恶意代码让用户误以为网络出问题（开辟新的流程分支）![](https://upload-images.jianshu.io/upload_images/24396273-291b083b36389b3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2.HOOK方式

在iOS中HOOK技术有以下几种：

*   Method Swizzling：利用OC的Runtime特性，动态改变`SEL（方法编号）`和`IMP（方法实现）`的对应关系，达到OC方法调用流程改变的目的
*   fishhook：这是FaceBook提供的一个动态修改链接machO文件的工具，利用machO文件加载原理，通过修改懒加载和非懒加载两个表的指针达到C函数HOOK的目的
*   Cydia Substrate：原名为`Mobile Substrate`，它的主要作用是针对OC方法、C函数以及函数地址进行HOOK操作，且安卓也能使用

> 之前已经介绍过`Method Swizzling`了，OC的Runtime特性让它有了“黑魔法”之称，同时也是局限性所在

三者的区别如下：

*   `Method Swizzling`只适用于动态的OC方法（运行时确定函数实现地址）
*   `fishhook`适用于静态的C方法（编译时确定函数实现地址）
*   `Cydia Substrate`是一种强大的框架，只需要通过Logos语言（类似于正向开发）就可以进行Hook，适用于OC方法、C函数以及函数地址

## 二、fishhook

#### 1.fishhook的使用


整个fishhook就开放了一个结构体`rebinding`和两个函数

*   `rebind_symbols`hook项目中的所有函数名称
*   `rebind_symbols_image`只hook某一个资源库的函数名称![](https://upload-images.jianshu.io/upload_images/24396273-d3d3d0981e93adc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


使用fishhook的步骤：

1.  导入头文件

```
#import "fishhook.h"

```

2.  指定交换函数——`rebinding`结构体对象

```
struct rebinding nslog;
nslog.name = "NSLog";
nslog.replacement = fxNSlog;
nslog.replaced = (void *)&sys_nslog;
```

3.  调用`rebind_symbols`重新绑定符号

```
struct rebinding rebs[] = {nslog};
rebind_symbols(rebs, 1);
```

完整代码如下：

```
#import "fishhook.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    struct rebinding nslog;
    nslog.name = "NSLog";
    nslog.replacement = fxNSlog;
    nslog.replaced = (void *)&sys_nslog;

    struct rebinding rebs[] = {nslog};
    rebind_symbols(rebs, 1);
}

static void(*sys_nslog)(NSString *format, ...);

void fxNSlog(NSString *format, ...) {
    format = [format stringByAppendingFormat:@"已hook"];
    sys_nslog(format);
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"点击屏幕");
}

@end
```

*   定义`rebinding`来指定交换的函数
    *   `name`中指定Hook的函数名称（C字符串）
    *   `replacement`中指定新的函数地址
        *   c函数的名称就是函数指针——静态语言编译时就已确定
    *   `replaced`中存放原始函数地址的指针
        *   由于`NSLog`在Foundation库中，在每个手机的内存地址都不一样，不能通过`CMD+B`就能确定其内存地址
        *   这里fishhook在运行的时刻会动态获取到`NSLog`的地址，所以这里定义新的函数指针保存函数实现
        *   使用`&`修改变量内部的值
        *   使用`(void *)`进行类型转换
*   `rebind_symbols`中需要两个参数
    *   参数一是`rebinding`数组
    *   参数二是`rebinding`数组的长度

作为一个开发者，有一个学习的氛围跟一个交流圈子特别重要，这有个iOS交流群：[642363427](https://links.jianshu.com/go?to=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%253A%2F%2Flinks.jianshu.com%2Fgo%253Fto%253Dhttps%25253A%25252F%25252Flink.zhihu.com%25252F%25253Ftarget%25253Dhttps%2525253A%25252F%25252Flinks.jianshu.com%25252Fgo%2525253Fto%2525253Dhttps%252525253A%252525252F%252525252Fjq.qq.com%252525252F%252525253F_wv%252525253D1027%2525252526k%252525253DbESXgU0E)，不管你是小白还是大牛欢迎入驻 ，分享BAT,阿里面试题、面试经验，讨论技术，iOS开发者一起交流学习成长！

#### 2.fishhook的局限性

前面介绍了`fishhook`是用来hook C函数的，但在此前提下还是有它的局限性——无法交换自定义函数

*   `C函数`是静态的，在编译时就已经确定了函数地址（函数实现地址在MachO本地文件中）
*   `系统C函数`则是存在着动态的部分

> 那么为什么系统级别的C函数就可以呢？这就要说到PIC技术了

#### 3.PIC技术

`PIC技术`（Position-independent code），又叫做`位置独立代码`，是为了`系统C函数`在编译时期能够确认一个地址的一种技术手段

*   编译时在MachO文件中预留出一段空间——符号表（DATA段中）
*   dyld把应用加载到内存中时（此时在`Load Commands`中会依赖`Foundation`），在符号表中找到了`NSLog`函数，就会进行链接绑定——将`Foundation`中`NSLog`的真实地址赋值到`DATA段`的`NSLog`符号上

> 而自定义的C函数不会生成符号表，直接就是一个函数地址，所以fishhook的局限性就在于只有符号表内的符号可以hook（重新绑定符号）

#### 4.LLDB调试

因为`NSLog`是个懒加载的符号，所以加了行代言代码

1.  通过`image list`打印所有镜像资源——第一个就是进程的内存首地址![](https://upload-images.jianshu.io/upload_images/24396273-291ff6a9c62f8438.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


2.  获取到MachO中NSLog的内存偏移量![](https://upload-images.jianshu.io/upload_images/24396273-4c9ab72e6fb3bc88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


    使用计算器，内存首地址+内存偏移量=符号地址
3.  通过`memory read 内存地址`读出符号地址![](https://upload-images.jianshu.io/upload_images/24396273-fa67a22fbc28641f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


    由于iOS是小端模式，内存地址8位倒着读
4.  通过`dis -s 内存地址`查看汇编![](https://upload-images.jianshu.io/upload_images/24396273-6db4f180d8a1e78d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


    此时的符号并没有绑定，因为还没有使用
5.  过掉断点（使用NSLog）再进行查看![](https://upload-images.jianshu.io/upload_images/24396273-b3b5cb0a2b326ae5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


    此时的NSLog符号已经被绑定
6.  再跳到下一步（重绑定符号）进行查看![](https://upload-images.jianshu.io/upload_images/24396273-2ff7295b4f767642.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


    此时的NSLog符号已经被重绑定了

#### 5.fishhook原理探索

![](https://upload-images.jianshu.io/upload_images/24396273-a33fe25598401906.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](https://upload-images.jianshu.io/upload_images/24396273-8741ecd3b525a6a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来分析下`rebind_symbols`：

*   `prepend_rebindings`将`rebindings`数组不断添加到`_rebindings_head`链表的头部成为新的头节点
*   判断链表`_rebindings_head->next`是否为空来判断是否是第一次调用
    *   若首次调用则使用`_dyld_register_func_for_add_image`进行回调监听
    *   若不是则遍历调用`_rebind_symbols_for_image`
*   这个函数有个Int返回值
    *   如果重定向成功则返回 **0**
    *   如果失败则返回 **-1**

接下来是`_rebind_symbols_for_image`对MachO文件的操作：按照规则计算各种表的地址和指针在表中的偏移量

![](https://upload-images.jianshu.io/upload_images/24396273-75a487dc606fc125.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](https://upload-images.jianshu.io/upload_images/24396273-859f708971c67c9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


最后一步，`perform_rebinding_with_section`根据算好的符号表地址和偏移量，找到在符号表中用于指向共享库目标函数的指针，将该指针的值（即目标函数的地址）赋值给`rebinding`的`*replaced`，最后修改该指针的值为`replacement`（新的函数地址）

![](https://upload-images.jianshu.io/upload_images/24396273-88ea930c33f89354.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 6.fishhook原理的实践——在MachO中查找函数实现

接下来就根据字符串对应在符号表中的指针，找到其在共享库的函数实现的

1.  NSLog在`Lazy Symbol Pointers`中位列第一个，下标为0![](https://upload-images.jianshu.io/upload_images/24396273-45c935a004ecfbfd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


2.  `Dynamic System Table->Indirect Symbols`中的value与`Lazy Symbol Pointers`一一对应，此表中的Data(0xA9=169)即为另一个表的下标![](https://upload-images.jianshu.io/upload_images/24396273-fbbd799040132241.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


3.  拿着这个下标在`Symbol Table->Symbols`中找到NSLog，此时的Data对应`String Table Index`中的偏移值(0x0000009B)![](https://upload-images.jianshu.io/upload_images/24396273-2df160bd8f25e9b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


4.  最后在`String Table`中计算表头(0x000061EC)+偏移量(0x0000009B)，找到NSLog的函数地址![](https://upload-images.jianshu.io/upload_images/24396273-99ddc7bf53b594b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 7.fishhook应用场景

既然`fishhook`可以交换C函数，那么就可以交换`method_exchangeImplementations`等函数来防止HOOK，项目本身如果需要使用`method_exchangeImplementations`则可以使用新的函数地址来进行操作

*   攻：可以插入新的动态库提前进行HOOK方法，此时fishhook的防护就不起作用了
*   防：也可以提前插入动态库进行fishhook交换（逆向注入的动态库一般排在原项目的动态库之后），此时fishhook代码先于hook代码，所以还是fishhook还是生效的
*   攻：釜底抽薪——直接找到fishhook的函数地址进行修改，再次运行
*   ...

## 写在后面

在安全攻防领域中，hook原理起到至关重要的作用，而使用fishhook做防护虽然意义不大，但是了解fishhook原理对后续的学习还是挺有帮助的，想了解更多逆向相关知识不妨动动小手，添加一下咱们的交流群[642363427](https://links.jianshu.com/go?to=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%253A%2F%2Fjq.qq.com%2F%253F_wv%253D1027%2526k%253DDdPShmie)来为你的技术多添一份光彩。


