---
layout: article
title:  "Effective C++ (3rd Edition)"
categories: programing
tags: [c++, reading]
toc: false
image:
    teaser: programing/2018-06-27-Effective-C++(3rd-Edition)/teaser.jpg

date: 2018-06-27
---

因项目需要，将网络通信的实现部分抽离到C++，使Andoird与iOS能够复用部分逻辑代码，
故将《Effective C++》在空闲时间看了一遍。全书很多地方都让人有醍醐灌顶之感，
尤其是在啃完Primer之后。总之，本书对理解和掌握C++实践方面所应规避的坑还是很有帮助的。

---

## 1. 让自己习惯C++

C++是四个层次的语言，即C、Object-Oriented C++、Templeate C+_+和STL。

## 2. 尽量以const, enum, inline替换`#define`

对于单纯常量最好用const对象或enums替换`#defines`

对于形似函数的宏，最好改用inline函数替换`#defines`

注意即便宏的所有实参都加上了小括号，也无法避免++a时a变量被多次累加的情况

## 3. 尽可能使用const

声明const可帮助编译器侦测出错误用法，编译器实施bitwise(phisical) constness，
但写程序时应使用logical constness

## 4. 确认对象被使用前已被初始化

为内置型对象进行手动初始化，因为C++不保证初始化它们

总是使用成员初值列，绝对必要且比赋值高效

class的成员变量总是以其声明次序被初始化，所以使用初值列时也最好用声明次序

C++对于在不同编译单元内的non-local static对象的初始化次序无明确定义。
可以使用local static对象替换non-local static对象来避免，这也是单例的常用写法

## 5. 了解C++默认编写并调用哪些函数

编译器可以默认为class创建default构造函数、copy构造函数、
copy asignment操作符，以及析构函数

## 6. 若不想使用编译器自动生成的函数，就应该明确拒绝

为驳回编译器自动生成的函数，可将相应的成员函数声明为private并且不予实现。
也可以继承一个这样实现的基类。

## 7. 为多态基类声明virtual析构函数

带有多态性质的积累应该声明一个virtual析构函数
即如果class带有任何virtual函数， 它就应该拥有一个virtual析构函数

如果class不是设计用于基类或不是为了具备多态性质，就不应该声明virtual析构函数
如string类和STL的容器类vercor, list, set等

## 8. 别让析构函数抛异常

析构函数不要抛出任何异常，应该捕获然后做相应的处理

## 9. 绝不要在构造和析构过程中调用virtual函数

在构造和析构期间不要调用virtual函数，因为这类调用不会下降至derived class。
及此时会将对象视为base class类型，derived class还未初始化或已被析构。

## 10. 令operator=返回一个reference to \*this

令赋值操作符，包括标准赋值形式(=)和其他赋值相关运算(+=,-=,*=等)，返回一个
reference to \*this

## 11. 在operator=中处理自我赋值(例如a = a)

确保当对象自我赋值时有良好行为。可以比较两个对象的地址，
可以调整语句顺序，可以copy-and-swap

确保任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，
其行为仍然正确

做法1，可以使用证同测试，但不具备异常安全性，
即new Bitmap产生异常，会让pb始终指向一块被删除的Bitmap

