## 1.基础语法篇

### 1、 在main执行之前和之后执行的代码可能是什么？

**1.1 main函数执行之前**，主要就是初始化系统相关资源：

- 设置栈指针
- 初始化静态`static`变量和`global`全局变量，即`.data`段的内容
- 将未初始化部分的全局变量赋初值：数值型`short`，`int`，`long`等为`0`，`bool`为`FALSE`，指针为`NULL`等等，即`.bss`段的内容
- 全局对象初始化，在`main`之前调用构造函数，这是可能会执行前的一些代码
- 将main函数的参数`argc`，`argv`等传递给`main`函数，然后才真正运行`main`函数
- `__attribute__((constructor))`(https://www.jianshu.com/p/965f6f903114, https://zhuanlan.zhihu.com/p/474790212, https://zhuanlan.zhihu.com/p/599034039)

**1.2 main函数执行之后**：

- 全局对象的析构函数会在main函数之后执行；
- 可以用 **atexit** 注册一个函数，它会在main 之后执行; (https://blog.csdn.net/weixin_43781183/article/details/105759821, )
- `__attribute__((destructor))`





## 2、结构体内存对齐问题？

- 结构体内成员按照声明顺序存储，第一个成员地址和整个结构体地址相同。
- 未特殊说明时，按结构体中size最大的成员对齐（若有double成员，按8字节对齐。）

​      c++11以后引入两个关键字 **alignas**与 **alignof** 。其中alignof可以计算出类型的对齐方式，alignas可以指定结构体的对齐方式。
​       但是alignas在某些情况下是不能使用的(https://blog.51cto.com/zhangxinbei/1743179?articleABtest=0

https://cloud.tencent.com/developer/section/1009001, https://zhuanlan.zhihu.com/p/417061548)，具体见下面的例子:

```C++
// alignas 生效的情况

struct Info {
  uint8_t a;
  uint16_t b;
  uint8_t c;
};

std::cout << sizeof(Info) << std::endl;   // 6  2 + 2 + 2
std::cout << alignof(Info) << std::endl;  // 2

struct alignas(4) Info2 {
  uint8_t a;
  uint16_t b;
  uint8_t c;
};

std::cout << sizeof(Info2) << std::endl;   // 8  4 + 4
std::cout << alignof(Info2) << std::endl;  // 4

```

`alignas`将内存对齐调整为4个字节。所以`sizeof(Info2)`的值变为了8。

```C++
// alignas 失效的情况

struct Info {
  uint8_t a;
  uint32_t b;
  uint8_t c;
};

std::cout << sizeof(Info) << std::endl;   // 12  4 + 4 + 4
std::cout << alignof(Info) << std::endl;  // 4

struct alignas(2) Info2 {
  uint8_t a;
  uint32_t b;
  uint8_t c;
};

std::cout << sizeof(Info2) << std::endl;   // 12  4 + 4 + 4
std::cout << alignof(Info2) << std::endl;  // 4

```



若`alignas`小于自然对齐的最小单位，则被忽略。

​		如果想使用单字节对齐的方式，使用`alignas`是无效的。应该使用`#pragma pack(push,1)`或者使用`__attribute__((packed))`。



```C++
#if defined(__GNUC__) || defined(__GNUG__)
  #define ONEBYTE_ALIGN __attribute__((packed))
#elif defined(_MSC_VER)
  #define ONEBYTE_ALIGN
  #pragma pack(push,1)
#endif

struct Info {
  uint8_t a;
  uint32_t b;
  uint8_t c;
} ONEBYTE_ALIGN;

#if defined(__GNUC__) || defined(__GNUG__)
  #undef ONEBYTE_ALIGN
#elif defined(_MSC_VER)
  #pragma pack(pop)
  #undef ONEBYTE_ALIGN
#endif

std::cout << sizeof(Info) << std::endl;   // 6 1 + 4 + 1
std::cout << alignof(Info) << std::endl;  // 1

```

确定结构体中每个元素大小可以通过下面这种方法:

```c++
#if defined(__GNUC__) || defined(__GNUG__)
  #define ONEBYTE_ALIGN __attribute__((packed))
#elif defined(_MSC_VER)
  #define ONEBYTE_ALIGN
  #pragma pack(push,1)
#endif

/**
* 0 1   3     6   8 9            15
* +-+---+-----+---+-+-------------+
* | |   |     |   | |             |
* |a| b |  c  | d |e|     pad     |
* | |   |     |   | |             |
* +-+---+-----+---+-+-------------+
*/
struct Info {
  uint16_t a : 1;
  uint16_t b : 2;
  uint16_t c : 3;
  uint16_t d : 2;
  uint16_t e : 1;
  uint16_t pad : 7;
} ONEBYTE_ALIGN;

#if defined(__GNUC__) || defined(__GNUG__)
  #undef ONEBYTE_ALIGN
#elif defined(_MSC_VER)
  #pragma pack(pop)
  #undef ONEBYTE_ALIGN
#endif

std::cout << sizeof(Info) << std::endl;   // 2
std::cout << alignof(Info) << std::endl;  // 1

```



## 3、指针和引用的区别

- 指针是一个变量，存储的是一个地址，引用跟原来的变量实质上是同一个东西，是原变量的别名
- 指针可以有多级，引用只有一级
- 指针可以为空，引用不能为NULL且在定义时必须初始化
- 指针在初始化后可以改变指向，而引用在初始化之后不可再改变
- sizeof指针得到的是本指针的大小，sizeof引用得到的是引用所指向变量的大小
- 当把指针作为参数进行传递时，也是将实参的一个拷贝传递给形参，两者指向的地址相同，但不是同一个变量，在函数中改变这个变量的指向不影响实参，而引用却可以。
- 引用本质是一个指针，同样会占4字节内存；指针是具体变量，需要占用存储空间（，具体情况还要具体分析）。
- 引用在声明时必须初始化为另一变量，一旦出现必须为typename refname &varname形式；指针声明和定义可以分开，可以先只声明指针变量而不初始化，等用到时再指向具体变量。
- 引用一旦初始化之后就不可以再改变（变量可以被引用为多次，但引用只能作为一个变量引用）；指针变量可以重新指向别的变量。
- 不存在指向空值的引用，必须有具体实体；但是存在指向空值的指针。

参考代码：

```c++
void test(int *p)
{
　　int a=1;
　　p=&a;
　　cout<<p<<" "<<*p<<endl;
}

int main(void)
{
    int *p=NULL;
    test(p);
    if(p==NULL)
    cout<<"指针p为NULL"<<endl;
    return 0;
}
//运行结果为：
//0x22ff44 1
//指针p为NULL


void testPTR(int* p) {
	int a = 12;
	p = &a;

}

void testREFF(int& p) {
	int a = 12;
	p = a;

}
void main()
{
	int a = 10;
	int* b = &a;
	testPTR(b);//改变指针指向，但是没改变指针的所指的内容
	cout << a << endl;// 10
	cout << *b << endl;// 10

	a = 10;
	testREFF(a);
	cout << a << endl;//12
}

```

​		在编译器看来, int a = 10; int &b = a; 等价于 int * const b = &a; 而 b = 20; 等价于 *b = 20; 自动转换为指针和自动解引用。





## 4、在传递函数参数时，什么时候该使用指针，什么时候该使用引用呢？

https://blog.csdn.net/qq_35820102/article/details/82863296

- 需要返回函数内局部变量的内存的时候用指针。使用指针传参需要开辟内存，用完要记得释放指针，不然会内存泄漏。而返回局部变量的引用是没有意义的
- 对栈空间大小比较敏感（比如递归）的时候使用引用。使用引用传递不需要创建临时变量，开销要更小
- 类对象作为参数传递的时候使用引用，这是C++类对象传递的标准方式





## 5、堆和栈的区别

- 申请方式不同。
  - 栈由系统自动分配。
- 堆是自己申请和释放的。
- 申请大小限制不同。
  - 栈顶和栈底是之前预设好的，栈是向栈底扩展，大小固定，可以通过ulimit -a查看，由ulimit -s修改。
  - 堆向高地址扩展，是不连续的内存区域，大小可以灵活调整。
- 申请效率不同。
  - 栈由系统分配，速度快，不会有碎片。
  - 堆由程序员分配，速度慢，且会有碎片。

栈空间默认是4M, 堆区一般是 1G - 4G

|                  |                              堆                              |                              栈                              |
| ---------------- | :----------------------------------------------------------: | :----------------------------------------------------------: |
| **管理方式**     |       堆中资源由**程序员控制**（容易产生memory leak）        |           栈资源由编**译器自动管理**，无需手工控制           |
| **内存管理机制** | 系统有一个记录空闲内存地址的链表，当系统收到程序申请时，遍历该链表，寻找第一个空间大于申请空间的堆结点，删 除空闲结点链表中的该结点，并将该结点空间分配给程序（大多数系统会在这块内存空间首地址记录本次分配的大小，这样delete才能正确释放本内存空间，另外系统会将多余的部分重新放入空闲链表中） | 只要栈的剩余空间大于所申请空间，系统为程序提供内存，否则报异常提示栈溢出。（这一块理解一下链表和队列的区别，不连续空间和连续空间的区别，应该就比较好理解这两种机制的区别了） |
| **空间大小**     | 堆是不连续的内存区域（因为系统是用链表来存储空闲内存地址，自然不是连续的），堆大小受限于计算机系统中有效的虚拟内存（32bit 系统理论上是4G），所以堆的空间比较灵活，比较大 | 栈是一块连续的内存区域，大小是操作系统预定好的，windows下栈大小是2M（也有是1M，在 编译时确定，VC中可设置） |
| **碎片问题**     |    对于堆，频繁的new/delete会造成大量碎片，使程序效率降低    | 对于栈，它是有点类似于数据结构上的一个先进后出的栈，进出一一对应，不会产生碎片。（看到这里我突然明白了为什么面试官在问我堆和栈的区别之前先问了我栈和队列的区别） |
| **生长方向**     |                  堆向上，向高地址方向增长。                  |                  栈向下，向低地址方向增长。                  |
| **分配方式**     |              堆都是动态分配（没有静态分配的堆）              | 栈有静态分配和动态分配，静态分配由编译器完成（如局部变量分配），动态分配由alloca函数分配，但栈的动态分配的资源由编译器进行释放，无需程序员实现。 |
| **分配效率**     |  堆由C/C++函数库提供，机制很复杂。所以堆的效率比栈低很多。   | 栈是其系统提供的数据结构，计算机在底层对栈提供支持，分配专门 寄存器存放栈地址，栈操作有专门指令。 |

**形象的比喻**

​		栈就像我们去饭馆里吃饭，只管点菜（发出申请）、付钱、和吃（使用），吃饱了就走，不必理会切菜、洗菜等准备工作和洗碗、刷锅等扫尾工作，他的好处是快捷，但是自由度小。

​		堆就象是自己动手做喜欢吃的菜肴，比较麻烦，但是比较符合自己的口味，而且自由度大.



## 6、你觉得堆快一点还是栈快一点？

https://blog.csdn.net/AlbenXie/article/details/103824830

​		毫无疑问是栈快一点。

​		因为操作系统会在底层对栈提供支持，会分配专门的寄存器存放栈的地址，栈的入栈出栈操作也十分简单，并且有专门的指令执行，所以栈的效率比较高也比较快。

​		而堆的操作是由C/C++函数库提供的，在分配堆内存的时候需要一定的算法寻找合适大小的内存。并且获取堆的内容需要两次访问，第一次访问指针，第二次根据指针保存的地址访问内存，因此堆比较慢。





## 7、区别以下指针类型？

```c++
int *p[10]
int (*p)[10]
int *p(int)
int (*p)(int)
```

- int *p[10]表示指针数组，强调数组概念，是一个数组变量，数组大小为10，数组内每个元素都是指向int类型的指针变量。
- int (*p)[10]表示数组指针，强调是指针，只有一个变量，是指针类型，不过指向的是一个int类型的数组，这个数组大小是10。
- int *p(int)是函数声明，函数名是p，参数是int类型的，返回值是int *类型的。
- int (*p)(int)是函数指针，强调是指针，该指针指向的函数具有int类型参数，并且返回值是int类型的





## 8、new / delete 与 malloc / free的异同

**相同点**

- 都可用于内存的动态申请和释放

**不同点**

- 前者是C++运算符，后者是C/C++语言标准库函数

- new自动计算要分配的空间大小，malloc需要手工计算

- new是类型安全的，malloc不是。例如：

- ```c++
  int *p = new float[2]; //编译错误
  int *p = (int*)malloc(2 * sizeof(double));//编译无错误
  
  ```

- new调用名为**operator new**的标准库函数分配足够空间并调用相关对象的构造函数，delete对指针所指对象运行适当的析构函数；然后通过调用名为**operator delete**的标准库函数释放该对象所用内存。后者均没有相关调用

- 后者需要库文件支持，前者不用

- new是封装了malloc，直接free不会报错，但是这只是释放内存，而不会析构对象





## 9、new和delete是如何实现的？

- new的实现过程是：
  - 首先调用名为**operator new**的标准库函数，分配足够大的原始为类型化的内存，以保存指定类型的一个对象；
  - 接下来运行该类型的一个构造函数，用指定初始化构造对象；
  - 最后返回指向新分配并构造后的的对象的指针
- delete的实现过程：
  - 对指针指向的对象运行适当的析构函数；
  - 然后通过调用名为**operator delete**的标准库函数释放该对象所用内存





## 10、malloc和new的区别？

- malloc和free是标准库函数，支持覆盖；new和delete是运算符，支持重载。
- malloc仅仅分配内存空间，free仅仅回收空间，不具备调用构造函数和析构函数功能，用malloc分配空间存储类的对象存在风险；new和delete除了分配回收功能外，还会调用构造函数和析构函数。
- malloc和free返回的是void类型指针（必须进行类型转换），new和delete返回的是具体类型指针。



## 11、既然有了malloc/free，C++中为什么还需要new/delete呢？直接用malloc/free不好吗？

- malloc/free和new/delete都是用来申请内存和回收内存的。
- 在对非基本数据类型的对象使用的时候，对象创建的时候还需要执行构造函数，销毁的时候要执行析构函数。而malloc/free是库函数，是已经编译的代码，所以不能把构造函数和析构函数的功能强加给malloc/free，所以new/delete是必不可少的



## 12、被free回收的内存是立即返还给操作系统吗？

​		不是的，被free回收的内存会首先被**ptmalloc**使用双链表保存起来，当用户下一次申请内存的时候，会尝试从这些内存中寻找合适的返回。这样就避免了频繁的系统调用，占用过多的系统资源。**同时ptmalloc也会尝试对小块内存进行合并，避免过多的内存碎片。**



## 13、宏定义和函数有何区别？

- 宏在**预处理阶段**完成替换，之后被替换的文本参与编译，相当于直接插入了代码，运行时不存在函数调用，执行起来更快；函数调用在**运行时**需要跳转到具体调用函数。
- 宏定义属于在结构中插入代码，没有返回值；函数调用具有返回值。
- 宏定义参数没有类型，不进行类型检查；函数参数具有类型，需要检查类型。
- 宏定义不要在最后加分号



## 14、宏定义和typedef区别？

- 宏主要用于定义常量及书写复杂的内容；typedef主要用于定义类型别名。
- 宏替换发生在编译阶段之前，属于文本插入替换；typedef是**编译**的一部分。
- 宏不检查类型；typedef会检查数据类型。
- 宏不是语句，不在在最后加分号；typedef是语句，要加分号标识结束。
- 注意对指针的操作，typedef char * p_char和#define p_char char *区别巨大





## 15、变量声明和定义区别？

- 声明仅仅是把变量的声明的位置及类型提供给编译器，并不分配内存空间；定义要在定义的地方为其分配存储空间。
- 相同变量可以在多处声明（外部变量extern），但只能在一处定义





## 16、strlen和sizeof区别？

- sizeof是运算符，并不是函数，结果在编译时得到而非运行中获得；strlen是字符处理的库函数。

- sizeof参数可以是任何数据的类型或者数据（sizeof参数不退化）；strlen的参数只能是字符指针且结尾是'\0'的字符串。

- 因为sizeof值在编译时确定，所以不能用来得到动态分配（运行时分配）存储空间的大小。

  ```c++
    int main(int argc, char const *argv[]){
        
        const char* str = "name";
  
        sizeof(str); // 取的是指针str的长度，是8
        strlen(str); // 取的是这个字符串的长度，不包含结尾的 \0。大小是4
        return 0;
    }
  
  ```

  

## 16.2、（补充题）一个指针占多少字节？

在16题中有提到sizeof（str）的值为8，是在64位的编译环境下的，指针的占用大小为8字节；

而在32位环境下，指针占用大小为4字节。

一个指针占内存的大小跟编译环境有关，而与机器的位数无关。

还有疑问的，可以自行打开Visual Studio编译器自己实验一番。



## 17、常量指针和指针常量区别？

- 指针常量是一个指针，读成常量的指针，指向一个只读变量，也就是后面所指明的int const 和 const int，都是一个常量，可以写作int const *p或const int *p。
- 常量指针是一个不能给改变指向的指针。指针是个常量，必须初始化，一旦初始化完成，它的值（也就是存放在指针中的地址）就不能在改变了，即不能中途改变指向，如int *const p。



## 18、a和&a有什么区别？

假设数组int a[10]; int (*p)[10] = &a;其中：

- a是数组名，是数组首元素地址，+1表示地址值加上一个int类型的大小，如果a的值是0x00000001，加1操作后变为0x00000005。*(a + 1) = a[1]。
- &a是数组的指针，其类型为int (*)[10]（就是前面提到的数组指针），其加1时，系统会认为是数组首地址加上整个数组的偏移（10个int型变量），值为数组a尾元素后一个元素的地址。
- 若(int *)p ，此时输出 *p时，其值为a[0]的值，因为被转为int *类型，解引用时按照int类型大小来读取。





## 19、C++和Python的区别

包括但不限于：

- Python是一种脚本语言，是解释执行的，而C++是编译语言，是需要编译后在特定平台运行的。python可以很方便的跨平台，但是效率没有C++高。
- Python使用缩进来区分不同的代码块，C++使用花括号来区分
- C++中需要事先定义变量的类型，而Python不需要，Python的基本数据类型只有数字，布尔值，字符串，列表，元组等等
- Python的库函数比C++的多，调用起来很方便



## 20、C++和C语言的区别

- C++中new和delete是对内存分配的运算符，取代了C中的malloc和free。
- 标准C++中的字符串类取代了标准C函数库头文件中的字符数组处理函数（C中没有字符串类型）。
- C++中用来做控制态输入输出的iostream类库替代了标准C中的stdio函数库。
- C++中的try/catch/throw异常处理机制取代了标准C中的setjmp()和longjmp()函数。
- 在C++中，允许有相同的函数名，不过它们的参数类型不能完全相同，这样这些函数就可以相互区别开来。而这在C语言中是不允许的。也就是C++可以重载，C语言不允许。
- C++语言中，允许变量定义语句在程序中的任何地方，只要在是使用它之前就可以；而C语言中，必须要在函数开头部分。而且C++不允许重复定义变量，C语言也是做不到这一点的
- 在C++中，除了值和指针之外，新增了引用。引用型变量是其他变量的一个别名，我们可以认为他们只是名字不相同，其他都是相同的。
- C++相对与C增加了一些关键字，如：bool、using、dynamic_cast、namespace等等



## 21、STL中的allocator、deallocator

1. 第一级配置器直接使用malloc()、free()和relloc()，第二级配置器视情况采用不同的策略：当配置区块超过128bytes时，视之为足够大，便调用第一级配置器；当配置器区块小于128bytes时，为了降低额外负担，使用复杂的内存池整理方式，而不再用一级配置器；
2. 第二级配置器主动将任何小额区块的内存需求量上调至8的倍数，并维护16个free-list，各自管理大小为8~128bytes的小额区块；
3. 空间配置函数allocate()，首先判断区块大小，大于128就直接调用第一级配置器，小于128时就检查对应的free-list。如果free-list之内有可用区块，就直接拿来用，如果没有可用区块，就将区块大小调整至8的倍数，然后调用refill()，为free-list重新分配空间；
4. 空间释放函数deallocate()，该函数首先判断区块大小，大于128bytes时，直接调用一级配置器，小于128bytes就找到对应的free-list然后释放内存。





## 22、STL中hash table扩容发生什么？

https://blog.csdn.net/sinat_14819779/article/details/116708832

1. hash table表格内的元素称为桶（bucket),而由桶所链接的元素称为节点（node),其中存入桶元素的容器为stl本身很重要的一种序列式容器——vector容器。之所以选择vector为存放桶元素的基础容器，主要是因为vector容器本身具有动态扩容能力，无需人工干预。
2. 向前操作：首先尝试从目前所指的节点出发，前进一个位置（节点），由于节点被安置于list内，所以利用节点的next指针即可轻易完成前进操作，如果目前正巧是list的尾端，就跳至下一个bucket身上，那正是指向下一个list的头部节点。





## 23、常见容器性质总结？

1. vector 底层数据结构为数组 ，支持快速随机访问

2. list 底层数据结构为双向链表，支持快速增删

3. deque 底层数据结构为一个中央控制器和多个缓冲区，详细见STL源码剖析P146，支持首尾（中间不能）快速增删，也支持随机访问

   deque是一个双端队列(double-ended queue)，也是在堆中保存内容的.它的保存形式如下:

   [堆1] --> [堆2] -->[堆3] --> ...

   每个堆保存好几个元素,然后堆和堆之间有指针指向,看起来像是list和vector的结合品.

4. stack 底层一般用list或deque实现，封闭头部即可，不用vector的原因应该是容量大小有限制，扩容耗时

5. queue 底层一般用list或deque实现，封闭头部即可，不用vector的原因应该是容量大小有限制，扩容耗时（stack和queue其实是适配器,而不叫容器，因为是对容器的再封装）

6. priority_queue 的底层数据结构一般为vector为底层容器，堆heap为处理规则来管理底层容器实现

7. set 底层数据结构为红黑树，有序，不重复

8. multiset 底层数据结构为红黑树，有序，可重复

9. map 底层数据结构为红黑树，有序，不重复

10 .multimap 底层数据结构为红黑树，有序，可重复

11. unordered_set 底层数据结构为hash表，无序，不重复

12. unordered_multiset 底层数据结构为hash表，无序，可重复

13. unordered_map 底层数据结构为hash表，无序，不重复

14. unordered_multimap 底层数据结构为hash表，无序，可重复



## 24、vector的增加删除都是怎么做的？为什么是1.5或者是2倍？

1. 新增元素：vector通过一个连续的数组存放元素，如果集合已满，在新增数据的时候，就要分配一块更大的内存，将原来的数据复制过来，释放之前的内存，在插入新增的元素；

2. 对vector的任何操作，一旦引起空间重新配置，指向原vector的所有迭代器就都失效了 ；

3. 初始时刻vector的capacity为0，塞入第一个元素后capacity增加为1；

4. 不同的编译器实现的扩容方式不一样，VS2015中以1.5倍扩容，GCC以2倍扩容。

   **对比可以发现采用采用成倍方式扩容，可以保证常数的时间复杂度，而增加指定大小的容量只能达到O(n)的时间复杂度，因此，使用成倍的方式扩容。**

5. 考虑可能产生的堆空间浪费，成倍增长倍数不能太大，使用较为广泛的扩容方式有两种，以2二倍的方式扩容，或者以1.5倍的方式扩容。

6. 以2倍的方式扩容，导致下一次申请的内存必然大于之前分配内存的总和，导致之前分配的内存不能再被使用，所以最好倍增长因子设置为(1,2)之间：

7. 向量容器vector的成员函数pop_back()可以删除最后一个元素.

8. 而函数erase()可以删除由一个iterator指出的元素，也可以删除一个指定范围的元素。

9. 还可以采用通用算法remove()来删除vector容器中的元素.

10. 不同的是：采用remove一般情况下不会改变容器的大小，而pop_back()与erase()等成员函数会改变容器的大小。



## 25、说一下STL每种容器对应的迭代器

| 容器                                   | 迭代器         |
| -------------------------------------- | -------------- |
| vector、deque                          | 随机访问迭代器 |
| stack、queue、priority_queue           | 无             |
| list、(multi)set/map                   | 双向迭代器     |
| unordered_(multi)set/map、forward_list | 前向迭代器     |



## 26、STL中迭代器失效的情况有哪些？

以vector为例：

**插入元素：**

​	1、尾后插入：size < capacity时，首迭代器不失效尾迭代失效（未重新分配空间），size == capacity时，所有迭代器均失效（需要重新分配空间）。

​	2、中间插入：中间插入：size < capacity时，首迭代器不失效但插入元素之后所有迭代器失效，size == capacity时，所有迭代器均失效。

**删除元素：**

尾后删除：只有尾迭代失效。

中间删除：删除位置之后所有迭代失效。

deque 和 vector 的情况类似,

而list双向链表每一个节点内存不连续, 删除节点仅当前迭代器失效,erase返回下一个有效迭代器;

map/set等关联容器底层是红黑树删除节点不会影响其他节点的迭代器, 使用递增方法获取下一个迭代器 mmp.erase(iter++);

unordered_(hash) 迭代器意义不大, rehash之后, 迭代器应该也是全部失效.



## 27、STL中vector的实现

​		vector是一种序列式容器，其数据安排以及操作方式与array非常类似，两者的唯一差别就是对于空间运用的灵活性，众所周知，array占用的是静态空间，一旦配置了就不可以改变大小，如果遇到空间不足的情况还要自行创建更大的空间，并手动将数据拷贝到新的空间中，再把原来的空间释放。vector则使用灵活的动态空间配置，维护一块**连续的线性空间**，在空间不足时，可以自动扩展空间容纳新元素，做到按需供给。其在扩充空间的过程中仍然需要经历：**重新配置空间，移动数据，释放原空间**等操作。这里需要说明一下动态扩容的规则：以原大小的两倍配置另外一块较大的空间（或者旧长度+新增元素的个数），源码：

```c++
const size_type len  = old_size + max(old_size, n);
```

Vector扩容倍数与平台有关，在Win + VS 下是 1.5倍，在 Linux + GCC 下是 2 倍

测试代码:

```c++
#include <iostream>
#include <vector>
using namespace std;

int main()
{
    //在Linux + GCC下
	vector<int> res(2,0);
	cout << res.capacity() <<endl; //2
	res.push_back(1);
	cout << res.capacity() <<endl;//4
	res.push_back(2);
	res.push_back(3);
    cout << res.capacity() <<endl;//8
	return 0;
    
    
    //在 win 10 + VS2019下
    vector<int> res(2,0);
	cout << res.capacity() <<endl; //2
	res.push_back(1);
	cout << res.capacity() <<endl;//3
	res.push_back(2);
	res.push_back(3);
    cout << res.capacity() <<endl;//6
     
}

```



​		运行上述代码，一开始配置了一块长度为2的空间，接下来插入一个数据，长度变为原来的两倍，为4，此时已占用的长度为3，再继续两个数据，此时长度变为8，可以清晰的看到空间的变化过程

​		**需要注意的是，频繁对vector调用push_back()对性能是有影响的，这是因为每插入一个元素，如果空间够用的话还能直接插入，若空间不够用，则需要重新配置空间，移动数据，释放原空间等操作，对程序性能会造成一定的影响.**





## 28、STL中slist的实现

​		list是双向链表，而slist（single linked list）是单向链表，它们的主要区别在于：**前者的迭代器是双向的Bidirectional iterator，后者的迭代器属于单向的Forward iterator。**虽然slist的很多功能不如list灵活，但是其所耗用的空间更小，操作更快。

​		根据STL的习惯，插入操作会将新元素插入到指定位置之前，而非之后，然而slist是不能回头的，只能往后走，因此在slist的其他位置插入或者移除元素是十分不明智的，但是在slist开头却是可取的，slist特别提供了insert_after()和erase_after供灵活应用。考虑到效率问题，slist只提供push_front()操作，元素插入到slist后，存储的次序和输入的次序是相反的

slist的单向迭代器如下图所示：

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205071953335.png)

