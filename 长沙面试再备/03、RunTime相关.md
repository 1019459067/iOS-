# 一、objc在向一个对象发送消息时，发生了什么？

objc在向一个对象发送消息时，runtime会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，如果一直到根类还没找到，转向拦截调用，走消息转发机制，一旦找到，就去执行它的实现IMP。

# 二、objc中向一个nil对象发送消息将会发生什么？

如果向一个nil对象发送消息，首先在寻找对象的isa指针时就是0地址返回了，所以不会出现任何错误。也不会崩溃。

>详解：

>如果一个方法返回值是一个对象，那么发送给nil的消息将返回0(nil)；

>如果方法返回值为指针类型，其指针大小为小于或者等于sizeof(void*) ，float，double，long double 或者long long的整型标量，发送给nil的消息将返回0；

>如果方法返回值为结构体,发送给nil的消息将返回0。结构体中各个字段的值将都是0；

>如果方法的返回值不是上述提到的几种情况，那么发送给nil的消息的返回值将是未定义的。

# 三、objc中向一个对象发送消息[obj foo]和objc_msgSend()函数之间有什么关系？

在objc编译时，[obj foo] 会被转意为：objc_msgSend(obj, @selector(foo));。

# 四、什么时候会报unrecognized selector的异常？

objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，如果，在最顶层的父类中依然找不到相应的方法时，会进入消息转发阶段，如果消息三次转发流程仍未实现，则程序在运行时会挂掉并抛出异常unrecognized selector sent to XXX 。

# 五、能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？

不能向编译后得到的类中增加实例变量；

能向运行时创建的类中添加实例变量；

1.因为编译后的类已经注册在 runtime 中,类结构体中的 objc_ivar_list  实例变量的链表和 instance_size 实例变量的内存大小已经确定，同时runtime会调用  class_setvarlayout 或 class_setWeaklvarLayout 来处理strong weak 引用.所以不能向存在的类中添加实例变量。
2.运行时创建的类是可以添加实例变量，调用class_addIvar函数. 但是的在调用 objc_allocateClassPair 之后，objc_registerClassPair 之前,原因同上.

# 六、给类添加一个属性后，在类结构体里哪些元素会发生变化？

instance_size ：实例的内存大小；objc_ivar_list *ivars:属性列表

# 六、一个objc对象的isa的指针指向什么？有什么作用？

指向他的类对象,从而可以找到对象上的方法