```C++
Widget& Widget::operator=(const Widget& rhs) {
    if (this == &rhs) return *this;
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

做法2: 赋值pb所指东西之前别delete pb

```
Widget& Widget::operator=(const Widget& rhs) {
    Bitmap* pOrig = pb;
    pb = new Bitmap(*rhs.pb);
    delete pOrig;
    return *this;
}
```

做法3: copy-and-swap

```
Widget& Widget::operator=(const Widget& rhs) {
    Widget temp(rhs);
    swap(temp);
    return *this;
}
```

## 12. 复制对象时勿忘其每一个成分

Copying函数(copy构造函数和copy assignment操作符)应该确保对象内所有成员变量
及所有base class成分

不要尝试以某个copying函数实现另一个copying函数。
应该将共同机能放进第三个函数中已供调用。

## 13. 以对象管理资源

为防止资源泄漏，请使用RAII(Resource Acquisition Is Initialization)对象，
他们在构造函数中获得资源并在析构函数中释放资源。

两个常备使用的RAII classes分别是tr1::shared\_ptr和auto\_ptr。
前者通常是较佳选择，因为其copy行为比较直观，但无法避免环状引用。
后者copy行为会使被复制对象指向null

## 14. 在资源管理类中小心copying行为

复制RAII对象必须一并复制它管理的资源，即深度拷贝

通常RAII对象常见的copying行为是抑制copying、使用引用计数法

## 15. 在资源管理类中提供对原始资源的访问

APIs往往要访问原始资源，所以每一个RAII class应该提供一个获取原始资源的办法

获取原始资源可以使用显示转换或隐式转换，一般而言显示转换更为安全

## 16. 成对使用new和delete时要采取相同形式

当使用new[]时在删除时使用delete[]，当使用new时在删除时使用delete
另外最好不要对数组作typedef操作，很容易在delete时忘记使用delete[]

## 17. 以独立语句将new对象置入智能指针

以独立语句将new对象置入智能指针内
否则一旦异常被抛出，有可能导致难以察觉的内存泄漏

因为编译器对于跨越语句的各项操作没有重新排列执行顺序的自由度

## 18. 让接口容易被正确使用，不易被误用

好的接口很容易被正确使用，不易被误用

促进正确使用的办法包括保持接口的一致性，与内置类型的行为兼容

阻止误用的版本包括建立新类型，限制类型上的操作，束缚对象值，
以及消除客户的资源管理责任

tr1::shared_ptr支持定制型删除器，可以预防DLL问题，也可用作自动解除互斥锁等

## 19. 设计class犹如设计type

class的设计就是type的设计，所以应该谨慎设计

## 20. 自定义类型以pass-by-reference-to-const替代pass-by-value

pass-by-referen-to-const比较高效，并可避免切割问题(slicing problem)，
即传递子类对象时被复制成父类对象

对于内置类型、STL迭代器和函数对象，使用pass-by-value比较合适

## 21. 必须返回对象时，别返回其reference

绝不要返回pointer或reference指向一个local stack对象、heap-allocated对象
以及local static对象

## 22. 将成员变量声明为private

成员变量使用private可以使class更容易扩展

## 23. 宁以non-member、non-friend替换member函数

使用non-member、non-friend函数替换member函数可以增加封装型和扩展性

## 24. 若所有参数皆需类型转换，请为此采用non-member函数

如果你需要为某个函数的所有参数(包括被this指针所指的那个隐喻参数)进行
类型转换，那么这个函数必须是个non-member，且如无必要，也无须作为friend函数

## 25. 考虑写出一个不抛异常的swap函数

当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数
不抛出异常

如果你童工一个member swap，也应该提供一个non-member swap用来调用前者。
对于class(非templates)，也请特例化std::swap，命名空间不能使用std

调用swap时应针对std::swap使用using声明式，然后调用swap时不带任何命名空间修饰

可以在用户定义类型中特例化std template，但不要尝试为std命名空间添加内容

## 26. 尽可能延后变量定义的出现时间

尽可能延后定义式的出现，尽可能在构造时初始化

## 27. 尽量少做转型动作

C++提供四种新型转型:
```
const_cast<T>(expression)
dinamic_cast<T>(expression)
reinterpret_cast<T>(expression)
static_cast<T>(expression)
```

`const_cast`通常被用来将对象的常量性移除(cast away the constness)

`dynimaic_cast`主要用来执行安全向下转型(safe downcasting)，用来决定
某对象是否归属继承体系的某个类型。它是唯一无法由旧式语法执行的动作，
也是唯一可能耗费大量运行成本的动作

`reinterpret_cast`意图执行低级转型，实际结果取决于编译器，即不可移植，
如将一个pointer to int转型为一个int

`static_cast`用来强制隐式转换(implicit conversions)，
例如将non-const转为const，将int转为double等

如果可以尽量避免转型，特别是避免dynamic_cast

如果转型是必要的，试着将它隐藏在某个函数背后

宁可用c++-style(新型)转型，也不要使用旧式转型，前者很容易辨识出来，
而且也有着分门别类的执掌

## 28. 避免返回handles指向对象内部成分

避免返回handles(包括引用、指针和迭代器)指向对象内部。
const成员函数的返回值尽量使用const修饰，
此外应避免虚吊(dangling)handles的出现

dangling handles的例子:

```
GUIObject* pgo;
const Point* pUpperLeft = &((boundingBox(*pgo).upperLeft()))
// boundingBox在该表达式之后会被销毁导致Point被析构，pUpperLeft指向不存在的对象
```

## 29. 为异常安全而努力是值得的

异常安全函数即使发生异常也不会泄漏资源或允许任何数据结构被破坏，
可分为三种可能的保证：基本型、强烈型、不抛异常性

强烈保证提供要么成功，要么恢复原有状态的原子性，
往往能够以copy-and-swap的方式实现

函数提供的异常安全保证通常
等于其调用的各个函数的异常安全保证中的最弱者

## 30. 透彻了解inline

将大多数inlining限制在小型、被频繁调用的函数身上。
这样可以使二进制升级更容易，也可使潜在的代码膨胀问题最小化

不要只因为function templates出现在头文件，就将它们声明为inline

## 31. 将文件间的编译依赖关系最小化

实现最小化编译依赖性的一般思路是：依赖声明式，而不要依赖定义式
通常的做法有Handle Class和Interface Class

程序库头文件应该完全仅有声明式的形式存在，
这种做法无论是否涉及templates都适用

## 32. 确定public继承是is-a关系

public继承意味is-a，适用于base class的每一件事情也适用于derived class身上
因为每一个derived class对象也都是一个base class对象

## 33. 避免遮掩继承而来的名称

derived class内的名称会遮掩base class内的名称(包括成员名和函数名)，
在public继承下往往不希望如此

为了让被遮掩的名称重见天日，可使用using声明式或转交函数(forwarding fuctions)

```
// 1. using声明式
calss Derived: public Base {
public:
    using Base::mf1; // 让Base里的所有mf1可见
    using Base::mf3; // 让Base里的所有mf3可见
    virtual void mf1();
    void mf3();
}
// 2. forwarding函数
// 私有继承基类的公有和保护成员都作为派生类的私有成员
class Derived: private Base {
    virtual void mf1() {
        Base::mf1();
    }
}
```

## 34. 区分接口继承和实现继承

derived class总是会继承base class的接口

pure virtual是只继承接口(必须重新实现)
impure virtual继承接口及缺省实现(可被重新重新)
non-virtual继承接口及强制性实现(理论上也可被重新实现，但不要这么做)

## 35. 考虑virtual函数以外的其他选择

使用non-virtual interface(NVI)方式是Template Method(模版方法)设计模式的一种
特殊形式。它以public non-virtual成员函数包裹较低访问性(private或protected)
的virtual函数。

将virtual函数替换为"函数指针成员变量"，这是Strategy(策略)设计模式的一种
分解表现形式。

以tr1::function成员变量替换virtual函数，因而允许使用任何可调用物(callable entity)
搭配一个兼容的函数签名式。这也是Strategy(策略)实际模式的某种形式。

将继承体系内的virtual函数替换为另一个继承体系内的virtual函数。这是
Strategy(策略)设计模式的传统实现方式。(这个说法很精辟)

## 36. 绝不重新定义继承而来的non-virtual函数

绝对不要重新定义继承而来的non-virtual函数

## 37. 绝不重新定义继承而来的缺省参数值

绝对不要重新定义一个继承而来缺省参数值，因为虽然virtual函数(唯一应该覆写的)
是动态绑定，但其中的缺省参数值都是静态绑定

可以通过模版方法来定义有缺省参数值又需要子类继承的函数

## 38. 通过复合塑膜出has-a或"根据某物实现出"

根据某物实现出(is-implemented-in-terms-of)指A基于B实现，比如Set可以基于list实现
Set类中包含一个list，然后由list来实现Set的功能

在应用域，复合意味着has-a，在实现域，复合意味着is-implemented-in-terms-of
(根据某物实现出)

## 39. 明智而审慎地使用private继承

private继承意味is-implemented-in-terms-of(根据某物实现出)。
当derived class需要访问protected base class的成员，或需要重新定义
继承而来的virtual函数时，这么设计是有一定合理性的

和复合不同，privete继承可以使用空白基类最优化(EBO)，因为一般情况空白基类
会被默默安插一个char到空对象中，对致力于对象尺寸最小化的开发者而言，
可能这点会很重要

实际上在使用private继承时，应该先考虑能否使用复合或其他方式替代

## 40. 明智而审慎地使用多重继承

多重继承比单一继承更复杂，容易导致歧义性，C++会先检查歧义性才去检查可见性，
所以有时需要virtual继承来避免

virtual继承会增加大小、速度、初始化(及赋值)复杂度等等成本。如果vritual base
class不带任何数据(类似java的接口)，则将是最具实用价值的情况

多重继承的确有其正当用途，比如使用public继承某个interface class，
同时使用private继承某个协助实现的class

## 41. 了解隐式接口和编译期多态

class和template都支持接口和多态

对class而言接口是显式的(explicit)，以函数签名实现。
多态则是通过virutal函数在运行期发生

对template参数而言，接口是隐式的(implicit)，基于有效表达式实现。
多态则是通过template的具象化和函数重载解析在编译器发生

## 42. 了解typename的双重意义

声明template参数时，前缀关键字class和typename可互换

请使用关键字typename标识嵌套从属类型(嵌套于取决于template参数的东西内)名称，
但不得在base class lists(基类列)或member init list(成员初值列)内以它作为
base class的修饰符

## 43. 学习处理模板化基类内的成员

Obect Oriented C++跨进Template C++时，继承template的基类，会假设对
基类一无所知，因为template的基类可能会被特例化而不含相关成员。
解决办法可以在derived calss templates中通过this->调用基类的成员，
或通过显式写出的base class作用域修饰符完成

## 44. 将与参数无关的代码抽离templates

template生成多个class和多个函数，所以任何template代码都不该与某个造成膨胀
的template参数产生依赖关系

因非类型模版参数(non-type template parameters)而造成的代码膨胀，可以通过
函数参数或class成员变量替换template参数来消除

因类型参数(type parameters)而造成的代码膨胀，可以通过带有完全相同二进制
表述的具象类型(instantiation types)共享实现码

## 45. 运用成员函数模版接受所有的兼容类型

请使用member function templates(成员函数模板)生成"可接受的所有兼容类型"函数

如果声明了member templates用于泛化copy构造或泛化assignment操作符，还是需要
声明正常的copy构造函数和copy assignment操作符，否则系统会生成默认的相应的
copying函数

## 46. 需要类型转换时请在模版内定义非成员函数

当template class需要实现与此相关的template函数支持所有参数隐式类型转换时，
可以将那些函数定义为template class内部的friend函数
因为template的实参推导不考虑隐式类型转换

## 47. 请使用traits classes表现类型信息

traits classes使得类型相关信息在编译期可用。以templates和templates特化来实现

整合重载技术后，traits classes可以在编译期实现对类型执行if-else的判断功能
(无需真实的if-else语句)

## 48. 认识template元编程

template metaprogramming(TMP, 模板元编程)可将工作由运行期移至编译期，
因而得以实现早期错误侦测和更高的执行效率

TMP可被用来生成基于策略选择组合的客户定制代码，也可用来避免生成对某些
特殊类型并不适合的代码

## 49. 了解new-handler行为

`set_new_handler`允许客户指定一个函数，在内存分配时无法获得满足时被调用

`nothrow new(new (std::nothrow) SomeClass)`只适用于内存分配，后续构造函数
的调用仍然可能会抛出异常

## 50. 了解new和delete的合理替换时机

有很多理由需要写个自定义的new和delete，包括改善性能、对heap运用错误进行调试、
收集heap使用信息。但需要注意字节对齐(如double为8-byte对齐时访问最快速)等问题

## 51. 编写new和delete时需固守常规

operator new应该内含一个无穷循环，并在其中尝试分配内存，如果它无法满足内存
需求，就该调用new-handler。它应该也要有能力处理0 bytes申请。Class专属版本
还应该处理与正确大小不相等的(错误)申请

operator delete应该在收到null指针时不作任何处理。Class专属版本还应该处理
与正确打扫小不相等的(错误)申请

## 52. 写了placement new也要写placement delete

当你写了一个placement opeator new，请确定也写出了对应的placement opeator
delete。否则可能会在构造函数抛出异常时出现内存泄漏

当声明了placement new和placement delete，请不要遮掩它们的正常版本，
可以使用using语句导入其正常版本

## 53. 不要轻忽编译器的警告

严肃对待编译器发出的警告信息。努力在你的编译器的最高警告级别下争取
无任何警告的荣誉

不要过度依赖编译器的报警能力，因为不同编译器的处理可能不一致，原本依赖
的警告信息可能在移植到另一个编译器后消失

## 54. 让自己熟悉包括TR1在内的标准函数库

C++标准库主要机能由STL、iostreams、locales组成。并包含C99标准程序库

TR1添加了智能指针(`tr1::shared_ptr`等)、一般化函数指针(`tr1::function`)、
hash-based容器、正则表达式(regular expressions)以及另外10个组件的支持

TR1自身知识一份规范。为取得TR1，需要一份实现，可以选择boost

## 55. 让自己熟悉boost

boost是一个社群，也是一个网站，致力于免费、开源的C++程序库开发

boost提供很多tr1组件的实现，以及其他许多程序库


---
The End.

zhlinh

Email: zhlinhng@gmail.com

2018-06-27