slist默认采用alloc空间配置器配置节点的空间，其数据结构主要代码如下：

```c++
template <class T, class Allco = alloc>
class slist
{
	...
private:
    ...
    static list_node* create_node(const value_type& x){}//配置空间、构造元素
    static void destroy_node(list_node* node){}//析构函数、释放空间
private:
    list_node_base head; //头部
public:
    iterator begin(){}
    iterator end(){}
    size_type size(){}
    bool empty(){}
    void swap(slist& L){}//交换两个slist，只需要换head即可
    reference front(){} //取头部元素
    void push_front(const value& x){}//头部插入元素
    void pop_front(){}//从头部取走元素
    ...
}
```





举个例子：

```c++
#include <forward_list>
#include <algorithm>
#include <iostream>
using namespace std;

int main()
{
	forward_list<int> fl;
	fl.push_front(1);
	fl.push_front(3);
	fl.push_front(2);
	fl.push_front(6);
	fl.push_front(5);

	forward_list<int>::iterator ite1 = fl.begin();
	forward_list<int>::iterator ite2 = fl.end();
	for(;ite1 != ite2; ++ite1)
	{
		cout << *ite1 <<" "; // 5 6 2 3 1
	}
	cout << endl;

	ite1 = find(fl.begin(), fl.end(), 2); //寻找2的位置

	if (ite1 != ite2)
		fl.insert_after(ite1, 99);
	for (auto it : fl)
	{
		cout << it << " ";  //5 6 2 99 3 1
	}
	cout << endl;

	ite1 = find(fl.begin(), fl.end(), 6); //寻找6的位置
	if (ite1 != ite2)
		fl.erase_after(ite1);
	for (auto it : fl)
	{
		cout << it << " ";  //5 6 99 3 1
	}
	cout << endl;	
	return 0;
}
```

​		需要注意的是C++标准委员会没有采用slist的名称，forward_list在C++ 11中出现，它与slist的区别是没有size()方法。



## 29、STL中list的实现

​		相比于vector的连续线型空间，list显得复杂许多，但是它的好处在于插入或删除都只作用于一个元素空间，因此list对空间的运用是十分精准的，对任何位置元素的插入和删除都是常数时间。list不能保证节点在存储空间中连续存储，也拥有迭代器，迭代器的“++”、“--”操作对于的是指针的操作，list提供的迭代器类型是双向迭代器：Bidirectional iterators。

list节点的结构见如下源码：

```c++
template <class T>
struct __list_node{
    typedef void* void_pointer;
    void_pointer prev;
    void_pointer next;
    T data;
}

```

​		从源码可看出list显然是一个双向链表。list与vector的另一个区别是，在插入和接合操作之后，都不会造成原迭代器失效，而vector可能因为空间重新配置导致迭代器失效。

​		此外list也是一个环形链表，因此只要一个指针便能完整表现整个链表。list中node节点指针始终指向尾端的一个空白节点，因此是一种“**前闭后开**”的区间结构

​		list的空间管理默认采用alloc作为空间配置器，为了方便的以节点大小为配置单位，还定义一个list_node_allocator函数可一次性配置多个节点空间

​		由于list的双向特性，其支持在头部（front)和尾部（back)两个方向进行push和pop操作，当然还支持erase，splice，sort，merge，reverse，sort等操作，这里不再详细阐述。







## 30、STL中的deque的实现

​		vector是**单向开口**（尾部）的连续线性空间，deque则是一种**双向开口的**连续线性空间，虽然vector也可以在头尾进行元素操作，但是其头部操作的效率十分低下（主要是涉及到整体的移动）

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205071953980.png)



​		**deque和vector的最大差异一个是deque运行在常数时间内对头端进行元素操作，二是deque没有容量的概念，它是动态地以分段连续空间组合而成，可以随时增加一段新的空间并链接起来**。

​		deque虽然也提供随机访问的迭代器，但是其迭代器并不是普通的指针，其复杂程度比vector高很多，因此除非必要，否则一般使用vector而非deque。如果需要对deque排序，可以先将deque中的元素复制到vector中，利用sort对vector排序，再将结果复制回deque

​		deque由一段一段的定量连续空间组成，一旦需要增加新的空间，只要配置一段定量连续空间拼接在头部或尾部即可，因此deque的最大任务是如何维护这个整体的连续性

deque的数据结构如下：



```c++
class deque
{
    ...
protected:
    typedef pointer* map_pointer;//指向map指针的指针
    map_pointer map;//指向map
    size_type map_size;//map的大小
public:
    ...
    iterator begin();
    itertator end();
    ...
}

```

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220021322.png)



​		deque内部有一个指针指向map，map是一小块连续空间，其中的每个元素称为一个节点，node，每个node都是一个指针，指向另一段较大的连续空间，称为缓冲区，这里就是deque中实际存放数据的区域，默认大小512bytes。整体结构如上图所示。

​		deque的迭代器数据结构如下：

```c++
struct __deque_iterator
{
    ...
    T* cur;//迭代器所指缓冲区当前的元素
    T* first;//迭代器所指缓冲区第一个元素
    T* last;//迭代器所指缓冲区最后一个元素
    map_pointer node;//指向map中的node
    ...
}
```

从deque的迭代器数据结构可以看出，为了保持与容器联结，迭代器主要包含上述4个元素：

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220021453.png)

​		deque迭代器的“++”、“--”操作是远比vector迭代器繁琐，其主要工作在于缓冲区边界，如何从当前缓冲区跳到另一个缓冲区，当然deque内部在插入元素时，如果map中node数量全部使用完，且node指向的缓冲区也没有多余的空间，这时会配置新的map（2倍于当前+2的数量）来容纳更多的node，也就是可以指向更多的缓冲区。在deque删除元素时，也提供了元素的析构和空闲缓冲区空间的释放等机制.