详解：下图很好的描述了对象，类，元类之间的关系:
![enter description here](https://upload-images.jianshu.io/upload_images/814874-255852b8fb05b05a.png)

图中实线是 super_class指针，虚线是isa指针。

1.Root class (class)其实就是NSObject，NSObject是没有超类的，所以Root class(class)的superclass指向nil。
2.每个Class都有一个isa指针指向唯一的Meta class
3.Root class(meta)的superclass指向Root class(class)，也就是NSObject，形成一个回路。
4.每个Meta class的isa指针都指向Root class (meta)。

# 七、[self class] 与 [super class]

下面的代码输出什么？

```   
@implementation Son : Father
- (id)init
{
   self = [super init];
   if (self) {
       NSLog(@"%@", NSStringFromClass([self class]));
       NSLog(@"%@", NSStringFromClass([super class]));
   }
   return self;
}
@end
```

NSStringFromClass([self class]) = Son
NSStringFromClass([super class]) = Son

详解：这个题目主要是考察关于 Objective-C 中对 self 和 super 的理解。

self 是类的隐藏参数，指向当前调用方法的这个类的实例；

super 本质是一个编译器标示符，和 self 是指向的同一个消息接受者。不同点在于：super 会告诉编译器，当调用方法时，去调用父类的方法，而不是本类中的方法。

当使用 self 调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；而当使用 super 时，则从父类的方法列表中开始找。然后调用父类的这个方法。

在调用[super class]的时候，runtime会去调用objc_msgSendSuper方法，而不是objc_msgSend；

``` 
OBJC_EXPORT void objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )

/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained Class class;
#else
    __unsafe_unretained Class super_class;
#endif
    /* super_class is the first class to search */
};
```

在objc_msgSendSuper方法中，第一个参数是一个objc_super的结构体，这个结构体里面有两个变量，一个是接收消息的receiver，一个是当前类的父类super_class。

objc_msgSendSuper的工作原理应该是这样的:
从objc_super结构体指向的superClass父类的方法列表开始查找selector，找到后以objc->receiver去调用父类的这个selector。注意，最后的调用者是objc->receiver，而不是super_class！

那么objc_msgSendSuper最后就转变成:

``` 
// 注意这里是从父类开始msgSend，而不是从本类开始
objc_msgSend(objc_super->receiver, @selector(class))

/// Specifies an instance of a class.  这是类的一个实例
    __unsafe_unretained id receiver;   


// 由于是实例调用，所以是减号方法
- (Class)class {
    return object_getClass(self);
}
```
由于找到了父类NSObject里面的class方法的IMP，又因为传入的入参objc_super->receiver = self。self就是son，调用class，所以父类的方法class执行IMP之后，输出还是son，最后输出两个都一样，都是输出son。

# 八、runtime如何通过selector找到对应的IMP地址？

每一个类对象中都一个方法列表,方法列表中记录着方法的名称,方法实现,以及参数类型,其实selector本质就是方法名称,通过这个方法名称就可以在方法列表中找到对应的方法实现.

# 九、_objc_msgForward函数是做什么的，直接调用它将会发生什么？

_objc_msgForward是 IMP 类型，用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，_objc_msgForward会尝试做消息转发。

详解：_objc_msgForward在进行消息转发的过程中会涉及以下这几个方法：

 1. List itemresolveInstanceMethod:方法 (或resolveClassMethod:)。
 2. List itemforwardingTargetForSelector:方法
 3. List itemmethodSignatureForSelector:方法
 4. List itemforwardInvocation:方法
 5. List itemdoesNotRecognizeSelector: 方法

具体请见：请看Runtime在工作中的运用 第三章Runtime方法调用流程；

# 十、runtime如何实现weak变量的自动置nil？知道SideTable吗？

> runtime 对注册的类会进行布局，对于 weak 修饰的对象会放入一个 hash 表中。 用 weak 指向的对象内存地址作为key，当此对象的引用计数为0的时候会 dealloc，假如 weak 指向的对象内存地址是a，那么就会以a为键， 在这个 weak表中搜索，找到所有以a为键的 weak 对象，从而设置为 nil。

**更细一点的回答：**

1.初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。
2.添加引用时：objc_initWeak函数会调用objc_storeWeak() 函数， objc_storeWeak()的作用是更新指针指向，创建对应的弱引用表。
3.释放时,调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。

SideTable结构体是负责管理类的引用计数表和weak表，

详解：参考自《Objective-C高级编程》一书
**1.初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。**

``` 
{
    NSObject *obj = [[NSObject alloc] init];
    id __weak obj1 = obj;
}
```

当我们初始化一个weak变量时，runtime会调用 NSObject.mm 中的objc_initWeak函数。

``` 
// 编译器的模拟代码
 id obj1;
 objc_initWeak(&obj1, obj);
/*obj引用计数变为0，变量作用域结束*/
 objc_destroyWeak(&obj1);
```
通过objc_initWeak函数初始化“附有weak修饰符的变量（obj1）”，在变量作用域结束时通过objc_destoryWeak函数释放该变量（obj1）。

**2.添加引用时：objc_initWeak函数会调用objc_storeWeak() 函数， objc_storeWeak()的作用是更新指针指向，创建对应的弱引用表。**

objc_initWeak函数将“附有weak修饰符的变量（obj1）”初始化为0（nil）后，会将“赋值对象”（obj）作为参数，调用objc_storeWeak函数。

``` 
obj1 = 0；
obj_storeWeak(&obj1, obj);
```

**也就是说：**

weak 修饰的指针默认值是 nil （在Objective-C中向nil发送消息是安全的）

然后obj_destroyWeak函数将0（nil）作为参数，调用objc_storeWeak函数。

``` 
objc_storeWeak(&obj1, 0);
```
前面的源代码与下列源代码相同。

``` 
// 编译器的模拟代码
id obj1;
obj1 = 0;
objc_storeWeak(&obj1, obj);
/* ... obj的引用计数变为0，被置nil ... */
objc_storeWeak(&obj1, 0);
```
objc_storeWeak函数把第二个参数的赋值对象（obj）的内存地址作为键值，将第一个参数__weak修饰的属性变量（obj1）的内存地址注册到 weak 表中。如果第二个参数（obj）为0（nil），那么把变量（obj1）的地址从weak表中删除。

由于一个对象可同时赋值给多个附有__weak修饰符的变量中，所以对于一个键值，可注册多个变量的地址。

可以把objc_storeWeak(&a, b)理解为：objc_storeWeak(value, key)，并且当key变nil，将value置nil。在b非nil时，a和b指向同一个内存地址，在b变nil时，a变nil。此时向a发送消息不会崩溃：在Objective-C中向nil发送消息是安全的。

**3.释放时,调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。**

当weak引用指向的对象被释放时，又是如何去处理weak指针的呢？当释放对象时，其基本流程如下：

1.调用objc_release
2.因为对象的引用计数为0，所以执行dealloc
3.在dealloc中，调用了_objc_rootDealloc函数
4.在_objc_rootDealloc中，调用了object_dispose函数
5.调用objc_destructInstance
6.最后调用objc_clear_deallocating

对象被释放时调用的objc_clear_deallocating函数:

1.从weak表中获取废弃对象的地址为键值的记录
2.将包含在记录中的所有附有 weak修饰符变量的地址，赋值为nil
3.将weak表中该记录删除
4.从引用计数表中删除废弃对象的地址为键值的记录

**总结:**

其实Weak表是一个hash（哈希）表，Key是weak所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

# 十一、isKindOfClass 与 isMemberOfClass

下面代码输出什么？

```
@interface Sark : NSObject
@end

@implementation Sark
@end
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
        BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
        BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
        BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
        NSLog(@"%d %d %d %d", res1, res2, res3, res4);
    }
    return 0;
}
```

答案：1 0 0 0

**详解：**

在isKindOfClass中有一个循环，先判断class是否等于meta class，不等就继续循环判断是否等于meta class的super class，不等再继续取super class，如此循环下去。

[NSObject class]执行完之后调用isKindOfClass，第一次判断先判断NSObject和 NSObject的meta class是否相等，之前讲到meta class的时候放了一张很详细的图，从图上我们也可以看出，NSObject的meta class与本身不等。接着第二次循环判断NSObject与meta class的superclass是否相等。还是从那张图上面我们可以看到：Root class(meta) 的superclass就是 Root
class(class)，也就是NSObject本身。所以第二次循环相等，于是第一行res1输出应该为YES。

同理，[Sark class]执行完之后调用isKindOfClass，第一次for循环，Sark的Meta Class与[Sark class]不等，第二次for循环，Sark Meta Class的super class 指向的是 NSObject Meta Class， 和Sark Class不相等。第三次for循环，NSObject Meta Class的super class指向的是NSObject Class，和 Sark Class 不相等。第四次循环，NSObject Class 的super class 指向 nil， 和 Sark Class不相等。第四次循环之后，退出循环，所以第三行的res3输出为NO。

isMemberOfClass的源码实现是拿到自己的isa指针和自己比较，是否相等。

第二行isa 指向 NSObject 的 Meta Class，所以和 NSObject Class不相等。第四行，isa指向Sark的Meta Class，和Sark Class也不等，所以第二行res2和第四行res4都输出NO。

# 十二、使用runtime Associate方法关联的对象，需要在主对象dealloc的时候释放么？

无论在MRC下还是ARC下均不需要，被关联的对象在生命周期内要比对象本身释放的晚很多，它们会在被 NSObject -dealloc 调用的object_dispose()方法中释放。

**详解：**

``` 
1、调用 -release ：引用计数变为零
对象正在被销毁，生命周期即将结束. 
不能再有新的 __weak 弱引用，否则将指向 nil.
调用 [self dealloc]

2、 父类调用 -dealloc 
继承关系中最直接继承的父类再调用 -dealloc 
如果是 MRC 代码 则会手动释放实例变量们（iVars）
继承关系中每一层的父类 都再调用 -dealloc

3、NSObject 调 -dealloc 
只做一件事：调用 Objective-C runtime 中object_dispose() 方法

4. 调用 object_dispose()
为 C++ 的实例变量们（iVars）调用 destructors
为 ARC 状态下的 实例变量们（iVars） 调用 -release 
解除所有使用 runtime Associate方法关联的对象 
解除所有 __weak 引用 
调用 free()
```

# 十二、什么是method swizzling（俗称黑魔法)

简单说就是进行方法交换

在Objective-C中调用一个方法，其实是向一个对象发送消息，查找消息的唯一依据是selector的名字。利用Objective-C的动态特性，可以实现在运行时偷换selector对应的方法实现，达到给方法挂钩的目的。
每个类都有一个方法列表，存放着方法的名字和方法实现的映射关系，selector的本质其实就是方法名，IMP有点类似函数指针，指向具体的Method实现，通过selector就可以找到对应的IMP。
换方法的几种实现方式

 - 利用 method_exchangeImplementations 交换两个方法的实现 
 - 利用 class_replaceMethod替换方法的实现 
 - 利用 method_setImplementation 来直接设置某个方法的IMP

# 十三、实例对象的数据结构？

具体可以参看 `Runtime` 源代码，在文件 `objc-private.h` 的第 `127-232` 行。

```
struct objc_object {
	isa_t isa;
	//...
}
```

本质上 `objc_object` 的私有属性只有一个 `isa` 指针。指向 `类对象` 的内存地址。

# 十四、类对象的数据结构？

具体可以参看 `Runtime` 源代码。

> 类对象就是 `objc_class`。

```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass; //父类指针
    cache_t cache;             // formerly cache pointer and vtable 方法缓存
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags 用于获取地址

    class_rw_t *data() { 
        return bits.data(); // &FAST_DATA_MASK 获取地址值
    }
```

它的结构相对丰富一些。继承自`objc_object`结构体，所以包含`isa`指针

- `isa`：指向元类
- `superClass`: 指向父类
- `Cache`: 方法的缓存列表
- `data`: 顾名思义，就是数据。是一个被封装好的 `class_rw_t` 。

# 十五、元类对象的数据结构?

`元类对象` 和 `类对象` 数据结构是一致的的参见第二题。

# 十六、`Category` 的实现原理？

> 被添加在了 `class_rw_t` 的对应结构里。

`Category` 实际上是 `Category_t` 的结构体，在运行时，新添加的方法，都被以倒序插入到原有方法列表的最前面，所以不同的`Category`，添加了同一个方法，执行的实际上是最后一个。

拿方法列表举例，实际上是一个二维的数组。


`Category` 如果翻看源码的话就会知道实际上是一个 `_catrgory_t` 的结构体。

--
例如我们在程序中写了一个 `Nsobject+Tools` 的分类，那么被编译为 `C++` 之后，实际上是：


```c++
static struct _catrgory_t _OBJC_$_CATEGORY_NSObject_$_Tools __attribute__ ((used,section),("__DATA,__objc__const"))
{
    // name
    // class
    // instance method list
    // class method list
    // protocol list
    // properties
}
```


`Category` 在刚刚编译完的时候，和原来的类是分开的，只有在程序运行起来后，通过 `Runtime` ，`Category` 和原来的类才会合并到一起。

`mememove`，`memcpy`：这俩方法是位移、复制，简单理解就是原有的方法移动到最后，根根新开辟的控件，把前面的位置留给分类，然后分类中的方法，按照倒序依次插入，可以得出的结论就就是，越晚参与编译的分类，里面的方法才是生效的那个。

# 十七、如何给 `Category` 添加属性？关联对象以什么形式进行存储？

查看的是 `关联对象` 的知识点。

详细的说一下 `关联对象`。

`关联对象` 以哈希表的格式，存储在一个全局的单例中。



```objc
@interface NSObject (Extension)

@property (nonatomic,copy  ) NSString *name;

@end


@implementation NSObject (Extension)

- (void)setName:(NSString *)name {
    
    objc_setAssociatedObject(self, @selector(name), name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}


- (NSString *)name {
    
    return objc_getAssociatedObject(self,@selector(name));
}

@end
```

# 十八、`Category` 有哪些用途？

- 给系统类添加方法、属性（需要关联对象）。
- 对某个类大量的方法，可以实现按照不同的名称归类。

# 十九、`Category` 和 `Extension` 有什么区别

* 1.`分类` 的加载在 `运行时`，`类拓展` 的加载在 `编译时`。
* 2.`类拓展` 不能给系统的类添加方法。
* 3.`类拓展` 只以声明的形式存在，一般存在 .m 文件中。

# 二十、说一下 `Method Swizzling`? 说一下在实际开发中你在什么场景下使用过?

交换方法。

可以结合 `Aspect` 的使用一起说。

其实方法交换，应用于统计个页面停留时长，记录用户路径，Hook系统方法啥的。。。还有不少用途

# 二十一、如何实现动态添加方法和属性？

直接查 API 即可


# 二十二、说一下对 `isa` 指针的理解， 对象的`isa` 指针指向哪里？`isa` 指针有哪两种类型？

`isa` 等价于 `is kind of`

- 实例对象 `isa` 指向类对象
- 类对象指 `isa` 向元类对象
- 元类对象的 `isa` 指向元类的基类


`isa` 有两种类型
- 纯指针，指向内存地址
- `NON_POINTER_ISA`，除了内存地址，还存有一些其他信息


## isa源码分析

在Runtime源码查看isa_t是共用体。简化结构如下：

```objc
union isa_t 
{
    Class cls;
    uintptr_t bits;
    # if __arm64__ // arm64架构
#   define ISA_MASK        0x0000000ffffffff8ULL //用来取出33位内存地址使用（&）操作
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1; //0：代表普通指针，1：表示优化过的，可以存储更多信息。
        uintptr_t has_assoc         : 1; //是否设置过关联对象。如果没设置过，释放会更快
        uintptr_t has_cxx_dtor      : 1; //是否有C++的析构函数
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 内存地址值
        uintptr_t magic             : 6; //用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1; //是否有被弱引用指向过
        uintptr_t deallocating      : 1; //是否正在释放
        uintptr_t has_sidetable_rc  : 1; //引用计数器是否过大无法存储在ISA中。如果为1，那么引用计数会存储在一个叫做SideTable的类的属性中
        uintptr_t extra_rc          : 19; //里面存储的值是引用计数器减1

#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__ // arm86架构,模拟器是arm86
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };

# else
#   error unknown architecture for packed isa
# endif

}
```

> 注意：什么是位域？

# 二十三、`Obj-C` 中的类信息存放在哪里？

类方法存储在元类。

- 对象方法、属性、成员变量、协议等存放在 Class 对象中。
- 类方法存放在 meta-class 对象中。
- 成员变量的具体指，存放在 instance 对象中。

# 二十四、一个 `NSObject` 对象占用多少内存空间？

受限于内存分配的机制，一个 `NSObject`对象都会分配 `16Bit` 的内存空间。

但是实际上在 64位 下，只使用了 `8bit`;
在32位下，只使用了 `4bit`

一个 NSObject 实例对象成员变量所占的大小，实际上是 8KB
```objc
#import <Objc/Runtime>
Class_getInstanceSize([NSObject Class])
```

本质是
```objc
size_t class_getInstanceSize(Class cls)
{
    if (!cls) return 0;
    return cls->alignedInstanceSize();
}
```

获取 Obj-C 指针所指向的内存的大小，实际上是16KB
```objc
#import <malloc/malloc.h>
malloc_size((__bridge const void *)obj); 
```


对象在分配内存空间时，会进行内存对齐，所以在 iOS 中，分配内存空间都是 16字节 的倍数。



可以通过以下网址 ：openSource.apple.com/tarballs 来查看源代码。

# 二十四、说一下对 `class_ro_t` 的理解？

存储了当前类在编译期就已经确定的属性、方法以及遵循的协议。


```objc
struct class_ro_t {  
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;

    const uint8_t * ivarLayout;

    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
```

`baseMethodList`，`baseProtocols`，`ivars`，`baseProperties`三个都是以为数组。


# 二十五、说一下对 `class_rw_t` 的理解？

`rw`代表可读可写。

`ObjC` 类中的属性、方法还有遵循的协议等信息都保存在 `class_rw_t` 中：

```objc
// 可读可写
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro; // 指向只读的结构体,存放类初始信息

    /*
     这三个都是二位数组，是可读可写的，包含了类的初始内容、分类的内容。
     methods中，存储 method_list_t ----> method_t
     二维数组，method_list_t --> method_t
     这三个二位数组中的数据有一部分是从class_ro_t中合并过来的。
     */
    method_array_t methods; // 方法列表（类对象存放对象方法，元类对象存放类方法）
    property_array_t properties; // 属性列表
    protocol_array_t protocols; //协议列表

    Class firstSubclass;
    Class nextSiblingClass;
    
    //...
    }
```

# 二十六、分类和类拓展的区别?

* 1.`分类` 的加载在 `运行时`，`类拓展` 的加载在 `编译时`。

* 2.`分类` 不能给系统的类添加方法。
* 3.`类拓展` 只以声明的形式存在，一般存在 .m 文件中。

# 二十七、如何运用 `Runtime` 字典转模型？

`Runtime` 遍历 `ivar_list`,结合 `KVC` 赋值。

# 二十八、如何运用 `Runtime` 进行模型的归解档

`Runtime` 遍历 `ivar_list`。

# 二十九、在 `Obj-C` 中为什么叫发消息而不叫函数调用？

结合 `objc_msgsend` 讲一下，接收消息的过程。

# 三十、说一下 `Runtime` 的方法缓存？存储的形式、数据结构以及查找的过程？

`cache_t`增量扩展的哈希表结构。哈希表内部存储的 `bucket_t`。

`bucket_t` 中存储的是 `SEL` 和 `IMP`的键值对。

- 如果是有序方法列表，采用二分查找

- 如果是无序方法列表，直接遍历查找

### cache_t结构体

```objc
// 缓存曾经调用过的方法，提高查找速率
struct cache_t {
    struct bucket_t *_buckets; // 散列表
    mask_t _mask; //散列表的长度 - 1
    mask_t _occupied; // 已经缓存的方法数量，散列表的长度使大于已经缓存的数量的。
    //...
}
```

```objc
struct bucket_t {
    cache_key_t _key; //SEL作为Key @selector()
    IMP _imp; // 函数的内存地址
	//...
}
```

散列表查找过程，在`objc-cache.mm`文件中

```objc
// 查询散列表，k
bucket_t * cache_t::find(cache_key_t k, id receiver)
{
    assert(k != 0); // 断言

    bucket_t *b = buckets(); // 获取散列表
    mask_t m = mask(); // 散列表长度 - 1
    mask_t begin = cache_hash(k, m); // & 操作
    mask_t i = begin; // 索引值
    do {
        if (b[i].key() == 0  ||  b[i].key() == k) {
            return &b[i];
        }
    } while ((i = cache_next(i, m)) != begin);
    // i 的值最大等于mask,最小等于0。

    // hack
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)k, cls);
}
```
上面是查询散列表函数，其中`cache_hash(k, m)`是静态内联方法，将传入的`key`和`mask`进行`&`操作返回`uint32_t`索引值。`do-while`循环查找过程，当发生冲突`cache_next`方法将索引值减1。

# 三十一、是否了解 `Type Encoding`?

不懂的可以看文档，中文名 - `类型编码`。

https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html

# 三十二、`Objective-C` 如何实现多重继承？

`Objective-C` 只支持多层继承，不支持多重继承。想实现的话，可以使用协议。

# 三十三、`Category` 可不可以添加实例对象？为什么？

答案是不可以。   
Category 的结构体内部没有容纳 Ivar 的数据结构。

有很详细的分析，具体可以看[这篇文章](https://www.jianshu.com/p/dcc3284b65bf)

# 三十四、`Obj-c`对象、类的本质是通过什么数据结构实现的？

结构体。

譬如一个最常见的 `Nsobject` 对象。将其编译为 `C++` 代码后，实际上就是一个 `Nsobject_Impl` 结构体。

# 三十五、`Category` 在编译过后，是在什么时机与原有的类合并到一起的？ 

1. 程序启动后，通过编译之后，Runtime 会进行初始化，调用 `_objc_init`。
2. 然后会 `map_images`。
3. 接下来调用 `map_images_nolock`。
4. 再然后就是 `read_images`，这个方法会读取所有的类的相关信息。
5. 最后是调用 `reMethodizeClass:`，这个方法是重新方法化的意思。
6. 在 `reMethodizeClass:` 方法内部会调用 **`attachCategories:`** ，这个方法会传入 Class 和 Category ，会将方法列表，协议列表等与原有的类合并。最后加入到 **`class_rw_t`** 结构体中。

# 题目一：下面的代码输出什么？

```
@implementation Son : Father
- (id)init {
    self = [super init];
    if (self) {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```

**结果：** Son / Son

**分析：**

对于上面的答案，第一个的结果应该是我们的预期结果，但是第二个结果却让我们很费解了。

那我们利用前面文章讲过的知识点来分析一下整个的流程。

因为，Son 及 Father 都没有实现 -(Class)calss 方法，所以这里所有的调用最终都会找到基类 NSObject 中，并且在其中找到 -(Class)calss 方法。那我们需要了解的就是在 NSObject 中这个方法的实现了。

在 NSObject.mm 中可以找到 -(Class)class 的实现：

```
- (Class)class {
    return object_getClass(self);
}
```

在 objc_class.mm 中找到 object_getClass 的实现：

```
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
```

> ps：上面的方法定义可以去[官方OpenSource](https://link.juejin.im?target=https%3A%2F%2Fopensource.apple.com%2Fsource%2Fobjc4%2F)中下载源码哦。

可以看到，最终这个方法返回的是，调用这个方法的 objc 的 isa 指针。那我们只需要知道在题干中的代码里面最终是谁在调用 -(Class)class 方法就可以找到答案了。

接下来，我们利用 **clang -rewrite-objc** 命令，将题干的代码转化为如下代码：

```
NSLog((NSString *)&__NSConstantStringImpl__var_folders_8k_cgm28r0d0bz94xnnrr606rf40000gn_T_Car_3f2069_mi_0, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("class"))));
NSLog((NSString *)&__NSConstantStringImpl__var_folders_8k_cgm28r0d0bz94xnnrr606rf40000gn_T_Car_3f2069_mi_1, NSStringFromClass(((Class (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Car"))}, sel_registerName("class"))));
```

从上方可以得出，调用 [Father class] 的时候，本质是在调用

```
objc_msgSendSuper(struct objc_super *super, SEL op, ...)
```

struct objc_super 的定义如下：

```
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};
```

从定义可以得知：当利用 super 调用方法时，只要编译器看到super这个标志，就会让当前对象去调用父类方法，本质还是当前对象在调用，是去父类找实现，super 仅仅是一个编译指示器。但是消息的接收者 receiver 依然是self。最终在 NSObject 获取 isa 指针的时候，获取到的依旧是 self 的 isa，所以，我们得到的结果是：Son。

**扩展一下：** 看看下方的代码会输出什么？

```
@interface Father : NSObject
@end

@implementation Father

- (Class)class {
    return [Father class];
}

@end

---

@interface Son : Father
@end

@implementation Son

- (id)init {
    self = [super init];
    if (self) {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}

@end

int main(int argc, const char * argv[]) {
    Son *foo = [[Son alloc]init];
    return 0;
}

---输出：---
Father
Father
```
* * *

# 题目二：以下的代码会输出什么结果？

```
@interface Sark : NSObject
@end
@implementation Sark
@end

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...

        NSLog(@"%@", [NSObject class]);
        NSLog(@"%@", [Sark class]);

        BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
        BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
        BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
        BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
        NSLog(@"%d--%d--%d--%d", res1, res2, res3, res4);
    }
    return 0;
}
```

**结果：** 1--0--0--0

**分析：**

首先，我们先去查看一下题干中两个方法的源码：

```
- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}
```

可以得知：

*   isKindOfClass 的执行过程是拿到自己的 isa 指针和自己比较，若不等则继续取 isa 指针所指的 super class 进行比较。如此循环。
*   isMemberOfClass 是拿到自己的 isa 指针和自己比较，是否相等。

1.  [NSObject class] 执行完之后调用 isKindOfClass，第一次判断先判断 NSObject 和 NSObject 的 meta class 是否相等，之前讲到 meta class 的时候放了一张很详细的图，从图上我们也可以看出，NSObject 的 meta class 与本身不等。接着第二次循环判断 NSObject 与meta class 的 superclass 是否相等。还是从那张图上面我们可以看到：Root class(meta) 的 superclass 就是 Root class(class)，也就是 NSObject 本身。所以第二次循环相等，于是第一行 res1 输出应该为YES。

2.  isa 指向 NSObject 的 Meta Class，所以和 NSObject Class不相等。

3.  [Sark class] 执行完之后调用 isKindOfClass，第一次 for 循环，Sark 的 Meta Class 与 [Sark class] 不等，第二次 for 循环，Sark Meta Class 的 super class 指向的是 NSObject Meta Class， 和 Sark Class 不相等。第三次 for 循环，NSObject Meta Class 的 super class 指向的是 NSObject Class，和 Sark Class 不相等。第四次循环，NSObject Class 的super class 指向 nil， 和 Sark Class 不相等。第四次循环之后，退出循环，所以第三行的 res3 输出为 NO。

4.  isa 指向 Sark 的 Meta Class，和 Sark Class 也不等。