## 31、STL中stack和queue的实现

**stack**

​		stack（栈）是一种先进后出（First In Last Out）的数据结构，只有一个入口和出口，那就是栈顶，除了获取栈顶元素外，没有其他方法可以获取到内部的其他元素，其结构图如下：

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220021348.png)

​		stack这种单向开口的数据结构很容易由**双向开口的deque和list**形成，只需要根据stack的性质对应移除某些接口即可实现，stack的源码如下：

```c++
template <class T, class Sequence = deque<T> >
class stack
{
	...
protected:
    Sequence c;
public:
    bool empty(){return c.empty();}
    size_type size() const{return c.size();}
    reference top() const {return c.back();}
    const_reference top() const{return c.back();}
    void push(const value_type& x){c.push_back(x);}
    void pop(){c.pop_back();}
};

```

​		从stack的数据结构可以看出，其所有操作都是围绕Sequence完成，而Sequence默认是deque数据结构。stack这种**“修改某种接口，形成另一种风貌”的行为，成为adapter(配接器)**。常将其归类为container adapter而非container

​		stack除了默认使用deque作为其底层容器之外，也可以使用双向开口的list，只需要在初始化stack时，将list作为第二个参数即可。由于stack只能操作顶端的元素，因此其内部元素无法被访问，也不提供迭代器。





**queue**

​		queue（队列）是一种先进先出（First In First Out）的数据结构，只有一个入口和一个出口，分别位于最底端和最顶端，出口元素外，没有其他方法可以获取到内部的其他元素，其结构图如下：

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220021436.png)

​		类似的，queue这种“先进先出”的数据结构很容易由双向开口的deque和list形成，只需要根据queue的性质对应移除某些接口即可实现，queue的源码如下：

```c++
template <class T, class Sequence = deque<T> >
class queue
{
	...
protected:
    Sequence c;
public:
    bool empty(){return c.empty();}
    size_type size() const{return c.size();}
    reference front() const {return c.front();}
    const_reference front() const{return c.front();}
    void push(const value_type& x){c.push_back(x);}
    void pop(){c.pop_front();}
};

```

​		从queue的数据结构可以看出，其所有操作都也都是是围绕Sequence完成，Sequence默认也是deque数据结构。queue也是一类container adapter。

​		同样，queue也可以使用list作为底层容器，不具有遍历功能，没有迭代器。

## 36、set和map的区别，multimap和multiset的区别

​		set只提供一种数据类型的接口，但是会将这一个元素分配到key和value上，而且它的compare_function用的是 identity()函数，这个函数是输入什么输出什么，这样就实现了set机制，set的key和value其实是一样的了。其实他保存的是两份元素，而不是只保存一份元素

​		map则提供两种数据类型的接口，分别放在key和value的位置上，他的比较function采用的是红黑树的comparefunction（），保存的确实是两份元素。

​		他们两个的insert都是采用红黑树的insert_unique() 独一无二的插入 。

​		multimap和map的唯一区别就是：multimap调用的是红黑树的insert_equal(),可以重复插入而map调用的则是独一无二的插入insert_unique()，multiset和set也一样，底层实现都是一样的，只是在插入的时候调用的方法不一样。

**红黑树概念**

面试时候现场写红黑树代码的概率几乎为0，但是红黑树一些基本概念还是需要掌握的。

1、它是二叉排序树（继承二叉排序树特显）：

- 若左子树不空，则左子树上所有结点的值均小于或等于它的根结点的值。

- 若右子树不空，则右子树上所有结点的值均大于或等于它的根结点的值。

  - 左、右子树也分别为二叉排序树。

  2、它满足如下几点要求：

  - 树中所有节点非红即黑。
  - 根节点必为黑节点。
  - 红节点的子节点必为黑（黑节点子节点可为黑）。
  - 从根到NULL的任何路径上黑结点数相同。

  3、查找时间一定可以控制在O(logn)。



## 32、STL中的heap的实现

​		**heap（堆）并不是STL的容器组件，是priority queue（优先队列）的底层实现机制**，因为binary max heap（大根堆）总是最大值位于堆的根部，优先级最高。

​		binary heap本质是一种complete binary tree（完全二叉树），整棵binary tree除了最底层的叶节点之外，都是填满的，但是叶节点从左到右不会出现空隙，如下图所示就是一颗完全二叉树

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220021792.png)

​		完全二叉树内没有任何节点漏洞，是非常紧凑的，这样的一个好处是可以使用array来存储所有的节点，因为当其中某个节点位于$i$处，其左节点必定位于$2i$处，右节点位于$2i+1$处，父节点位于$i/2$（向下取整）处。这种以array表示tree的方式称为隐式表述法。

​		因此我们可以使用一个array和一组heap算法来实现max heap（每个节点的值大于等于其子节点的值）和min heap（每个节点的值小于等于其子节点的值）。由于array不能动态的改变空间大小，用vector代替array是一个不错的选择。

那heap算法有哪些？常见有的插入、弹出、排序和构造算法，下面一一进行描述。

**push_heap插入算法**

​		由于完全二叉树的性质，新插入的元素一定是位于树的最底层作为叶子节点，并填补由左至右的第一个空格。事实上，在刚执行插入操作时，新元素位于底层vector的end()处，之后是一个称为percolate up（上溯）的过程，举个例子如下图：

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220022842.png)

​		新元素50在插入堆中后，先放在vector的end()存着，之后执行上溯过程，调整其根结点的位置，以便满足max heap的性质，如果了解大根堆的话，这个原理跟大根堆的调整过程是一样的。

**pop_heap算法**

​		heap的pop操作实际弹出的是根节点吗，但在heap内部执行pop_heap时，只是将其移动到vector的最后位置，然后再为这个被挤走的元素找到一个合适的安放位置，使整颗树满足完全二叉树的条件。这个被挤掉的元素首先会与根结点的两个子节点比较，并与较大的子节点更换位置，如此一直往下，直到这个被挤掉的元素大于左右两个子节点，或者下放到叶节点为止，这个过程称为percolate down（下溯）。举个例子：

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220022993.png)

​		根节点68被pop之后，移到了vector的最底部，将24挤出，24被迫从根节点开始与其子节点进行比较，直到找到合适的位置安身，需要注意的是pop之后元素并没有被移走，如果要将其移走，可以使用pop_back()。

**sort算法**

​		一言以蔽之，因为pop_heap可以将当前heap中的最大值置于底层容器vector的末尾，heap范围减1，那么不断的执行pop_heap直到树为空，即可得到一个递增序列。



**make_heap算法**

将一段数据转化为heap，一个一个数据插入，调用上面说的两种percolate算法即可。

代码实测

```cpp
#include <iostream>
#include <algorithm>
#include <vector>
using namespace std;

int main()
{
	vector<int> v = { 0,1,2,3,4,5,6 };
	make_heap(v.begin(), v.end()); //以vector为底层容器
	for (auto i : v)
	{
		cout << i << " "; // 6 4 5 3 1 0 2
	}
	cout << endl;
	v.push_back(7);
	push_heap(v.begin(), v.end());
	for (auto i : v)
	{
		cout << i << " "; // 7 6 5 4 1 0 2 3
	}
	cout << endl;
	pop_heap(v.begin(), v.end());
	cout << v.back() << endl; // 7 
	v.pop_back();
	for (auto i : v)
	{
		cout << i << " "; // 6 4 5 3 1 0 2
	}
	cout << endl;
	sort_heap(v.begin(), v.end());
	for (auto i : v)
	{
		cout << i << " "; // 0 1 2 3 4 5 6
	}
	return 0;
}
```



## 33、STL中的priority_queue的实现

​		priority_queue，优先队列，是一个拥有权值观念的queue，它跟queue一样是顶部入口，底部出口，在插入元素时，元素并非按照插入次序排列，它会自动根据权值（通常是元素的实值）排列，权值最高，排在最前面，如下图所示。

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220022333.png)



​		默认情况下，priority_queue使用一个max-heap完成，底层容器使用的是一般为vector为底层容器，堆heap为处理规则来管理底层容器实现 。priority_queue的这种实现机制导致其不被归为容器，而是一种容器配接器。关键的源码如下：



```c++
template <class T, class Squence = vector<T>, 
class Compare = less<typename Sequence::value_tyoe> >
class priority_queue{
	...
protected:
    Sequence c; // 底层容器
    Compare comp; // 元素大小比较标准
public:
    bool empty() const {return c.empty();}
    size_type size() const {return c.size();}
    const_reference top() const {return c.front()}
    void push(const value_type& x)
    {
        c.push_heap(x);
        push_heap(c.begin(), c.end(),comp);
    }
    void pop()
    {
        pop_heap(c.begin(), c.end(),comp);
        c.pop_back();
    }
};

```

​		priority_queue的所有元素，进出都有一定的规则，只有queue顶端的元素（权值最高者），才有机会被外界取用，它没有遍历功能，也不提供迭代器

举个例子：

```c++
#include <queue>
#include <iostream>
using namespace std;

int main()
{
	int ia[9] = {0,4,1,2,3,6,5,8,7 };
	priority_queue<int> pq(ia, ia + 9);
	cout << pq.size() <<endl;  // 9
	for(int i = 0; i < pq.size(); i++)
	{
		cout << pq.top() << " "; // 8 8 8 8 8 8 8 8 8
	}
	cout << endl;
	while (!pq.empty())
	{
		cout << pq.top() << ' ';// 8 7 6 5 4 3 2 1 0
		pq.pop();
	}
	return 0;
}

```



## 34、STL中set的实现？

​		STL中的容器可分为序列式容器（sequence）和关联式容器（associative），set属于关联式容器。

​		set的特性是，所有元素都会根据元素的值自动被排序（默认升序），set元素的键值就是实值，实值就是键值，		

​		set不允许有两个相同的键值

​		set不允许迭代器修改元素的值，其迭代器是一种constance iterators

​		标准的STL set以RB-tree（红黑树）作为底层机制，几乎所有的set操作行为都是转调用RB-tree的操作行为，这里补充一下红黑树的特性：

- 每个节点不是红色就是黑色
- 根结点为黑色
- 如果节点为红色，其子节点必为黑
- 任一节点至（NULL）树尾端的任何路径，所含的黑节点数量必相同

关于红黑树的具体操作过程，比较复杂读者可以翻阅《算法导论》详细了解。

举个例子：



```c++
#include <set>
#include <iostream>
using namespace std;


int main()
{
	int i;
	int ia[5] = { 1,2,3,4,5 };
	set<int> s(ia, ia + 5);
	cout << s.size() << endl; // 5
	cout << s.count(3) << endl; // 1
	cout << s.count(10) << endl; // 0

	s.insert(3); //再插入一个3
	cout << s.size() << endl; // 5
	cout << s.count(3) << endl; // 1

	s.erase(1);
	cout << s.size() << endl; // 4

	set<int>::iterator b = s.begin();
	set<int>::iterator e = s.end();
	for (; b != e; ++b)
		cout << *b << " "; // 2 3 4 5
	cout << endl;

	b = find(s.begin(), s.end(), 5);
	if (b != s.end())
		cout << "5 found" << endl; // 5 found

	b = s.find(2);
	if (b != s.end())
		cout << "2 found" << endl; // 2 found

	b = s.find(1);
	if (b == s.end())
		cout << "1 not found" << endl; // 1 not found
	return 0;
}

```

​		关联式容器尽量使用其自身提供的find()函数查找指定的元素，效率更高，因为STL提供的find()函数是一种顺序搜索算法。



## 35、STL中map的实现

​		map的特性是所有元素会根据键值进行自动排序。map中所有的元素都是pair，拥有键值(key)和实值(value)两个部分，并且不允许元素有相同的key

​		**一旦map的key确定了，那么是无法修改的，但是可以修改这个key对应的value，**因此map的迭代器既不是constant iterator，也不是mutable iterator

​		标准STL map的底层机制是RB-tree（红黑树），另一种以hash table为底层机制实现的称为hash_map。map的架构如下图所示

![img](https://axiu-image-bed.oss-cn-shanghai.aliyuncs.com/img/202205220022980.png)

​		map的在构造时缺省采用递增排序key，也使用alloc配置器配置空间大小，需要注意的是在插入元素时，调用的是红黑树中的insert_unique()方法，而非insert_euqal()（multimap使用）

举个例子:

```c++
#include <map>
#include <iostream>
#include <string>
using namespace std;

int main()
{
	map<string, int> maps;
    //插入若干元素
	maps["jack"] = 1;
	maps["jane"] = 2;
	maps["july"] = 3;
	//以pair形式插入
	pair<string, int> p("david", 4);
	maps.insert(p);
	//迭代输出元素
	map<string, int>::iterator iter = maps.begin();
	for (; iter != maps.end(); ++iter)
	{
		cout << iter->first << " ";
		cout << iter->second << "--"; //david 4--jack 1--jane 2--july 3--
	}
	cout << endl;
	//使用subscipt操作取实值
	int num = maps["july"];
	cout << num << endl; // 3
	//查找某key
	iter = maps.find("jane");
	if(iter != maps.end())
		cout << iter->second << endl; // 2
    //修改实值
	iter->second = 100;
	int num2 = maps["jane"]; // 100
	cout << num2 << endl;
	
	return 0;
}

```

​		需要注意的是subscript（下标）操作既可以作为左值运用（修改内容）也可以作为右值运用（获取实值）。例如：

```c++
maps["abc"] = 1; //左值运用int num = masp["abd"]; //右值运用 
```

无论如何，subscript操作符都会先根据键值找出实值，源码如下：

```c++
...T& operator[](const key_type& k){	
    return (*((insert(value_type(k, T()))).first)).second;
}...
```

​		代码运行过程是：首先根据键值和实值做出一个元素，这个元素的实值未知，因此产生一个与实值型别相同的临时对象替代：

```c++
value_type(k, T());
```

​	再将这个对象插入到map中，并返回一个pair：

```c++
pair<iterator,bool> insert(value_type(k, T()));
```

​		pair第一个元素是迭代器，指向当前插入的新元素，如果插入成功返回true，此时对应左值运用，根据键值插入实值。插入失败（重复插入）返回false，此时返回的是已经存在的元素，则可以取到它的实值

```c++
(insert(value_type(k, T()))).first; //迭代器
*((insert(value_type(k, T()))).first); //解引用
(*((insert(value_type(k, T()))).first)).second; //取出实值
```

由于这个实值是以引用方式传递，因此作为左值或者右值都可以.





## 36、set和map的区别，multimap和multiset的区别

​		set只提供一种数据类型的接口，但是会将这一个元素分配到key和value上，而且它的compare_function用的是 identity()函数，这个函数是输入什么输出什么，这样就实现了set机制，set的key和value其实是一样的了。其实他保存的是两份元素，而不是只保存一份元素

​		map则提供两种数据类型的接口，分别放在key和value的位置上，他的比较function采用的是红黑树的comparefunction（），保存的确实是两份元素。

​		他们两个的insert都是采用红黑树的insert_unique() 独一无二的插入 。

​		multimap和map的唯一区别就是：multimap调用的是红黑树的insert_equal(),可以重复插入而map调用的则是独一无二的插入insert_unique()，multiset和set也一样，底层实现都是一样的，只是在插入的时候调用的方法不一样。

**红黑树概念**

面试时候现场写红黑树代码的概率几乎为0，但是红黑树一些基本概念还是需要掌握的。

1、它是二叉排序树（继承二叉排序树特显）：

- 若左子树不空，则左子树上所有结点的值均小于或等于它的根结点的值。

- 若右子树不空，则右子树上所有结点的值均大于或等于它的根结点的值。

  - 左、右子树也分别为二叉排序树。

  2、它满足如下几点要求：

  - 树中所有节点非红即黑。
  - 根节点必为黑节点。
  - 红节点的子节点必为黑（黑节点子节点可为黑）。
  - 从根到NULL的任何路径上黑结点数相同。

  3、查找时间一定可以控制在O(logn)。





## 37、STL中unordered_map和map的区别和应用场景

​		map支持键值的自动排序，底层机制是红黑树，红黑树的查询和维护时间复杂度均为$O(logn)$，但是空间占用比较大，因为每个节点要保持父节点、孩子节点及颜色的信息

​		unordered_map是C++ 11新添加的容器，底层机制是哈希表，通过hash函数计算元素位置，其查询时间复杂度为O(1)，维护时间与bucket桶所维护的list长度有关，但是建立hash表耗时较大

​		从两者的底层机制和特点可以看出：map适用于有序数据的应用场景，unordered_map适用于高效查询的应用场景



## 38、hashtable中解决冲突有哪些方法？

**记住前三个：**

**线性探测**

​		使用hash函数计算出的位置如果已经有元素占用了，则向后依次寻找，找到表尾则回到表头，直到找到一个空位

**开链**

​		每个表格维护一个list，如果hash函数计算出的格子相同，则按顺序存在这个list中

**再散列**

​		发生冲突时使用另一种hash函数再计算一个地址，直到不冲突

**二次探测**

​		使用hash函数计算出的位置如果已经有元素占用了，按照$1^2$、$2^2$、$3^2$...的步长依次寻找，如果步长是随机数序列，则称之为伪随机探测

**公共溢出区**

​		一旦hash函数计算的结果相同，就放入公共溢出区



## 39、C++中有几种类型的new

在C++中，new有三种典型的使用方法：plain new，nothrow new和placement new。

（1）**plain new**

言下之意就是普通的new，就是我们常用的new，在C++中定义如下：

```c++
void* operator new(std::size_t) throw(std::bad_alloc);
void operator delete(void *) throw();
```

​		因此**plain new**在空间分配失败的情况下，抛出异常**std::bad_alloc**而不是返回NULL，因此通过判断返回值是否为NULL是徒劳的，举个例子：

```c++
#include <iostream>
#include <string>
using namespace std;
int main()
{
	try
	{
		char *p = new char[10e11];
		delete p;
	}
	catch (const std::bad_alloc &ex)
	{
		cout << ex.what() << endl;
	}
	return 0;
}
//执行结果：bad allocation

```

（2）**nothrow new**

nothrow new在空间分配失败的情况下是不抛出异常，而是返回NULL，定义如下：

```c++
void * operator new(std::size_t,const std::nothrow_t&) throw();
void operator delete(void*) throw();
```

```c++
#include <iostream>
#include <string>
using namespace std;

int main()
{
	char *p = new(nothrow) char[10e11];
	if (p == NULL) 
	{
		cout << "alloc failed" << endl;
	}
	delete p;
	return 0;
}
//运行结果：alloc failed
```

（3）**placement new**

​		这种new允许在一块已经分配成功的内存上重新构造对象或对象数组。placement new不用担心内存分配失败，因为它根本不分配内存，它做的唯一一件事情就是调用对象的构造函数。定义如下：

```c++
void* operator new(size_t,void*);
void operator delete(void*,void*);
```

使用placement new需要注意两点：

- palcement new的主要用途就是反复使用一块较大的动态分配的内存来构造不同类型的对象或者他们的数组
- placement new构造起来的对象数组，要显式的调用他们的析构函数来销毁（析构函数并不释放对象的内存），千万不要使用delete，这是因为placement new构造起来的对象或数组大小并不一定等于原来分配的内存大小，使用delete会造成内存泄漏或者之后释放内存时出现运行时错误。

```c++
#include <iostream>
#include <string>
using namespace std;
class ADT{
	int i;
	int j;
public:
	ADT(){
		i = 10;
		j = 100;
		cout << "ADT construct i=" << i << "j="<<j <<endl;
	}
	~ADT(){
		cout << "ADT destruct" << endl;
	}
};
int main()
{
	char *p = new(nothrow) char[sizeof ADT + 1];
	if (p == NULL) {
		cout << "alloc failed" << endl;
	}
	ADT *q = new(p) ADT;  //placement new:不必担心失败，只要p所指对象的的空间足够ADT创建即可
	//delete q;//错误!不能在此处调用delete q;
	q->ADT::~ADT();//显示调用析构函数
	delete[] p;
	return 0;
}
//输出结果：
//ADT construct i=10j=100
//ADT destruct

```



## 40、C++的异常处理的方法

​	在程序执行过程中，由于程序员的疏忽或是系统资源紧张等因素都有可能导致异常，任何程序都无法保证绝对的稳定，常见的异常有：

- 数组下标越界
- 除法计算时除数为0
- 动态分配空间时空间不足
- ...

如果不及时对这些异常进行处理，程序多数情况下都会崩溃。

**（1）try、throw和catch关键字**

C++中的异常处理机制主要使用**try**、**throw**和**catch**三个关键字，其在程序中的用法如下：

```c++
#include <iostream>
using namespace std;
int main()
{
    double m = 1, n = 0;
    try {
        cout << "before dividing." << endl;
        if (n == 0)
            throw - 1;  //抛出int型异常
        else if (m == 0)
            throw - 1.0;  //拋出 double 型异常
        else
            cout << m / n << endl;
        cout << "after dividing." << endl;
    }
    catch (double d) {
        cout << "catch (double)" << d << endl;
    }
    catch (...) {
        cout << "catch (...)" << endl;
    }
    cout << "finished" << endl;
    return 0;
}
//运行结果
//before dividing.
//catch (...)
//finished
```

​		代码中，对两个数进行除法计算，其中除数为0。可以看到以上三个关键字，程序的执行流程是先执行try包裹的语句块，如果执行过程中没有异常发生，则不会进入任何catch包裹的语句块，如果发生异常，则使用throw进行异常抛出，再由catch进行捕获，throw可以抛出各种数据类型的信息，代码中使用的是数字，也可以自定义异常class。**catch根据throw抛出的数据类型进行精确捕获（不会出现类型转换），如果匹配不到就直接报错，可以使用catch(...)的方式捕获任何异常（不推荐）。**当然，如果catch了异常，当前函数如果不进行处理，或者已经处理了想通知上一层的调用者，可以在catch里面再throw异常。

**（2）函数的异常声明列表**

​		有时候，程序员在定义函数的时候知道函数可能发生的异常，可以在函数声明和定义时，指出所能抛出异常的列表，写法如下：

```c++
int fun() throw(int,double,A,B,C){...};

```

​		这种写法表名函数可能会抛出int,double型或者A、B、C三种类型的异常，如果throw中为空，表明不会抛出任何异常，如果没有throw则可能抛出任何异常。



**（3）C++标准异常类 exception**

C++ 标准库中有一些类代表异常，这些类都是从 exception 类派生而来的，如下图所示:

![1676549342422](../../../%E9%9D%A2%E8%AF%95/%E5%85%AB%E8%82%A1%E6%96%87/C++/1676549342422.png)



- bad_typeid：使用typeid运算符，如果其操作数是一个多态类的指针，而该指针的值为 NULL，则会拋出此异常，例如：

```c++
#include <iostream>
#include <typeinfo>
using namespace std;

class A{
public:
  virtual ~A();
};
 
using namespace std;
int main() {
	A* a = NULL;
	try {
  		cout << typeid(*a).name() << endl; // Error condition
  	}
	catch (bad_typeid){
  		cout << "Object is NULL" << endl;
  	}
    return 0;
}
//运行结果：bject is NULL
```

- bad_cast：在用 dynamic_cast 进行从多态基类对象（或引用）到派生类的引用的强制类型转换时，如果转换是不安全的，则会拋出此异常
- bad_alloc：在用 new 运算符进行动态内存分配时，如果没有足够的内存，则会引发此异常
- out_of_range:用 vector 或 string的at 成员函数根据下标访问元素时，如果下标越界，则会拋出此异常





## 41、static的用法和作用？

1. 先来介绍它的第一条也是最重要的一条：**隐藏**。（static函数，static变量均可）

   ​	    当同时编译多个文件时，所有未加static前缀的全局变量和函数都具有全局可见性。

2. static的第二个作用是**保持变量内容的持久**。（static变量中的记忆功能和全局生存期）存储在静态数据区的变量会在程序刚开始运行时就完成初始化，也是唯一的一次初始化。**共有两种变量存储在静态存储区：全局变量和static变量**，只不过和全局变量比起来，static可以控制变量的可见范围，说到底static还是用来隐藏的。

3. static的第三个作用是**默认初始化为0**（static变量）

   ​		其实全局变量也具备这一属性，因为全局变量也存储在静态数据区。**在静态数据区，内存中所有的字节默认值都是0x00**，某些时候这一特点可以减少程序员的工作量。

4. static的第四个作用：

   C++中的类成员声明static

   1. **函数体内static变量的作用范围为该函数体**，不同于auto变量，该变量的内存只被分配一次，因此其值在下次调用时仍维持上次的值；
   2. 在模块内的static全局变量可以被模块内所有函数访问，但不能被模块外其它函数访问；
   3. 在模块内的static函数只可被这一模块内的其它函数调用，这个函数的使用范围被限制在声明它的模块内；
   4. **在类中的static成员变量属于整个类所拥有**，对类的所有对象只有一份拷贝；
   5. 在类中的static成员函数属于整个类所拥有，**这个函数不接收this指针**，因而只能访问类的static成员变量。

   类内：

   1. static类对象必须要在类外进行初始化，**static修饰的变量先于对象存在**，所以static修饰的变量要在类外初始化；
   2. 由于static修饰的类成员属于类，不属于对象，因此static类成员函数是没有this指针的，this指针是指向本对象的指针。正因为没有this指针，所以static类成员函数不能访问非static的类成员，只能访问 static修饰的类成员；
   3. **static成员函数不能被virtual修饰**，static成员不属于任何对象或实例，所以加上virtual没有任何实际意义；静态成员函数没有this指针，**虚函数的实现是为每一个对象分配一个vptr指针，而vptr是通过this指针调用的，所以不能为virtual**；虚函数的调用关系，**this->vptr->ctable->virtual function**





## 42、指针和const的用法

1. 当const修饰指针时，由于const的位置不同，它的修饰对象会有所不同。

2. int *const p2中const修饰p2的值,所以理解为p2的值不可以改变，即p2只能指向固定的一个变量地址，但可以通过*p2读写这个变量的值。顶层指针表示指针本身是一个常量

3. int const *p1或者const int *p1两种情况中const修饰*p1，所以理解为*p1的值不可以改变，即不可以给*p1赋值改变p1指向变量的值，但可以通过给p赋值不同的地址改变这个指针指向。

   ​	底层指针表示指针所指向的变量是一个常量。



## 43、形参与实参的区别？

1. **形参变量只有在被调用时才分配内存单元**，在调用结束时， 即刻释放所分配的内存单元。因此，形参只有在函数内部有效。 函数调用结束返回主调函数后则不能再使用该形参变量。
2. 实参可以是常量、变量、表达式、函数等， 无论实参是何种类型的量，在进行函数调用时，它们都必须具有确定的值， 以便把这些值传送给形参。 因此应预先用赋值，输入等办法使实参获得确定值，会产生一个临时变量。
3. 实参和形参在数量上，类型上，顺序上应严格一致， 否则会发生“类型不匹配”的错误。
4. 函数调用中发生的数据传送是单向的。 即只能把实参的值传送给形参，而不能把形参的值反向地传送给实参。 因此在函数调用过程中，形参的值发生改变，而实参中的值不会变化。
5. 当形参和实参不是指针类型时，在该函数运行时，形参和实参是不同的变量，他们在内存中位于不同的位置，形参将实参的内容复制一份，在该函数运行结束的时候形参被释放，而实参内容不会改变。





## 44、值传递、指针传递、引用传递的区别和效率

1. 值传递：有一个形参向函数所属的栈拷贝数据的过程，如果值传递的对象是类对象 或是大的结构体对象，将耗费一定的时间和空间。（传值）
2. 指针传递：同样有一个形参向函数所属的栈拷贝数据的过程，但拷贝的数据是一个固定为4字节的地址。（传值，传递的是地址值）
3. 引用传递：同样有上述的数据拷贝过程，但其是针对地址的，相当于为该数据所在的地址起了一个别名。（传地址）
4. 效率上讲，指针传递和引用传递比值传递效率高。一般主张使用引用传递，代码逻辑上更加紧凑、清晰。





## 45、静态变量什么时候初始化

1.  初始化只有一次，但是可以多次赋值，在主程序之前，编译器已经为其分配好了内存。
2.  静态局部变量和全局变量一样，数据都存放在全局区域，**所以在主程序之前，编译器已经为其分配好了内存**，**但在C和C++中静态局部变量的初始化节点又有点不太一样**。**在C中，初始化发生在代码执行之前，编译阶段分配好内存之后，就会进行初始化，所以我们看到在C语言中无法使用变量对静态局部变量进行初始化，在程序运行结束，变量所处的全局内存会被全部回收。**
3.  **而在C++中，初始化时在执行相关代码时才会进行初始化，主要是由于C++引入对象后，要进行初始化必须执行相应构造函数和析构函数，在构造函数或析构函数中经常会需要进行某些程序中需要进行的特定操作，并非简单地分配内存。所以C++标准定为全局或静态对象是有首次用到时才会进行构造，并通过atexit()来管理。在程序结束，按照构造顺序反方向进行逐个析构。所以在C++中是可以使用变量对静态局部变量进行初始化的。**



## 46、const关键字的作用有哪些?

1. **阻止一个变量被改变**，可以使用const关键字。在定义该const变量时，通常需要对它进行初始化，因为以后就没有机会再去改变它了；
2. 对指针来说，可以指定指针本身为const，也可以指定指针所指的数据为const，或二者同时指定为const；
3. 在一个函数声明中，const可以修饰形参，表明它是一个输入参数，在函数内部不能改变其值；
4. 对于类的成员函数，若指定其为const类型，则表明其是一个常函数，不能修改类的成员变量，类的常对象只能访问类的常成员函数；
5. **对于类的成员函数，有时候必须指定其返回值为const类型，以使得其返回值不为“左值”**。
6. const成员函数可以访问非const对象的非const数据成员、const数据成员，也可以访问const对象内的所有数据成员；
7. 非const成员函数可以访问非const对象的非const数据成员、const数据成员，但不可以访问const对象的任意数据成员；
8. 一个没有明确声明为const的成员函数被看作是将要修改对象中数据成员的函数，而且编译器不允许它为一个const对象所调用。因此const对象只能调用const成员函数。
9. const类型变量可以通过类型转换符const_cast将const类型转换为非const类型；
10. const类型变量必须定义的时候进行初始化，因此也导致如果类的成员变量有const类型的变量，那么该变量必须在类的初始化列表中进行初始化；
11. 对于函数值传递的情况，因为参数传递是通过复制实参创建一个临时变量传递进函数的，函数内只能改变临时变量，但无法改变实参。则这个时候无论加不加const对实参不会产生任何影响。但是在引用或指针传递函数调用中，因为传进去的是一个引用或指针，这样函数内部可以改变引用或指针所指向的变量，这时const 才是实实在在地保护了实参所指向的变量。因为在编译阶段编译器对调用函数的选择是根据实参进行的，所以，只有引用传递和指针传递可以用是否加const来重载。一个拥有顶层const的形参无法和另一个没有顶层const的形参区分开来。



## 47、什么是类的继承？

1. 类与类之间的关系

   **has-A**包含关系，用以描述一个类由多个部件类构成，实现has-A关系用类的成员属性表示，即一个类的成员属性是另一个已经定义好的类；

   **use-A**，一个类使用另一个类，通过类之间的成员函数相互联系，定义友元或者通过传递参数的方式来实现；

   **is-A**，继承关系，关系具有传递性；

2. 继承的相关概念

      	所谓的继承就是一个类继承了另一个类的属性和方法，这个新的类包含了上一个类的属性和方法，被称为子类或者派生类，被继承的类称为父类或者基类；

3. 继承的特点

   子类拥有父类的所有属性和方法，子类可以拥有父类没有的属性和方法，子类对象可以当做父类对象使用；

4. 继承中的访问控制

   public、protected、private

5. 继承中的构造和析构函数

6. 继承中的兼容性原则



## 48、从汇编层去解释一下引用

```pascal
9:      int x = 1;			00401048  mov     dword ptr [ebp-4],1

10:     int &b = x;			0040104F  lea     eax,[ebp-4]
							00401052  mov     dword ptr [ebp-8],eax

```

​	x的地址为ebp-4，b的地址为ebp-8，因为栈内的变量内存是从高往低进行分配的，所以b的地址比x的低。

​	lea eax,[ebp-4] 这条语句将x的地址ebp-4放入eax寄存器

​	mov dword ptr [ebp-8],eax 这条语句将eax的值放入b的地址

​	ebp-8中上面两条汇编的作用即：将x的地址存入变量b中，这不和将某个变量的地址存入指针变量是一样的吗？**所以从汇编层次来看，的确引用是通过指针来实现的。**





## 49、深拷贝与浅拷可以描述一下吗？

​		浅复制 ：只是拷贝了基本类型的数据，而引用类型数据，复制后也是会发生引用，我们把这种拷贝叫做“（浅复制）浅拷贝”，换句话说，**浅复制仅仅是指向被复制的内存地址**，如果原地址中对象被改变了，那么浅复制出来的对象也会相应改变。

​		深复制 ：在计算机中开辟了一块新的内存地址用于存放复制的对象。

​		在某些状况下，类内成员变量需要动态开辟堆内存，如果实行位拷贝，也就是把对象里的值完全复制给另一个对象，如A=B。这时，如果B中有一个成员变量指针已经申请了内存，那A中的那个成员变量也指向同一块内存。这就出现了问题：当B把内存释放了（如：析构），这时A内的指针就是野指针了，出现运行错误。





## 50、new和malloc的区别

1. new/delete是C++**关键字**，需要编译器支持。malloc/free是**库函数**，需要头文件支持；

2. 使用new操作符申请内存分配时**无须指定**内存块的大小，编译器会根据类型信息自行计算。而malloc则需要**显式地指出**所需内存的尺寸。

3. new操作符内存分配成功时，返回的是对象类型的指针，类型严格与对象匹配，无须进行类型转换，故new是符合类型**安全性**的操作符。而malloc内存分配成功则是返回void * ，需要通过强制类型转换将void*指针转换成我们需要的类型。

4. new内存分配失败时，会抛出bac_alloc异常。malloc分配内存失败时返回NULL。

5. new会先调用operator new函数，申请足够的内存（通常底层使用malloc实现）。然后调用类型的构造函数，初始化成员变量，最后返回自定义类型指针。delete先调用析构函数，然后调用operator delete函数释放内存（通常底层使用free实现）。malloc/free是库函数，只能动态的申请和释放内存，无法强制要求其做自定义类型对象构造和析构工作。





## 51、delete p、delete [] p、allocator都有什么作用？

1. 动态数组管理new一个数组时，[]中必须是一个整数，但是不一定是常量整数，普通数组必须是一个常量整数；

2. new动态数组返回的并不是数组类型，而是一个**元素类型的指针**；

3. delete[]时，数组中的元素**按逆序的顺序进行销毁**；

4. new在内存分配上面有一些局限性，new的机制是将内存分配和对象构造组合在一起，同样的，delete也是将对象析构和内存释放组合在一起的。allocator将这两部分分开进行，allocator申请一部分内存，不进行初始化对象，只有当需要的时候才进行初始化操作。





## 52、new和delete的实现原理， delete是如何知道释放内存的大小的？

1. new简单类型直接调用operator new分配内存；

​		而对于复杂结构，**先调用operator new分配内存，然后在分配的内存上调用构造函数**；

​		对于简单类型，new[]计算好大小后调用operator new；

​		对于复杂数据结构，new[]先调用operator new[]分配内存，然后在p的前四个字节写入数组大小n，然后调用n次构造函数，针对复杂类型，new[]会额外存储数组大小；

​		① new表达式调用一个名为operator new(operator new[])函数，分配一块足够大的、原始的、未命名的内存空间；

​		② 编译器运行相应的构造函数以构造这些对象，并为其传入初始值；

​		③ 对象被分配了空间并构造完成，返回一个指向该对象的指针。

2. delete简单数据类型默认只是调用free函数；

   复杂数据类型先调用析构函数再调用operator delete；针对简单类型，delete和delete[]等同。假设指针p指向new[]分配的内存。因为要4字节存储数组大小，实际分配的内存地址为[p-4]，系统记录的也是这个地址。delete[]实际释放的就是p-4指向的内存。而delete会直接释放p指向的内存，这个内存根本没有被系统记录，所以会崩溃。

3. 需要在 new [] 一个对象数组时，需要保存数组的维度，**C++ 的做法是在分配数组空间时多分配了 4 个字节的大小，专门保存数组的大小，在 delete [] 时就可以取出这个保存的数，就知道了需要调用析构函数多少次了**。



## 53、malloc申请的存储空间能用delete释放吗?

​		不能，malloc /free主要为了兼容C，new和delete 完全可以取代malloc /free的。

​		malloc /free的操作对象都是必须明确大小的，而且不能用在动态类上。

​		new 和delete会自动进行类型检查和大小，malloc/free不能执行构造函数与析构函数，所以动态对象它是不行的。

​		当然从理论上说使用malloc申请的内存是可以通过delete释放的。不过一般不这样写的。而且也不能保证每个C++的运行时都能正常.



## 54、malloc与free的实现原理？

1. 在标准C库中，提供了malloc/free函数分配释放内存，**这两个函数底层是由brk、mmap、munmap这些系统调用实现的**;

2. brk是将数据段(.data)的最高地址指针_edata往高地址推,mmap是在进程的虚拟地址空间中（堆和栈中间，称为文件映射区域的地方）找一块空闲的虚拟内存。这两种方式分配的都是虚拟内存，没有分配物理内存。在第一次访问已分配的虚拟地址空间的时候，发生缺页中断，操作系统负责分配物理内存，然后建立虚拟内存和物理内存之间的映射关系；

3. malloc小于128k的内存，使用brk分配内存，将_edata往高地址推；malloc大于128k的内存，使用mmap分配内存，在堆和栈之间找一块空闲内存分配；brk分配的内存需要等到高地址内存释放以后才能释放，而mmap分配的内存可以单独释放。当最高地址空间的空闲内存超过128K（可由M_TRIM_THRESHOLD选项调节）时，执行内存紧缩操作（trim）。在上一个步骤free的时候，发现最高地址空闲内存超过128K，于是内存紧缩。

4. malloc是从堆里面申请内存，也就是说函数返回的指针是指向堆里面的一块内存。操作系统中有一个记录空闲内存地址的链表。当操作系统收到程序的申请时，就会遍历该链表，然后就寻找第一个空间大于所申请空间的堆结点，然后就将该结点从空闲结点链表中删除，并将该结点的空间分配给程序。



## 55、malloc、realloc、calloc的区别

1. malloc函数

```c++
void* malloc(unsigned int num_size);
int *p = malloc(20*sizeof(int));申请20个int类型的空间；
```

2. calloc函数

```c++
void* calloc(size_t n,size_t size);
int *p = calloc(20, sizeof(int));
```

省去了人为空间计算；malloc申请的空间的值是随机初始化的，calloc申请的空间的值是初始化为0的；

3. realloc函数

```c++
void realloc(void *p, size_t new_size);
```

给动态分配的空间分配额外的空间，用于扩充容量。



## 56、有哪些情况必须用到成员列表初始化？作用是什么？

1. 必须使用成员初始化的四种情况

   ① 当初始化一个引用成员时；

   ② 当初始化一个常量成员时；

   ③ 当调用一个基类的构造函数，而它拥有一组参数时；

   ④ 当调用一个成员类的构造函数，而它拥有一组参数时；

2. 成员初始化列表做了什么

   ① 编译器会一一操作初始化列表，以适当的顺序在构造函数之内安插初始化操作，并且在任何显示用户代码之前；

   ② list中的项目顺序是由类中的成员声明顺序决定的，不是由初始化列表的顺序决定的；



## 57、C++中新增了string，它与C语言中的 char *有什么区别吗？它是如何实现的？

​		string继承自basic_string,其实是对char*进行了封装，封装的string包含了char*数组，容量，长度等等属性。

​		string可以进行动态扩展，在每次扩展的时候另外申请一块原空间大小两倍的空间（2*n），然后将原字符串拷贝过去，并加上新增的内容。





## 58、什么是内存泄露，如何检测与避免

**内存泄露**

​		一般我们常说的内存泄漏是指**堆内存的泄漏**。堆内存是指程序从堆中分配的，大小任意的(内存块的大小可以在程序运行期决定)内存块，使用完后必须显式释放的内存。应用程序般使用malloc,、realloc、 new等函数从堆中分配到块内存，使用完后，程序必须负责相应的调用free或delete释放该内存块，否则，这块内存就不能被再次使用，我们就说这块内存泄漏了

**避免内存泄露的几种方式**

- 计数法：使用new或者malloc时，让该数+1，delete或free时，该数-1，程序执行完打印这个计数，如果不为0则表示存在内存泄露
- 一定要将基类的析构函数声明为**虚函数**
- 对象数组的释放一定要用**delete []**
- 有new就有delete，有malloc就有free，保证它们一定成对出现

**检测工具**

- Linux下可以使用**Valgrind工具**
- Windows下可以使用**CRT库**





## 59、对象复用的了解，零拷贝的了解

**对象复用**

​		对象复用其本质是一种设计模式：**Flyweight享元模式**。

​		通过将对象存储到“**对象池**”中实现对象的重复利用，这样可以避免多次创建重复对象的开销，节约系统资源。

**零拷贝**

​		零拷贝就是一种避免 CPU 将数据从一块存储拷贝到另外一块存储的技术。

​		零拷贝技术可以减少数据拷贝和共享总线操作的次数。

​			https://zhuanlan.zhihu.com/p/183861524

​		在C++中，vector的一个成员函数**emplace_back()**很好地体现了零拷贝技术，它跟push_back()函数一样可以将一个元素插入容器尾部，区别在于：**使用push_back()函数需要调用拷贝构造函数和转移构造函数，而使用emplace_back()插入的元素原地构造，不需要触发拷贝构造和转移构造**，效率更高。举个例子：

```c++
#include <vector>
#include <string>
#include <iostream>
using namespace std;

struct Person
{
    string name;
    int age;
    //初始构造函数
    Person(string p_name, int p_age): name(std::move(p_name)), age(p_age)
    {
         cout << "I have been constructed" <<endl;
    }
     //拷贝构造函数
     Person(const Person& other): name(std::move(other.name)), age(other.age)
    {
         cout << "I have been copy constructed" <<endl;
    }
     //转移构造函数
     Person(Person&& other): name(std::move(other.name)), age(other.age)
    {
         cout << "I have been moved"<<endl;
    }
};

int main()
{
    vector<Person> e;
    cout << "emplace_back:" <<endl;
    e.emplace_back("Jane", 23); //不用构造类对象

    vector<Person> p;
    cout << "push_back:"<<endl;
    p.push_back(Person("Mike",36));
    return 0;
}
//输出结果：
//emplace_back:
//I have been constructed
//push_back:
//I have been constructed
//I am being moved.

```







## 60、介绍面向对象的三大特性，并且举例说明

​	三大特性：继承、封装和多态

**（1）继承**

​	**让某种类型对象获得另一个类型对象的属性和方法。**

​	它可以使用现有类的所有功能，并在无需重新编写原来的类的情况下对这些功能进行扩展

​	常见的继承有三种方式：

​		1.	**实现继承**：指使用基类的属性和方法而无需额外编码的能力

​		2.	**接口继承**：指仅使用属性和方法的名称、但是子类必须提供实现的能力

​		3.	**可视继承**：指子窗体（类）使用基窗体（类）的外观和实现代码的能力（C++里好像不怎么用）

​		例如，将人定义为一个抽象类，拥有姓名、性别、年龄等公共属性，吃饭、睡觉、走路等公共方法，在定义一个具体的人时，就可以继承这个抽象类，既保留了公共属性和方法，也可以在此基础上扩展跳舞、唱歌等特有方法。

​	**（2）封装**

​		数据和代码捆绑在一起，避免外界干扰和不确定性访问。

​		封装，也就是**把客观事物封装成抽象的类**，并且类可以把自己的数据和方法只让可信的类或者对象操作，对不可信的进行信息隐藏，例如：将公共的数据或方法使用public修饰，而不希望被访问的数据或方法采用private修饰。

**（3）多态**

​		同一事物表现出不同事物的能力，即向不同对象发送同一消息，不同的对象在接收时会产生不同的行为**（重载实现编译时多态，虚函数实现运行时多态）**。

​		多态性是允许你将父对象设置成为和一个或更多的他的子对象相等的技术，赋值之后，父对象就可以根据当前赋值给它的子对象的特性以不同的方式运作。**简单一句话：允许将子类类型的指针赋值给父类类型的指针**

实现多态有二种方式：**覆盖**（override），**重载**（overload）。

​	覆盖：是指子类重新定义父类的虚函数的做法。

​	重载：是指允许存在多个同名函数，而这些函数的参数表不同（或许参数个数不同，或许参数类型不同，或许两者都不同）。例如：基类是一个抽象对象——人，那教师、运动员也是人，而使用这个抽象对象既可以表示教师、也可以表示运动员。



## 61、成员初始化列表的概念，为什么用它会快一些？

**成员初始化列表的概念**

​		在类的构造函数中，不在函数体内对成员变量赋值，而是在构造函数的花括号前面使用冒号和初始化列表赋值

**效率**

​		用初始化列表会快一些的原因是，**对于类型，它少了一次调用构造函数的过程，而在函数体中赋值则会多一次调用。而对于内置数据类型则没有差别。**举个例子：

```c++
#include <iostream>
using namespace std;
class A
{
public:
    A()
    {
        cout << "默认构造函数A()" << endl;
    }
    A(int a)
    {
        value = a;
        cout << "A(int "<<value<<")" << endl;
    }
    A(const A& a)
    {
        value = a.value;
        cout << "拷贝构造函数A(A& a):  "<< value << endl;
    }
    int value;
};

class B
{
public:
    B() : a(1)
    {
        b = A(2);
    }
    A a;
    A b;
};
int main()
{
    B b;
}

//输出结果：
//A(int 1)
//默认构造函数A()
//A(int 2)
```

​		从代码运行结果可以看出，在构造函数体内部初始化的对象b多了一次构造函数的调用过程，而对象a则没有。由于对象成员变量的初始化动作发生在进入构造函数之前，对于内置类型没什么影响，但**如果有些成员是类**，那么在进入构造函数之前，会先调用一次默认构造函数，进入构造函数后所做的事其实是一次赋值操作(对象已存在)，所以**如果是在构造函数体内进行赋值的话，等于是一次默认构造加一次赋值，而初始化列表只做一次赋值操作。**



## 62、C++的四种强制转换reinterpret_cast/const_cast/static_cast /dynamic_cast

**1. reinterpret_cast**

​		reinterpret_cast<type-id> (expression)

​		type-id 必须是一个指针、引用、算术类型、函数指针或者成员指针。**它可以用于类型之间进行强制转换**。



**2. const_cast**

​	const_cast<type_id> (expression)

​	**该运算符用来修改类型的const或volatile属性**。除了const 或volatile修饰之外， type_id和expression的类型是一样的。用法如下：

- 常量指针被转化成非常量的指针，并且仍然指向原来的对象
- 常量引用被转换成非常量的引用，并且仍然指向原来的对象
- const_cast一般用于修改底指针。如const char *p形式

**3. static_cast**

​		static_cast < type-id > (expression)

​		该运算符把expression转换为type-id类型，但没有运行时类型检查来保证转换的安全性。它主要有如下几种用法：

- 用于类层次结构中基类（父类）和派生类（子类）之间指针或引用引用的转换
  - 进行上行转换（把派生类的指针或引用转换成基类表示）是安全的
  - 进行下行转换（把基类指针或引用转换成派生类表示）时，由于没有动态类型检查，所以是不安全的
- 用于基本数据类型之间的转换，如把int转换成char，把int转换成enum。这种转换的安全性也要开发人员来保证。
- 把空指针转换成目标类型的空指针
- 把任何类型的表达式转换成void类型

注意：static_cast不能转换掉expression的const、volatile、或者__unaligned属性。

**4. dynamic_cast**

​		有类型检查，基类向派生类转换比较安全，但是派生类向基类转换则不太安全

​		dynamic_cast <type-id> (expression)

​		该运算符把expression转换成type-id类型的对象。type-id 必须是类的指针、类的引用或者void*

​		如果 type-id 是类指针类型，那么expression也必须是一个指针，如果 type-id 是一个引用，那么 expression 也必须是一个引用

​		dynamic_cast运算符可以在执行期决定真正的类型，也就是说expression必须是多态类型。如果下行转换是安全的（也就说，如果基类指针或者引用确实指向一个派生类对象）这个运算符会传回适当转型过的指针。如果 如果下行转换不安全，这个运算符会传回空指针（也就是说，基类指针或者引用没有指向一个派生类对象）

​		dynamic_cast主要用于类层次间的上行转换和下行转换，还可以用于类之间的交叉转换

​		在类层次间进行上行转换时，dynamic_cast和static_cast的效果是一样的

​		在进行下行转换时，dynamic_cast具有类型检查的功能，比static_cast更安全

举个例子：

```c++
#include <bits/stdc++.h>
using namespace std;

class Base
{
public:
    Base() :b(1) {}
    virtual void fun() {};
    int b;
};

class Son : public Base
{
public:
    Son() :d(2) {}
    int d;
};

int main()
{
    int n = 97;

    //reinterpret_cast
    int *p = &n;
    //以下两者效果相同
    char *c = reinterpret_cast<char*> (p); 
    char *c2 =  (char*)(p);
    cout << "reinterpret_cast输出："<< *c2 << endl;
    //const_cast
    const int *p2 = &n;
    int *p3 = const_cast<int*>(p2);
    *p3 = 100;
    cout << "const_cast输出：" << *p3 << endl;
    
    Base* b1 = new Son;
    Base* b2 = new Base;

    //static_cast
    Son* s1 = static_cast<Son*>(b1); //同类型转换
    Son* s2 = static_cast<Son*>(b2); //下行转换，不安全
    cout << "static_cast输出："<< endl;
    cout << s1->d << endl;
    cout << s2->d << endl; //下行转换，原先父对象没有d成员，输出垃圾值

    //dynamic_cast
    Son* s3 = dynamic_cast<Son*>(b1); //同类型转换
    Son* s4 = dynamic_cast<Son*>(b2); //下行转换，安全
    cout << "dynamic_cast输出：" << endl;
    cout << s3->d << endl;
    if(s4 == nullptr)
        cout << "s4指针为nullptr" << endl;
    else
        cout << s4->d << endl;
    
    
    return 0;
}
//输出结果
//reinterpret_cast输出：a
//const_cast输出：100
//static_cast输出：
//2
//-33686019
//dynamic_cast输出：
//2
//s4指针为nullptr

```



​		从输出结果可以看出，在进行下行转换时，dynamic_cast安全的，如果下行转换不安全的话其会返回空指针，这样在进行操作的时候可以预先判断。而使用static_cast下行转换存在不安全的情况也可以转换成功，但是直接使用转换后的对象进行操作容易造成错误。



## 63、C++函数调用的压栈过程

## 63.1、以例子进行讲解

从代码入手，解释这个过程：

```c++
#include <iostream>
using namespace std;

int f(int n) 
{
    cout << n << endl;
    return n;
}

void func(int param1, int param2)
{
    int var1 = param1;
    int var2 = param2;
    printf("var1=%d,var2=%d", f(var1), f(var2));//如果将printf换为cout进行输出，输出结果则刚好相反
}

int main(int argc, char* argv[])
{
    func(1, 2);
    return 0;
}
//输出结果
//2
//1
//var1=1,var2=2

```



​		当函数从入口函数main函数开始执行时，编译器会将我们操作系统的运行状态，main函数的返回地址、main的参数、mian函数中的变量、进行依次压栈；

​		当main函数开始调用func()函数时，编译器此时会将main函数的运行状态进行压栈，再将func()函数的返回地址、func()函数的参数从右到左、func()定义变量依次压栈；

​		当func()调用f()的时候，编译器此时会将func()函数的运行状态进行压栈，再将的返回地址、f()函数的参数从右到左、f()定义变量依次压栈

​		从代码的输出结果可以看出，函数f(var1)、f(var2)依次入栈，而后先执行f(var2)，再执行f(var1)，最后打印整个字符串，将栈中的变量依次弹出，最后主函数返回。



## 63.2、文字化表述

函数的调用过程：

1）从栈空间分配存储空间

2）从实参的存储空间复制值到形参栈空间

3）进行运算

​		形参在函数未调用之前都是没有分配存储空间的，在函数调用结束之后，形参弹出栈空间，清除形参空间。

​		数组作为参数的函数调用方式是地址传递，形参和实参都指向相同的内存空间，调用完成后，形参指针被销毁，但是所指向的内存空间依然存在，不能也不会被销毁。

​		当函数有多个返回值的时候，不能用普通的 return 的方式实现，需要通过传回地址的形式进行，即地址/指针传递。





## 64、写C++代码时有一类错误是 coredump ，很常见，你遇到过吗？怎么调试这个错误？

​		coredump是程序由于异常或者bug在运行时异常退出或者终止，在一定的条件下生成的一个叫做core的文件，这个core文件会记录程序在运行时的内存，寄存器状态，内存指针和函数堆栈信息等等。对这个文件进行分析可以定位到程序异常的时候对应的堆栈调用信息。

- 使用gdb命令对core文件进行调试

以下例子在Linux上编写一段代码并导致segment fault 并产生core文件

```shell
mkdir coredumpTest
vim coredumpTest.cpp
```



在编辑器内键入

```c++
#include<stdio.h>
int main(){
    int i;
    scanf("%d",i);//正确的应该是&i,这里使用i会导致segment fault
    printf("%d\n",i);
    return 0;
}
```



编译

```
g++ coredumpTest.cpp -g -o coredumpTest
```



运行

```shell
./coredumpTest
```



使用gdb调试coredump

```shell
gdb [可执行文件名] [core文件名]
```



## 65、说说移动构造函数

1. ​		我们用对象a初始化对象b，后对象a我们就不在使用了，但是对象a的空间还在呀（在析构之前），既然拷贝构造函数，实际上就是把a对象的内容复制一份到b中，那么为什么我们不能直接使用a的空间呢？这样就避免了新的空间的分配，**大大降低了构造的成本。这就是移动构造函数设计的初衷**；

2. ​        **拷贝构造函数中，对于指针，我们一定要采用深层复制，而移动构造函数中，对于指针，我们采用浅层复制。**浅层复制之所以危险，是因为两个指针共同指向一片内存空间，若第一个指针将其释放，另一个指针的指向就不合法了。

   ​		所以我们只要避免第一个指针释放空间就可以了。避免的方法就是将第一个指针（比如a->value）置为NULL，这样在调用析构函数的时候，由于有判断是否为NULL的语句，所以析构a的时候并不会回收a->value指向的空间；

3. ​       **移动构造函数的参数和拷贝构造函数不同，拷贝构造函数的参数是一个左值引用，但是移动构造函数的初值是一个右值引用。**意味着，移动构造函数的参数是一个右值或者将亡值的引用。也就是说，只用用一个右值，或者将亡值初始化另一个对象的时候，才会调用移动构造函数。而那个move语句，就是将一个左值变成一个将亡值。





## 66、C++中将临时变量作为返回值时的处理过程

​		首先需要明白一件事情，临时变量，在函数调用过程中是被压到程序进程的栈中的，当函数退出时，临时变量出栈，即临时变量已经被销毁，临时变量占用的内存空间没有被清空，但是可以被分配给其他变量，所以有可能在函数退出时，该内存已经被修改了，对于临时变量来说已经是没有意义的值了

​		C语言里规定：16bit程序中，返回值保存在ax寄存器中，32bit程序中，返回值保持在eax寄存器中，如果是64bit返回值，edx寄存器保存高32bit，eax寄存器保存低32bit

​		由此可见，函数调用结束后，**返回值被临时存储到寄存器中**，并没有放到堆或栈中，也就是说与内存没有关系了。当退出函数的时候，临时变量可能被销毁，但是返回值却被放到寄存器中与临时变量的生命周期没有关系如果我们需要返回值，一般使用赋值语句就可以了。





## 67、如何获得结构成员相对于结构开头的字节偏移量

使用<stddef.h>头文件中的，offsetof宏。

举个例子：

```c++
#include <iostream>
#include <stddef.h>
using namespace std;

struct  S
{
	int x;
	char y;
	int z;
	double a;
};
int main()
{
	cout << offsetof(S, x) << endl; // 0
	cout << offsetof(S, y) << endl; // 4
	cout << offsetof(S, z) << endl; // 8
	cout << offsetof(S, a) << endl; // 12
	return 0;
}
```



在Visual Studio 2019 + Win10 下的输出情况如下

```c++
cout << offsetof(S, x) << endl; // 0
cout << offsetof(S, y) << endl; // 4
cout << offsetof(S, z) << endl; // 8
cout << offsetof(S, a) << endl; // 16 这里是 16的位置，因为 double是8字节，需要找一个8的倍数对齐，
```

当然了，如果加上 #pragma pack(4) 指定4字节对齐方式就可以了。



```c++
#pragma pack(4)
struct  S
{
	int x;
	char y;
	int z;
	double a;
};
void test02()
{
    cout << offsetof(S, x) << endl; // 0
    cout << offsetof(S, y) << endl; // 4
    cout << offsetof(S, z) << endl; // 8
    cout << offsetof(S, a) << endl; // 12
｝
```

S结构体中各个数据成员的内存空间划分如下所示，需要注意内存对齐.

![1678244547885](1678244547885.png)





## 68、静态类型和动态类型，静态绑定和动态绑定的介绍

- 静态类型：对象在声明时采用的类型，在**编译期**既已确定；

- 动态类型：通常是指一个指针或引用目前所指对象的类型，是在**运行期**决定的；

- 静态绑定：绑定的是静态类型，所对应的函数或属性依赖于对象的静态类型，发生在**编译期**；

- 动态绑定：绑定的是动态类型，所对应的函数或属性依赖于对象的动态类型，发生在**运行期**；

  从上面的定义也可以看出，**非虚函数一般都是静态绑定，而虚函数都是动态绑定**（如此才可实现多态性）。 举个例子：

```c++
#include <iostream>
using namespace std;

class A
{
public:
	/*virtual*/ void func() { std::cout << "A::func()\n"; }
};
class B : public A
{
public:
	void func() { std::cout << "B::func()\n"; }
};
class C : public A
{
public:
	void func() { std::cout << "C::func()\n"; }
};
int main()
{
	C* pc = new C(); //pc的静态类型是它声明的类型C*，动态类型也是C*；
	B* pb = new B(); //pb的静态类型和动态类型也都是B*；
	A* pa = pc;      //pa的静态类型是它声明的类型A*，动态类型是pa所指向的对象pc的类型C*；
	pa = pb;         //pa的动态类型可以更改，现在它的动态类型是B*，但其静态类型仍是声明时候的A*；
	C *pnull = NULL; //pnull的静态类型是它声明的类型C*,没有动态类型，因为它指向了NULL；
    
    pa->func();      //A::func() pa的静态类型永远都是A*，不管其指向的是哪个子类，都是直接调用A::func()；
	pc->func();      //C::func() pc的动、静态类型都是C*，因此调用C::func()；
	pnull->func();   //C::func() 不用奇怪为什么空指针也可以调用函数，因为这在编译期就确定了，和指针空不空没关系；
	return 0;
}
```

如果将A类中的virtual注释去掉，则运行结果是：

```c++
pa->func();      //B::func() 因为有了virtual虚函数特性，pa的动态类型指向B*，因此先在B中查找，找到后直接调用；
pc->func();      //C::func() pc的动、静态类型都是C*，因此也是先在C中查找；
pnull->func();   //空指针异常，因为是func是virtual函数，因此对func的调用只能等到运行期才能确定，然后才发现pnull是空指针；
```

在上面的例子中，

- 如果基类A中的func不是virtual函数，那么不论pa、pb、pc指向哪个子类对象，对func的调用都是在定义pa、pb、pc时的静态类型决定，早已在编译期确定了。
- 同样的空指针也能够直接调用no-virtual函数而不报错（这也说明一定要做空指针检查啊！），因此静态绑定不能实现多态；
- 如果func是虚函数，那所有的调用都要等到运行时根据其指向对象的类型才能确定，比起静态绑定自然是要有性能损失的，但是却能实现多态特性；

**本文代码里都是针对指针的情况来分析的，但是对于引用的情况同样适用。**

至此总结一下静态绑定和动态绑定的区别：

- 静态绑定发生在编译期，动态绑定发生在运行期；
- 对象的动态类型可以更改，但是静态类型无法更改；
- 要想实现动态，必须使用动态绑定；
- 在继承体系中只有虚函数使用的是动态绑定，其他的全部是静态绑定；

**建议：**

​		绝对不要重新定义继承而来的非虚(non-virtual)函数（《Effective C++ 第三版》条款36），因为这样导致函数调用由对象声明时的静态类型确定了，而和对象本身脱离了关系，没有多态，也这将给程序留下不可预知的隐患和莫名其妙的BUG；另外，在动态绑定也即在virtual函数中，要注意默认参数的使用。当缺省参数和virtual函数一起使用的时候一定要谨慎，不然出了问题怕是很难排查。 看下面的代码：

```c++
#include <iostream>
using namespace std;

class E
{
public:
	virtual void func(int i = 0)
	{
		std::cout << "E::func()\t" << i << "\n";
	}
};
class F : public E
{
public:
	virtual void func(int i = 1)
	{
		std::cout << "F::func()\t" << i << "\n";
	}
};

void test2()
{
	F* pf = new F();
	E* pe = pf;
	pf->func(); //F::func() 1  正常，就该如此；
	pe->func(); //F::func() 0  哇哦，这是什么情况，调用了子类的函数，却使用了基类中参数的默认值！
}
int main()
{
	test2();
	return 0;
}

```



## 69、引用是否能实现动态绑定，为什么可以实现？

​		可以。

​		引用在创建的时候必须初始化，在访问虚函数时，编译器会根据其所绑定的对象类型决定要调用哪个函数。注意只能调用虚函数。

举个例子：

```c++
#include <iostream>
using namespace std;

class Base 
{
public:
	virtual void  fun()
	{
		cout << "base :: fun()" << endl;
	}
};

class Son : public Base
{
public:
	virtual void  fun()
	{
		cout << "son :: fun()" << endl;
	}
	void func()
	{
		cout << "son :: not virtual function" <<endl;
	}
};

int main()
{
	Son s;
	Base& b = s; // 基类类型引用绑定已经存在的Son对象，引用必须初始化
	s.fun(); //son::fun()
	b.fun(); //son :: fun()
	return 0;
}
```

​		**需要说明的是虚函数才具有动态绑定**，上面代码中，Son类中还有一个非虚函数func()，这在b对象中是无法调用的，如果使用基类指针来指向子类也是一样的。



## 70、全局变量和局部变量有什么区别？

​		**生命周期不同**：

​				全局变量随主程序创建和创建，随主程序销毁而销毁；

​				局部变量在局部函数内部，甚至局部循环体等内部存在，退出就不存在；

​		**使用方式不同**：

​				通过声明后全局变量在程序的各个部分都可以用到；局部变量分配在堆栈区，只能在局部使用。

​		操作系统和编译器通过内存分配的位置可以区分两者，全局变量分配在全局数据段并且在程序开始运行的时候被加载。局部变量则分配在堆栈里面 。





## 71、指针加减计算要注意什么？

​		**指针加减本质是对其所指地址的移动，移动的步长跟指针的类型是有关系的，因此在涉及到指针加减运算需要十分小心，加多或者减多都会导致指针指向一块未知的内存地址，如果再进行操作就会很危险。**

举个例子

```c++
#include <iostream>
using namespace std;

int main()
{
	int *a, *b, c;
	a = (int*)0x500;
	b = (int*)0x520;
	c = b - a;
	printf("%d\n", c); // 8
	a += 0x020;
	c = b - a;
	printf("%d\n", c); // -24
	return 0;
}

```

​		首先变量a和b都是以16进制的形式初始化，将它们转成10进制分别是1280（5*16^2=1280）和1312（5*16^2+2*16=1312)， 那么它们的差值为32，也就是说a和b所指向的地址之间间隔32个位，但是考虑到是int类型占4位，所以c的值为32/4=8

​		a自增16进制0x20之后，其实际地址变为1280 + 2*16*4 = 1408，（因为一个int占4位，所以要乘4），这样它们的差值就变成了1312 - 1280 = -96，所以c的值就变成了-96/4 = -24

​		遇到指针的计算，**需要明确的是指针每移动一位，它实际跨越的内存间隔是指针类型的长度，建议都转成10进制**。





## 72、 怎样判断两个浮点数是否相等？

​		对两个浮点数判断大小和是否相等不能直接用==来判断，会出错！明明相等的两个数比较反而是不相等！对于两个浮点数比较只能通过相减并与预先设定的精度比较，记得要取绝对值！浮点数与0的比较也应该注意。与浮点数的表示方式有关。





## 73、方法调用的原理（栈，汇编）

​		1. 机器用栈来传递过程参数、存储返回信息、保存寄存器用于以后恢复，以及本地存储。而为单个过程分配的那部分栈称为帧栈；帧栈可以认为是程序栈的一段，它有两个端点，一个标识起始地址，一个标识着结束地址，两个指针结束地址指针esp，开始地址指针ebp;

​		2. 由一系列栈帧构成，这些栈帧对应一个过程，而且每一个栈指针+4的位置存储函数返回地址；每一个栈帧都建立在调用者的下方，当被调用者执行完毕时，这一段栈帧会被释放。由于栈帧是向地址递减的方向延伸，因此如果我们将栈指针减去一定的值，就相当于给栈帧分配了一定空间的内存。如果将栈指针加上一定的值，也就是向上移动，那么就相当于压缩了栈帧的长度，也就是说内存被释放了。

​		3. 过程实现：

​				① 备份原来的帧指针，调整当前的栈帧指针到栈指针位置；

​				② 建立起来的栈帧就是为被调用者准备的，当被调用者使用栈帧时，需要给临时变量分配预留内存；

​				③ 使用建立好的栈帧，比如读取和写入，一般使用mov，push以及pop指令等等。

​				④ 恢复被调用者寄存器当中的值，这一过程其实是从栈帧中将备份的值再恢复到寄存器，不过此时这些值可能已经不在栈顶了

​				⑤ 恢复被调用者寄存器当中的值，这一过程其实是从栈帧中将备份的值再恢复到寄存器，不过此时这些值可能已经不在栈顶了。

​				⑥ 释放被调用者的栈帧，释放就意味着将栈指针加大，而具体的做法一般是直接将栈指针指向帧指针，因此会采用类似下面的汇编代码处理。

​				⑦ 恢复调用者的栈帧，恢复其实就是调整栈帧两端，使得当前栈帧的区域又回到了原始的位置。

​				⑧ 弹出返回地址，跳出当前过程，继续执行调用者的代码。

​		4. 过程调用和返回指令

​				① call指令

​				② leave指令

​				③ ret指令



## 74、C++中的指针参数传递和引用参数传递有什么区别？底层原理你知道吗？

**1)** 	**指针参数传递本质上是值传递**，它所传递的是一个地址值。

​		值传递过程中，被调函数的形式参数作为被调函数的局部变量处理，会在栈中开辟内存空间以存放由主调函数传递进来的实参值，从而形成了实参的一个副本（替身）。

​		值传递的特点是，被调函数对形式参数的任何操作都是作为局部变量进行的，不会影响主调函数的实参变量的值（形参指针变了，实参指针不会变）。

**2)** 	引用参数传递过程中，被调函数的形式参数也作为局部变量在栈中开辟了内存空间，但是这时存放的是由主调函数放进来的实参变量的地址。

​		被调函数对形参（本体）的任何操作都被处理成间接寻址，即通过栈中存放的地址访问主调函数中的实参变量（根据别名找到主调函数中的本体）。

​		因此，被调函数对形参的任何操作都会影响主调函数中的实参变量。

**3)** 	引用传递和指针传递是不同的，虽然他们都是在被调函数栈空间上的一个局部变量，但是任何对于引用参数的处理都会通过一个间接寻址的方式操作到主调函数中的相关变量。

​		而对于指针传递的参数，如果改变被调函数中的指针地址，它将应用不到主调函数的相关变量。如果想通过指针参数传递来改变主调函数中的相关变量（地址），那就得使用指向指针的指针或者指针引用。

**4)** 		从编译的角度来讲，程序在编译时分别将指针和引用添加到符号表上，符号表中记录的是变量名及变量所对应地址。

​		指针变量在符号表上对应的地址值为指针变量的地址值，而引用在符号表上对应的地址值为引用对象的地址值（与实参名字不同，地址相同）。

​		符号表生成之后就不会再改，因此指针可以改变其指向的对象（指针变量中的值可以改），而引用对象则不能修改。





## 75、类如何实现只能静态分配和只能动态分配

1. 前者是把new、delete运算符重载为private属性。后者是把构造、析构函数设为protected属性，再用子类来动态创建

2. 建立类的对象有两种方式：

   ​	① 静态建立，静态建立一个类对象，就是由编译器为对象在栈空间中分配内存；

   ​	② 动态建立，A *p = new A();动态建立一个类对象，就是使用new运算符为对象在堆空间中分配内存。这个过程分为两步，第一步执行operator new()函数，在堆中搜索一块内存并进行分配；第二步调用类构造函数构造对象；

3. 只有使用new运算符，对象才会被建立在堆上，因此只要限制new运算符就可以实现类对象只能建立在栈上，可以将new运算符设为私有。





## 76、如果想将某个类用作基类，为什么该类必须定义而非声明？

​		派生类中包含并且可以使用它从基类继承而来的成员，为了使用这些成员，派生类必须知道他们是什么。

所以必须定义而非声明。



## 77、 继承机制中对象之间如何转换？指针和引用之间如何转换？

1. ​		将派生类指针或引用转换为基类的指针或引用被称为向上类型转换，向上类型转换会自动进行，而且向上类型转换是安全的。
2. ​        将基类指针或引用转换为派生类指针或引用被称为向下类型转换，向下类型转换不会自动进行，因为一个基类对应几个派生类，所以向下类型转换时不知道对应哪个派生类，所以在向下类型转换时必须加动态类型识别技术。RTTI技术，用dynamic_cast进行向下类型转换。





## 78、知道C++中的组合吗？它与继承相比有什么优缺点吗？

**一：继承**

​		继承是**Is a** 的关系，比如说Student继承Person,则说明Student is a Person。继承的优点是子类可以重写父类的方法来方便地实现对父类的扩展。

继承的缺点有以下几点：

​		①：父类的内部细节对子类是可见的。

​		②：子类从父类继承的方法在编译时就确定下来了，所以无法在运行期间改变从父类继承的方法的行为。

​		③：如果对父类的方法做了修改的话（比如增加了一个参数），则子类的方法必须做出相应的修改。所以说子类与父类是一种高耦合，违背了面向对象思想。

**二：组合**

​		组合也就是设计类的时候把要组合的类的对象加入到该类中作为自己的成员变量。

组合的优点：

​		①：当前对象只能通过所包含的那个对象去调用其方法，所以所包含的对象的内部细节对当前对象时不可见的。

​		②：当前对象与包含的对象是一个低耦合关系，如果修改包含对象的类中代码不需要修改当前对象类的代码。

​		③：当前对象可以在运行时动态的绑定所包含的对象。可以通过set方法给所包含对象赋值。

组合的缺点：

​		①：容易产生过多的对象。

​		②：为了能组合多个对象，必须仔细对接口进行定义。





## 79、函数指针？

**1) 什么是函数指针?**

​		函数指针指向的是特殊的数据类型，函数的类型是由其返回的数据类型和其参数列表共同决定的，而函数的名称则不是其类型的一部分。

​		一个具体函数的名字，如果后面不跟调用符号(即括号)，则该名字就是该函数的指针(注意：大部分情况下，可以这么认为，但这种说法并不很严格)。

**2) 函数指针的声明方法**

​		int (*pf)(const int&, const int&); (1)

​		上面的pf就是一个函数指针，指向所有返回类型为int，并带有两个const int&参数的函数。注意*pf两边的括号是必须的，否则上面的定义就变成了：

​		int *pf(const int&, const int&); (2)

​		而这声明了一个函数pf，其返回类型为int *， 带有两个const int&参数。

**3) 为什么有函数指针**

​		函数与数据项相似，函数也有地址。我们希望在同一个函数中通过使用相同的形参在不同的时间使用产生不同的效果。

**4) 一个函数名就是一个指针，它指向函数的代码。**

​		一个函数地址是该函数的进入点，也就是调用函数的地址。函数的调用可以通过函数名，也可以通过指向函数的指针来调用。函数指针还允许将函数作为变元传递给其他函数；

**5) 两种方法赋值：**

​		指针名 = 函数名； 指针名 = &函数名

​		https://blog.csdn.net/m0_46569169/article/details/124318184

​		https://blog.csdn.net/wangleizhenshuai/article/details/108320792





## 80、说一说你理解的内存对齐以及原因

​		1、 分配内存的顺序是按照声明的顺序。

​		2、 每个变量相对于起始位置的偏移量必须是该变量类型大小的整数倍，不是整数倍空出内存，直到偏移量是整数倍为止。

​		3、 最后整个结构体的大小必须是里面变量类型最大值的整数倍。

​	添加了#pragma pack(n)后规则就变成了下面这样：

​		1、 偏移量要是n和当前变量大小中较小值的整数倍

​		2、 整体大小要是n和最大变量大小中较小值的整数倍

​		3、 n值必须为1,2,4,8…，为其他值时就按照默认的分配规则





## 81、 结构体变量比较是否相等

1. 重载了 “==” 操作符
2. 元素的话，一个个比；
3. 指针直接比较，如果保存的是同一个实例地址，则(p1==p2)为真；

```c++
struct foo {

  int a;
  int b;

  bool operator==(const foo& rhs) *//* *操作运算符重载*

  {
    return( a == rhs.a) && (b == rhs.b);
  }
};
```





## 82、define、const、typedef、inline的使用方法？他们之间有什么区别？

一、const与#define的区别：

1. const定义的常量是变量带类型，而#define定义的只是个常数不带类型；
2. define只在预处理阶段起作用，简单的文本替换，而const在编译、链接过程中起作用；
3. define只是简单的字符串替换没有类型检查。而const是有数据类型的，是要进行判断的，可以避免一些低级错误；
4. define预处理后，占用代码段空间，const占用数据段空间；
5. const不能重定义，而define可以通过#undef取消某个符号的定义，进行重定义；
6. define独特功能，比如可以用来防止文件重复引用。

二、#define和别名typedef的区别

1. 执行时间不同，typedef在编译阶段有效，typedef有类型检查的功能；#define是宏定义，发生在预处理阶段，不进行类型检查；
2. 功能差异，typedef用来定义类型的别名，定义与平台无关的数据类型，与struct的结合使用等。#define不只是可以为类型取别名，还可以定义常量、变量、编译开关等。
3. 作用域不同，#define没有作用域的限制，只要是之前预定义过的宏，在以后的程序中都可以使用。而typedef有自己的作用域。

三、 define与inline的区别

1. \#define是关键字，inline是函数；
2. 宏定义在预处理阶段进行文本替换，inline函数在编译阶段进行替换；
3. inline函数有类型检查，相比宏定义比较安全；





## 83、你知道printf函数的实现原理是什么吗？

​		在C/C++中，对函数参数的扫描是从后向前的。

​		C/C++的函数参数是通过压入堆栈的方式来给函数传参数的（堆栈是一种先进后出的数据结构），最先压入的参数最后出来，在计算机的内存中，数据有2块，一块是堆，一块是栈（函数参数及局部变量在这里），而栈是从内存的高地址向低地址生长的，控制生长的就是堆栈指针了，最先压入的参数是在最上面，就是说在所有参数的最后面，最后压入的参数在最下面，结构上看起来是第一个，所以最后压入的参数总是能够被函数找到，因为它就在堆栈指针的上方。

​		printf的第一个被找到的参数就是那个字符指针，就是被双引号括起来的那一部分，函数通过判断字符串里控制参数的个数来判断参数个数及数据类型，通过这些就可算出数据需要的堆栈指针的偏移量了，下面给出printf("%d,%d",a,b);（其中a、b都是int型的）的汇编代码





## 84、为什么模板类一般都是放在一个h文件中

1. ​    模板定义很特殊。由template<…>处理的任何东西都意味着编译器在当时不为它分配存储空间，它一直处于等待状态直到被一个模板实例告知。在编译器和连接器的某一处，有一机制能去掉指定模板的多重定义。

   ​	所以为了容易使用，几乎总是在头文件中放置全部的模板声明和定义。

2. ​    在分离式编译的环境下，编译器编译某一个.cpp文件时并不知道另一个.cpp文件的存在，也不会去查找（当遇到未决符号时它会寄希望于连接器）。这种模式在没有模板的情况下运行良好，但遇到模板时就傻眼了，因为模板仅在需要的时候才会实例化出来。

   ​		所以，当编译器只看到模板的声明时，它不能实例化该模板，只能创建一个具有外部连接的符号并期待连接器能够将符号的地址决议出来。

   ​		然而当实现该模板的.cpp文件中没有用到模板的实例时，编译器懒得去实例化，所以，整个工程的.obj中就找不到一行模板实例的二进制代码，于是连接器也黔驴技穷了。





## 85、C++中类成员的访问权限和继承权限问题

1. 三种访问权限

   ​	① public: 用该关键字修饰的成员表示公有成员，该成员不仅可以在类内可以被 访问，在类外也是可以被访问的，是类对外提供的可访问接口；

   ​	② private: 用该关键字修饰的成员表示私有成员，该成员仅在类内可以被访问，在类体外是隐藏状态；

   ​	③ protected: 用该关键字修饰的成员表示保护成员，保护成员在类体外同样是隐藏状态，但是对于该类的派生类来说，相当于公有成员，在派生类中可以被访问。

2. 三种继承方式

   ​	① 若继承方式是public，基类成员在派生类中的访问权限保持不变，也就是说，基类中的成员访问权限，在派生类中仍然保持原来的访问权限；

   ​	② 若继承方式是private，基类所有成员在派生类中的访问权限都会变为私有(private)权限；

   ​	③ 若继承方式是protected，基类的共有成员和保护成员在派生类中的访问权限都会变为保护(protected)权限，私有成员在派生类中的访问权限仍然是私有(private)权限。







## 86、cout和printf有什么区别？

​		cout<<是一个函数，cout<<后可以跟不同的类型是因为cout<<已存在针对各种类型数据的重载，所以会自动识别数据的类型。

​		输出过程会首先将输出字符放入缓冲区，然后输出到屏幕。

​		cout是有缓冲输出:

```c++
cout < < "abc " < <endl; 
或cout < < "abc\n "; cout < <flush; //这两个才是一样的.
```

​		flush立即强迫缓冲输出。

​		printf是行缓冲输出，不是无缓冲输出





## 87、你知道重载运算符吗？

​	1、 我们只能重载已有的运算符，而无权发明新的运算符；对于一个重载的运算符，其优先级和结合律与内置类型一致才可以；不能改变运算符操作数个数；

​	2、 两种重载方式：成员运算符和非成员运算符，成员运算符比非成员运算符少一个参数；下标运算符、箭头运算符必须是成员运算符；

​	3、 引入运算符重载，是为了实现类的多态性；

​	4、 当重载的运算符是成员函数时，this绑定到左侧运算符对象。成员运算符函数的参数数量比运算符对象的数量少一个；至少含有一个类类型的参数；

​	5、 从参数的个数推断到底定义的是哪种运算符，当运算符既是一元运算符又是二元运算符（+，-，*，&）；

​	6、 下标运算符必须是成员函数，下标运算符通常以所访问元素的引用作为返回值，同时最好定义下标运算符的常量版本和非常量版本；

​	7、 箭头运算符必须是类的成员，解引用通常也是类的成员；重载的箭头运算符必须返回类的指针；





## 88、当程序中有函数重载时，函数的匹配原则和顺序是什么？

1. 名字查找
2. 确定候选函数
3. 寻找最佳匹配



## 89、定义和声明的区别

​		**如果是指变量的声明和定义：** 从编译原理上来说，声明是仅仅告诉编译器，有个某类型的变量会被使用，但是编译器并不会为它分配任何内存。而定义就是分配了内存。

​		**如果是指函数的声明和定义：** 声明：一般在头文件里，对编译器说：这里我有一个函数叫function() 让编译器知道这个函数的存在。 定义：一般在源文件里，具体就是函数的实现过程写明函数体。





## 90、全局变量和static变量的区别

1、全局变量（外部变量）的说明之前再冠以static就构成了静态的全局变量。

​		**全局变量本身就是静态存储方式，静态全局变量当然也是静态存储方式。**

​		**这两者在存储方式上并无不同**。这两者的区别在于非静态全局变量的作用域是整个源程序，当一个源程序由多个原文件组成时，**非静态的全局变量在各个源文件中都是有效的**。

​		**而静态全局变量则限制了其作用域，即只在定义该变量的源文件内有效，在同一源程序的其它源文件中不能使用它**。由于静态全局变量的作用域限于一个源文件内，只能为该源文件内的函数公用，因此可以避免在其他源文件中引起错误。

​		static全局变量与普通的全局变量的区别是static全局变量只初始化一次，防止在其他文件单元被引用。

2.static函数与普通函数有什么区别？ 

​		static函数与普通的函数作用域不同。尽在本文件中。只在当前源文件中使用的函数应该说明为内部函数（static），内部函数应该在当前源文件中说明和定义。

​		对于可在当前源文件以外使用的函数应该在一个头文件中说明，要使用这些函数的源文件要包含这个头文件。 static函数与普通函数最主要区别是static函数在内存中只有一份，普通静态函数在每个被调用中维持一份拷贝程序的局部变量存在于（堆栈）中，全局变量存在于（静态区）中，动态申请数据存在于（堆)。



## 91、 静态成员与普通成员的区别是什么？

1. 生命周期

   ​	静态成员变量从类被加载开始到类被卸载，一直存在；

   ​	普通成员变量只有在类创建对象后才开始存在，对象结束，它的生命期结束；

2. 共享方式

   ​	静态成员变量是全类共享；普通成员变量是每个对象单独享用的；

3. 定义位置

   ​	普通成员变量存储在栈或堆中，而静态成员变量存储在静态全局区；

4. 初始化位置

   ​	普通成员变量在类中初始化；静态成员变量在类外初始化；

5. 默认实参

   ​	可以使用静态成员变量作为默认实参.





## 92、说一下你理解的 ifdef endif代表着什么？

1. ​    一般情况下，源程序中所有的行都参加编译。但是有时希望对其中一部分内容只在满足一定条件才进行编译，也就是对一部分内容指定编译的条件，这就是“**条件编译**”。有时，希望当满足某条件时对一组语句进行编译，而当条件不满足时则编译另一组语句。
2.    条件编译命令最常见的形式为：

```c++
#ifdef 标识符  
程序段1  
#else  
程序段2  
#endif
```



​		它的作用是：当标识符已经被定义过(一般是用#define命令定义)，则对程序段1进行编译，否则编译程序段2。 其中#else部分也可以没有，即：

```
#ifdef  
程序段1  
#endif
```

3. ​    在一个大的软件工程里面，可能会有多个文件同时包含一个头文件，当这些文件编译链接成一个可执行文件上时，就会出现大量“重定义”错误。

   在头文件中使用#define、#ifndef、#ifdef、#endif能避免头文件重定义。





## 93、隐式转换，如何消除隐式转换？

1、C++的基本类型中并非完全的对立，部分数据类型之间是可以进行隐式转换的。**所谓隐式转换，是指不需要用户干预，编译器私下进行的类型转换行为**。很多时候用户可能都不知道进行了哪些转换

2、C++面向对象的多态特性，就是通过父类的类型实现对子类的封装。**通过隐式转换，你可以直接将一个子类的对象使用父类的类型进行返回。**在比如，数值和布尔类型的转换，整数和浮点数的转换等。某些方面来说，隐式转换给C++程序开发者带来了不小的便捷。C++是一门强类型语言，类型的检查是非常严格的。

3、 基本数据类型 基本数据类型的转换以取值范围的作为转换基础（保证精度不丢失）。隐式转换发生在从小->大的转换中。比如从char转换为int。从int->long。**自定义对象子类对象可以隐式的转换为父类对象。**

4、 C++中提供了explicit关键字，**在构造函数声明的时候加上explicit关键字，能够禁止隐式转换。**

5、如果构造函数只接受一个参数，则它实际上定义了转换为此类类型的隐式转换机制。可以通过将构造函数声明为explicit加以制止隐式类型转换，**关键字explicit只对一个实参的构造函数有效，需要多个实参的构造函数不能用于执行隐式转换，所以无需将这些构造函数指定为explicit。**





## 94、C++如何处理多个异常的？

1. ​     C++中的异常情况： 语法错误（编译错误）：比如变量未定义、括号不匹配、关键字拼写错误等等编译器在编译时能发现的错误，这类错误可以及时被编译器发现，而且可以及时知道出错的位置及原因，方便改正。 运行时错误：比如数组下标越界、系统内存不足等等。这类错误不易被程序员发现，它能通过编译且能进入运行，但运行时会出错，导致程序崩溃。为了有效处理程序运行时错误，C++中引入异常处理机制来解决此问题。
2. ​     C++异常处理机制： 异常处理基本思想：**执行一个函数的过程中发现异常，可以不用在本函数内立即进行处理， 而是抛出该异常，让函数的调用者直接或间接处理这个问题**。 C++异常处理机制由3个模块组成：try(检查)、throw(抛出)、catch(捕获) 抛出异常的语句格式为：throw 表达式；如果try块中程序段发现了异常则抛出异常。

```c++
try  {  可能抛出异常的语句；（检查） try 
{ 
可能抛出异常的语句；（检查） 
} 
catch（类型名[形参名]）//捕获特定类型的异常 
{ 
//处理1； 
} 
catch（类型名[形参名]）//捕获特定类型的异常 
{ 
//处理2； 
} 
catch（…）//捕获所有类型的异常 
{ 
}
```





## 95、如何在不使用额外空间的情况下，交换两个数？你有几种方法

```c++
1)  算术

x = x + y;
y = x - y;
x = x - y; 

2)  异或

x = x^y;// 只能对int,char..
y = x^y;
x = x^y;
```



## 96、你知道strcpy和memcpy的区别是什么吗？

1、复制的内容不同。**strcpy只能复制字符串，而memcpy可以复制任意内容**，例如字符数组、整型、结构体、类等。 

2、复制的方法不同。strcpy不需要指定长度，它遇到被复制字符的串结束符"\0"才结束，所以容易溢出。memcpy则是根据其第3个参数决定复制的长度。 

3、用途不同。通常在复制字符串时用strcpy，而需要复制其他类型数据时则一般用memcpy





## 97、程序在执行int main(int argc, char *argv[])时的内存结构，你了解吗？

​		参数的含义是程序在命令行下运行的时候，需要输入argc 个参数，每个参数是以char 类型输入的，依次存在数组里面，数组是 argv[]，所有的参数在指针

​		char * 指向的内存中，数组的中元素的个数为 argc 个，第一个参数为程序的名称。



## 98、volatile关键字的作用？

​		**volatile 关键字是一种类型修饰符，用它声明的类型变量表示可以被某些编译器未知的因素更改**，比如：操作系统、硬件或者其它线程等。遇到这个关键字声明的变量，编译器对访问该变量的代码就不再进行优化，从而可以提供对特殊地址的稳定访问。

​		声明时语法：int volatile vInt; 当要求使用 volatile 声明的变量的值的时候，系统总是重新从它所在的内存读取数据，即使它前面的指令刚刚从该处读取过数据。而且读取的数据立刻被保存。

volatile用在如下的几个地方：

1. 中断服务程序中修改的供其它程序检测的变量需要加volatile；
2. 多任务环境下各任务间共享的标志应该加volatile；
3. 存储器映射的硬件寄存器通常也要加volatile说明，因为每次对它的读写都可能由不同意义；



## 99、如果有一个空类，它会默认添加哪些函数？

```c++
1)  Empty(); // 缺省构造函数//
2)  Empty( const Empty& ); // 拷贝构造函数//
3)  ~Empty(); // 析构函数//
4)  Empty& operator=( const Empty& ); // 赋值运算符//
```





## 100、C++中标准库是什么？

1. C++ 标准库可以分为两部分：

   ​	标准函数库： 这个库是由通用的、独立的、不属于任何类的函数组成的。函数库继承自 C 语言。

   ​	面向对象类库： 这个库是类及其相关函数的集合。

2. 输入/输出 I/O、字符串和字符处理、数学、时间、日期和本地化、动态分配、其他、宽字符函数

3. 标准的 C++ I/O 类、String 类、数值类、STL 容器类、STL 算法、STL 函数对象、STL 迭代器、STL 分配器、本地化库、异常处理类、杂项支持库。



## 101、你知道const char* 与string之间的关系是什么吗？

1. string 是c++标准库里面其中一个，封装了对字符串的操作，实际操作过程我们可以用const char*给string类初始化
2. 三者的转化关系如下所示

```c++
a)  string转const char* 

string s = “abc”; 

const char* c_s = s.c_str(); 

b)  const char* 转string，直接赋值即可 

const char* c_s = “abc”; 
 string s(c_s); 

c)  string 转char* 
 string s = “abc”; 
 char* c; 
 const int len = s.length(); 
 c = new char[len+1]; 
 strcpy(c,s.c_str()); 

d)  char* 转string 
 char* c = “abc”; 
 string s(c); 

e)  const char* 转char* 
 const char* cpc = “abc”; 
 char* pc = new char[strlen(cpc)+1]; 
 strcpy(pc,cpc);

f)  char* 转const char*，直接赋值即可 
 char* pc = “abc”; 
 const char* cpc = pc;

```





## 102、你什么情况用指针当参数，什么时候用引用，为什么？

1. 使用引用参数的主要原因有两个：

   ​	程序员能**修改**调用函数中的数据对象

   ​	通过传递引用而不是整个数据–对象，可以**提高程序的运行速度**

2. 一般的原则： 

   ​	对于使用引用的值而不做修改的函数：

   ​	如果数据对象很小，如内置数据类型或者小型结构，则按照值传递；

   ​	如果数据对象是数组，则使用指针（唯一的选择），并且指针声明为指向const的指针；

   ​	如果数据对象是较大的结构，则使用const指针或者引用，已提高程序的效率。这样可以节省结构所需的时间和空间；

   ​	如果数据对象是类对象，则使用const引用（传递类对象参数的标准方式是按照引用传递）；

3. 对于修改函数中数据的函数：

   ​	如果数据是内置数据类型，则使用指针

   ​	如果数据对象是结构，则使用引用或者指针

   ​	如果数据是类对象，则使用引用

   ​	也有一种说法认为：“如果数据对象是数组，则只能使用指针”，这是不对的，比如

```c++
template<typename T, int N>
void func(T (&a)[N])
{
    a[0] = 2;
}

int main()
{
    int a[] = { 1, 2, 3 };
    func(a);
    cout << a[0] << endl;
    return 0;
}

```





## 103、你知道静态绑定和动态绑定吗？讲讲？

1. 对象的静态类型：对象在声明时采用的类型。是在**编译期**确定的。
2. 对象的动态类型：目前所指对象的类型。是在**运行期**决定的。对象的动态类型可以更改，但是静态类型无法更改。
3. 静态绑定：绑定的是对象的静态类型，某特性（比如函数依赖于对象的静态类型，发生在编译期。)
4. 动态绑定：绑定的是对象的动态类型，某特性（比如函数依赖于对象的动态类型，发生在运行期。)



## 104、如何设计一个计算仅单个子类的对象个数？

​	1、为类设计一个static静态变量count作为计数器；

​	2、类定义结束后初始化count;

​	3、在构造函数中对count进行+1;

​	4、设计拷贝构造函数，在进行拷贝构造函数中进行count +1，操作；

​	5、设计赋值构造函数，在进行赋值函数中对count+1操作；

​	6、在析构函数中对count进行-1；



## 105、怎么快速定位错误出现的地方?

​		1、如果是简单的错误，可以直接双击错误列表里的错误项或者生成输出的错误信息中带行号的地方就可以让编辑窗口定位到错误的位置上。

​		2、对于复杂的模板错误，最好使用生成输出窗口。

​		多数情况下出发错误的位置是最靠后的引用位置。如果这样确定不了错误，就需要先把自己写的代码里的引用位置找出来，然后逐个分析了。



## 106、成员初始化列表会在什么时候用到？它的调用过程是什么？

1. 当初始化一个引用成员变量时；
2. 初始化一个const成员变量时；
3. 当调用一个基类的构造函数，而构造函数拥有一组参数时；
4. 当调用一个成员类的构造函数，而他拥有一组参数；
5. 编译器会一一操作初始化列表，以适当顺序在构造函数之内安插初始化操作，并且在任何显示用户代码前。list中的项目顺序是由类中的成员声明顺序决定的，不是初始化列表中的排列顺序决定的。





## 107、在进行函数参数以及返回值传递时，可以使用引用或者值传递，其中使用引用的好处有哪些？

对比值传递，引用传参的好处：

​		1）在函数内部可以对此参数进行**修改**

​		2）提高函数调用和运行的**效率**（因为没有了传值和生成副本的时间和空间消耗）

​		如果函数的参数实质就是形参，不过这个形参的作用域只是在函数体内部，也就是说实参和形参是两个不同的东西，要想形参代替实参，肯定有一个值的传递。函数调用时，值的传递机制是通过“**形参=实参”**来对形参赋值达到传值目的，产生了一个实参的副本。即使函数内部有对参数的修改，也只是针对形参，也就是那个副本，实参不会有任何更改。函数一旦结束，形参生命也宣告终结，做出的修改一样没对任何变量产生影响。

​		用引用作为返回值最大的好处就是在内存中不产生被返回值的副本。

但是有以下的限制：

​		1）**不能返回局部变量的引用**。因为函数返回以后局部变量就会被销毁

​		2）**不能返回函数内部new分配的内存的引用。**虽然不存在局部变量的被动销毁问题，可对于这种情况（返回函数内部new分配内存的引用），又面临其它尴尬局面。例如，被函数返回的引用只是作为一 个临时变量出现，而没有被赋予一个实际的变量，那么这个引用所指向的空间（由new分配）就无法释放，造成memory leak

​		3）**可以返回类成员的引用，但是最好是const。**因为如果其他对象可以获得该属性的非常量的引用，那么对该属性的单纯赋值就会破坏业务规则的完整性。





## 108、说一说strcpy、sprintf与memcpy这三个函数的不同之处

1. 操作对象不同

   ① strcpy的两个操作对象均为字符串

   ② sprintf的操作源对象可以是多种数据类型，目的操作对象是字符串

   ③ memcpy的两个对象就是两个任意可操作的内存地址，并不限于何种数据类型。

2. 执行效率不同

   memcpy最高，strcpy次之，sprintf的效率最低。

3. 实现功能不同

   ① strcpy主要实现**字符串**变量间的拷贝

   ② sprintf主要实现其他**数据类型格式**到字符串的转化

   ③ memcpy主要是**内存块**间的拷贝。



## 109、将引用作为函数参数有哪些好处？

1. 传递引用给函数与传递指针的效果是一样的。

   ​	这时，被调函数的形参就成为原来主调函数中的实参变量或对象的一个别名来使用，所以在被调函数中对形参变量的操作就是对其相应的目标对象（在主调函数中）的操作。

2. 使用引用传递函数的参数，在内存中并没有产生实参的副本，它是直接对实参操作；

   ​	而使用一般变量传递函数的参数，当发生函数调用时，需要给形参分配存储单元，形参变量是实参变量的副本；

   ​	如果传递的是对象，还将调用拷贝构造函数。因此，当参数传递的数据较大时，用引用比用一般变量传递参数的效率和所占空间都好。

3. 使用指针作为函数的参数虽然也能达到与使用引用的效果，但是，在被调函数中同样要给形参分配存储单元，且需要重复使用"*指针变量名"的形式进行运算，这很容易产生错误且程序的阅读性较差；

   ​	另一方面，在主调函数的调用点处，必须用变量的地址作为实参。而引用更容易使用，更清晰。



## 110、你知道数组和指针的区别吗？

1. 数组在内存中是连续存放的，开辟一块连续的内存空间；数组所占存储空间：sizeof（数组名）；数组大小：sizeof(数组名)/sizeof(数组元素数据类型)；
2. 用运算符sizeof 可以计算出数组的容量（字节数）。sizeof(p),p 为指针得到的是一个指针变量的字节数，而不是p 所指的内存容量。
3. 编译器为了简化对数组的支持，实际上是利用指针实现了对数组的支持。具体来说，就是将表达式中的数组元素引用转换为指针加偏移量的引用。
4. 在向函数传递参数的时候，如果实参是一个数组，那用于接受的形参为对应的指针。也就是传递过去是数组的首地址而不是整个数组，能够提高效率；
5. 在使用下标的时候，两者的用法相同，都是原地址加上下标值，不过数组的原地址就是数组首元素的地址是固定的，指针的原地址就不是固定的。





## 111、如何阻止一个类被实例化？有哪些方法？

1. 将类定义为抽象基类或者将构造函数声明为private；
2. 不允许类外部创建类对象，只能在类内部创建对象





## 112、 如何禁止程序自动生成拷贝构造函数？

1. 为了阻止编译器默认生成拷贝构造函数和拷贝赋值函数，我们需要手动去重写这两个函数，某些情况﻿下，为了避免调用拷贝构造函数和﻿拷贝赋值函数，我们需要将他们设置成**private**，防止被调用。
2. 类的成员函数和friend函数还是可以调用private函数，**如果这个private函数只声明不定义，则会产生一个连接错误；**
3. 针对上述两种情况，**我们可以定一个base类，在base类中将拷贝构造函数和拷贝赋值函数设置成private**,那么派生类中编译器将不会自动生成这两个函数，且由于base类中该函数是私有的，因此，派生类将阻止编译器执行相关的操作



## 113、你知道Debug和Release的区别是什么吗？

1. 调试版本，包含调试信息，所以容量比Release大很多，并且不进行任何优化（优化会使调试复杂化，因为源代码和生成的指令间关系会更复杂），便于程序员调试。Debug模式下生成两个文件，除了.exe或.dll文件外，还有一个.pdb文件，该文件记录了代码中断点等调试信息；
2. 发布版本，不对源代码进行调试，编译时对应用程序的速度进行优化，使得程序在代码大小和运行速度上都是最优的。（调试信息可在单独的PDB文件中生成）。Release模式下生成一个文件.exe或.dll文件。
3. 实际上，Debug 和 Release 并没有本质的界限，他们只是一组编译选项的集合，编译器只是按照预定的选项行动。事实上，我们甚至可以修改这些选项，从而得到优化过的调试版本或是带跟踪语句的发布版本。



## 114、main函数的返回值有什么值得考究之处吗？

​		程序运行过程入口点main函数，main（）函数返回值类型必须是int，这样返回值才能传递给程序激活者（如操作系统）表示程序正常退出。

​		main（int args, char **argv） 参数的传递。参数的处理，一般会调用getopt（）函数处理，但实践中，这仅仅是一部分，不会经常用到的技能点。



## 115、模板会写吗？写一个比较大小的模板函数

```c++
#include<iostream> 
using namespace std; 

template<typename type1,typename type2>//函数模板 
type1 Max(type1 a,type2 b) 
{ 
   return a > b ? a : b; 
} 

void main() 
{ 
  cout<<"Max = "<<Max(5.5,'a')<<endl; 
} 

```



## 116、strcpy函数和strncpy函数的区别？哪个函数更安全？

函数原型:		

```c++
char* strcpy(char* strDest, const char* strSrc)
char *strncpy(char *dest, const char *src, size_t n)
```

1. - strcpy函数: 如果参数 dest 所指的内存空间不够大，可能会造成缓冲溢出(buffer Overflow)的错误情况，在编写程序时请特别留意，或者用strncpy()来取代。
   - strncpy函数：用来复制源字符串的前n个字符，src 和 dest 所指的内存区域不能重叠，且 dest 必须有足够的空间放置n个字符。
2. - 如果目标长>指定长>源长，则将源长全部拷贝到目标长，自动加上’\0’
   - 如果指定长<源长，则将源长中按指定长度拷贝到目标字符串，不包括’\0’
   - 如果指定长>目标长，运行时错误 ；