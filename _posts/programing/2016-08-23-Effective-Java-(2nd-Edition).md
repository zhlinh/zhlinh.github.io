---
layout: article
title:  "Effective Java (2nd Edition)"
categories: programing
tags: [java, reading]
toc: true
image:
    teaser: programing/2016-08-23-Effective-Java-(2nd-Edition)/teaser.jpg

date: 2016-08-23
---

近些天在看《Effective Java》第二版中文版，为了在某些地方降低自己的阅读速度，
便做了些笔记。读完整本书，感觉收获还是颇丰的，从阅读体验来说甚至还超过了
《Java编程思想》，前者重实践，后者重理论。尤其是前面一部分，翻译感觉很流畅，
然而从第九章开始，就开始漏洞频出了，一看序言，才发现是两个不同的译者翻的。
好吧，本来还想在此列举各种翻译毛病以印证自己所言不虚，想想毕竟还是瑕不掩瑜，
其实是限于篇幅，哈哈。当然也有可能前面也有错误自己没注意到，然后在后面碰到
几个抓狂的便下此定论。总体而言，还是相当不错的，值得推荐。

---

## 1 Todo

- [ ] 查询JavaBeans模式
- [ ] 查询Emacs的单线程的影响，这跳跃性思维也是醉。。。
- [ ] 查询java.lang.ref如何解决缓存的内存泄露问题。
- [ ] 查询弱引用weak reference，还有WeakHashMap。
- [ ] 查询本地对等体native peer。
- [ ] 查询Float.NaN，-0.0f及Double的类似常量。
- [ ] 查询Float.floatToIntBits()，Double.doubleToLongBits()生成散列码中看到
- [ ] 查询canoical representatition范式，总感觉这翻译有玄机。。。
- [ ] 查询包装类回调的SELF问题。
- [ ] 查询二进制兼容性。
- [ ] 查询视图(view)。
- [ ] 查询readObjectNoData。
- [ ] 查询Object Serialization Stream Protocol。

---

## 2 创建和销毁对象

### 2.1 模式

- 静态工厂模式
- Builder模式(与重叠构造器telescoping constuctor模式和JavaBeans模式的setter作比较)
- 重叠构造器模式的硬伤为代码冗长。
- JavaBeans模式的硬伤为线程不安全。
- 从Builder模式进而引出抽象工程模式,即可以将这样的Builder传递给别的方法。
- 定义一个通用的Builder接口：
{% highlight java %}
{% raw %}
public interface Builder<T> {
    public T build();
}
{% endraw %}
{% endhighlight %}
- Builder方法的不足： 1.多创建了它自身的构造器，影响性能 2.实际上代码会比重叠构造器模式更加冗长，除非真的很多参数(`>=4`)，但一开始确定了该模式，以后更易于扩展。
- 综上，最好一开始还是就用Builder，迷之微笑^_^
- 构造单例模式的方法：1，提供公有的可访问final且transient的静态域；2，提供私有的final且transient静态域及静态工厂方法；3，利用但元素的Emum(推荐)，因其实现了serializable，并保证了不可被反射构建，简单高效。
- 优先使用基本数据类型而不是装箱基本数据类型，要当心无意识的自动装箱。
- 维护对象池来避免创建对象并不是一个好的做法，除非其中的对象是非常重量级的(创建的代价昂贵，如数据库的连接池，还可以用以限制连接的数量)。否则如今的JVM实现具有高度优化的垃圾回收器，其性能很容易就会超越轻量级的对象池的性能（增加内存占用footprint）。
- 借助Heap剖析工具(Heap Profiler)可以帮助发现内存泄露问题。

### 2.2 finalize方法

- 应该避免使用终结方法(finalizer)。不能保证会被及时的执，java会延迟执行终结方法，终结方法线程的优先级比该应用程序的其他线程的要低的多。典型错误：用终结方法关闭已打开的文件。会有大量的对象同时等着被终结和回收，如果对象先被回收，则终结方法不会被执行。故重要的工作如解锁等不宜放在终结方法中。另外还有终结方法中不可处理异常及创建终结方法的代价昂贵的问题。
- 可以用显式的终结方法，如InputStream、OutputStream和java.sql.Connection上的close方法以及java.util.Timer的cancel方法。通常显式的终结方法会和try-finally结合起来使用。
- 子类覆盖了超类的终结方法，必须手工调用超类的终结方法，放到try-finally中。
{% highlight java %}
{% raw %}
@override protected void finalize() throws Throwable {
    try{
        ...
    } finally {
        super.finalize()
    }
}
{% endraw %}
{% endhighlight %}
- 可将终结方法放到匿名类中，该匿名类保存着外部类的引用，其唯一用途就是终结它的外围实例。该匿名类的单个实例被称为终结方法守护者finalizer guardian。

---

## 3 Object对象非final方法

- equals的五个约定：四个数学约定（自反性，对称性，传递性，一致性）加上一个非空约定（所有对象都不等于null，其实很容易解决，因为要判断等同性，都需先instanceof，在此之前还可以先用`==`判断是否为同一对象的引用，然后在此之后可以强制类型转换进行下一步检验，这样可以优化性能）。
- 在扩展值对象的时候增加值组件会破坏equls的约定，情况如下：
{% highlight java %}
{% raw %}
/**
 * Point: int x, int y
 * ColorPoint extends Point: int x, int y, Color color
 */
// 破坏对称性的覆盖方法
@Override public boolean equals(Object o) {
    if (!(o instanceof ColorPoint)) {
        return false;
    }
    return super.equals(o) && ((ColorPoint) o).color == color;
}
// 试比较new Point(1, 2) 和 new ColorPoint(1, 2, Color.RED)
// 符合对称性但破坏了传递性的覆盖方法
@Override public boolean equals(Object o) {
    if (!(o instanceof Point) {
        return false;
    }
    if (!(o instanceof ColorPoint)) {
        return o.equals(this);
    }
    return super.equals(o) && ((ColorPoint) o).color == color;
}
// 试比较new ColorPoint(1, 2, Color.RED), new Point(1, 2)和 new ColorPoint(1, 2, Color.BLUE)
{% endraw %}
{% endhighlight %}
- 很容易到用`if (o == null || o.getClass() != getClass()) {return false;}`来替换instanceof，这样确实不会破坏equals的约定，但这会破坏另外一个原则——里氏替换原则(Liskov substitution principle，适用于父类型的其他方法，子类型同样适用，即保证多态性)
- 权宜之计，利用复合代替继承。当然也可以在抽象类的子类中添加新的值组件，这样可以足equals的约定。
- 不要尝试将equals的单一参数Object换为其他类型，这样就不是覆盖了，OK？善用@Override注解可避免犯该错误。
- 另外，覆盖equals时总要覆盖hashCode。不然无法与基于散列hash的集合一起好好玩耍了。因为hashCode也有三个约定，一致性，相等的对象的hash code一定相等[因违反该条，本来相等的对象但hash code不一致，这样put进HashMap再get的时候就get不到啦]，不相等的对象的hash code不一定不相等[理想情况当然希望不相等，而且最好还是均匀分布，但谁知道怎么就落到了同一个散列桶链表里了呢，无辜脸。
- String, Integer和Date覆盖了Object的hashCode方法。
- 始终要覆盖toString方法，虽然这个是建议，但确实是一个很好的建议。println或printf打印对象时，可以直接显示自定义信息不是挺Happy的么。
- 谨慎覆盖clone方法。Cloneable接口的目的是作为对象的为mixin（混合类型），仅表明该对象允许克隆，即允许克隆功能更新到现有类中。但Object的clone方法是受保护的，所以需要借助反射机制。
- Cloneable会改变Object类中clone方法的行为(接口的非典型用法，不值得效仿)，如果一个类实现了Cloneble则Object的clone方法就返回该对象的逐域拷贝，否则抛出异常。
- 所有实现了Cloneable接口的类都应该用一个**公有**的方法覆盖clone。此公有方法首先调用super.clone，然后修正任何需要修正(不包含final域)的域，如使其完成深度拷贝。
- 推荐的做法是即便是为了继承而设计的类也不应该实现Cloneable接口，不提供行为良好的受保护的clone方法，其子类也不能实现Cloneable接口。许多专家级程序员干脆就不覆盖clone方法，也不调用它，除非拷贝数组。可以改用传入原对象作为参数的拷贝构造器、转换构造器、静态工厂或转换工厂的方法，如new TreeSet(new HashSet())。

> Coparable接口的唯一方法compareTo()

- ompareTo方法有四个约定：对称性，两个相同不等号的传递性，一个等号和一个不等号的传递性和等号返回0(最后这一条仅为强烈建议，但如果满足则表示与equals的顺序关系一致，不满足则与equals不一致)。
- 再解释一下不遵守最后一条约定的情况，也是可以接受的。比如HashSet中添加new BigDecimal("1.0")和new Big Decimal("1.00")，因为用equals比较二者是不相等的，所以HashSet会有两个元素。但如果使用的是TreeSet，那么因为通过comprareTo比较时是相等的，所以TreeSet会只有一个元素。
- 与违反hashCode约定的类会破坏其他依赖于散列做法的类一样，违反comparaTo约定的类会破坏其他依赖于比较关系的类，如有序集合类TreeSet和TreeMap，以及工具类Collections和Arrays。
- 因为compareTo方法的参数为泛型T（即须实现Comparable时须先声明参数），并不是Object，而且为静态的，所以与equals不同的是它不用进行类型检查与转换，因为如果参数不合适则无法编译。
- 对象的引用域可以递归地调用compareTo方法实现，如果一个域没有实现Comparable接口，或者你希望使用其他的排序方式，可以使用一个显示的Comparator(注意单词的拼写是para而不是pare)的compare方法来代替。
{% highlight java %}
{% raw %}
public final class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
}
... // Remainder omitted
{% endraw %}
{% endhighlight %}
- 比较整数基本类型的域，可使用关系操作符`<`和`>`,浮点域可以用Float.compare和Double.compare。注意如果两个整数比较如果不用关系操作符而是直接返回`x - y`，可能会出现溢出，然后产生错误的比较结果。

---

## 4 类和接口

- 尽可能降低可访问性。实例域决不能是私有的。类不能具有公有的静态final数据域并不能有返回该域的访问方法。正确的做法应为：
{% highlight java %}
{% raw %}
// 法一: 增加一个公有的不可变列表
private static final Thing[] PRIVATE_VA = { ... };
public static final List<Thing> VALUES = Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
// 法二: 返回私有数组的一个副本。
private static final Thing[] PRIVATE_VALUES = { ... };
public static final Thing[] values() {
    return PRIVATE_VALUES.clone();
}
{% endraw %}
{% endhighlight %}
- 尽可能降低可变性。
- 不可变的类必须是final的。否则其方法可能被子类覆盖，不能保证某实例是否为真正的父不可变类。如BigInteger和BigDecimal就存在这个问题。需要对它进检查，如果是其子类则需要进行保护性拷贝。
{% highlight java %}
{% raw %}
public static BigInteger safeInstance(BigInteger val) {
    if (val.getClass() != BigInteger.class) {
        return new BigInteger(val.toByteArra));
    }
    return val;
}
{% endraw %}
{% endhighlight %}
- 除非有很好的理由让类(或域)变为可变的，否则就应该是不可变final的。坚决不要为每一个get方法编写一个相应的set方法。然后解决潜在的性能问题时，才应该为不可变类提供公有的可变配套类。如String和StringBuilder（或已废弃的StringBuffer。
- 构造器应该创建完全初始化的对象，并建立起所有的约束关系。即不要提供重新初始化的方法（重新改变初始状态），与所增加的复杂性相比，重新初始化方法通常并没有带来太多的性能优势。
- 复合(composition)优先于类继承(inheritace)。复合得到的新类称为包装类(wrapper class)，其中的方法调用被复合类(转发类)的方法，该过程称为转发(forwarding)，这些方法称为转发方法(forwarding method)。复合和转发的组合并不是委托(delegation)，除非包装对象把自身传递给被包装的对象。
- 包装类不适合用在回调框架(callback framework)中。在回调框架中，对象把自身引用传递给其他的对象，用于后续的调用，称为回调。因为被包装的对象并不知道它外面的包装对象，所以它传递一个指向自身的引用(this)，回调时避开了外面的包装对象。这被称为SELF问题。
- 只有子类真正是超类的子类型(subtype)时，才适合用**继承**，即二者存在`is-a`关系。如果A本质上不是B的一部分，只是它的实现细节而已，那么应该考虑**复合转发**。Java平台的现有类库中，违反该规则的例子有栈Stack继承向量Vector，属性列表Properties继承散列表Hashtable。
- 为继承而设计的类必须提供文档说明，其中的方法调用了哪些可被覆盖的方法。这确实违背了好的API文档应该描述某个方法做了什么工作，而不是描述如何做到的。但这确实是有必要的，因为继承本身就破坏了封装性，要避免产生不良后果，则必须描述清楚实现细节。或者去除“可覆盖方法的每个自用(self-use)调用”，即可覆盖方法只能调用私有辅助方法。
- 构造器、clone(实现Clonable)、readObject（实现Serializable）决不能调用可被覆盖的方法。无论是直接调用还是间接调用。
- 为继承而设计的类中实现Serializable应该使readResolve或writeReplace称为受保护的而不是私有的方法。这正是为了允许继承，而把实现细节变成一个类的API的一部分。
- 对于普通的具体类，即并非为了继承而设计，则应禁止继承，方法有将类声明为final或者将所有的构造其变为私有的或包级私有的，然后提供公有的静态工厂。
- 接口优先于抽象类。而且应该考虑为接口提供骨架实现类或部分实现的抽象类。接口的设计应该谨慎并进行全面测试，因为后期再想改将会破坏所有实现这个接口的类。
- 避免使用常量接口（即接口不是为了定义类型而是用来导出常量，只包含静态的final域）。因为如果某个实现了该接口的类不想再使用这些常量，但它还必须实现该接口以保持二进制兼容性。导出常量的推荐方案有直接定义(若与当前类紧密相关)，或使用枚举类型导出，或使用不可实例化的工具类导出（然后java 1.5后就可以利用静态导入static import来避免用类名修饰常量名）。
- 类层次优先于标签类（含标签域以区分实例类型）。
- 策略模式，如排序中传入比较器函数，该比较器函数代表一种为元素排序的策略。当然当我们在设计具体的策略类的时候，还需要定义一个策略接口(strategy interface)。
- 具体策略并不一定要为公有的才可以导出。java类库的String类就是用这种模式，通过CASE_INSENSITIVE_ORDER域导出比较器。当然如果只被使用一次则可使用匿名类声明和初始化这个具体策略类。
{% highlight java %}
{% raw %}
// Exporting a concrete strategy
class Host {
    private static class StrLenCmp implements Comparator<String>, Serializable {
        public int compare(String s1, String s2) {
            return s1.length() - s2.length();
        }
    }
    // Returned comparator is serializable
    public static final Comparator<String> STRING_LENGTH_COMPARATOR = new StrLenCmp();
    ... // Bulk of class omitted
}
{% endraw %}
{% endhighlight %}
- 嵌套类(nested class)有四种：静态成员类(static member class)、非静态成员类(nonstatic member class)、匿名类(anonymous class)和局部类(local class)。除了第一种以外其他三种都被称为内部类(inner class)。
- 如果声明成员类不要求访问外围实例，就要始终用static修饰，使它成为静态成员类。

---

## 5 泛型（Generic）

- 声明中具有一个或者多个类型参数(type parameter，如类型参数E)的类或接口，就是泛型。比原生态类型(raw type)类型相比有安全性和表述性的优势，还支持原生态类型是为了提供兼容性。
- 原生态类型`List`和参数化的类型`List<Object>`之间的区别？其一，前者逃避了泛型检查，后者明确告诉编辑器，他能够持有任何类型的对象。其二，你可以将`List<String>`传给类型`List`的参数，却不能将它传给类型`List<Object>`的参数。泛型拥有子类型化(subtyping)的规则，**`List<String>`是`List`的子类型，却不是参数化类型`List<Object>`的子类型**。
- 有两个使用原生类型的例外，一是在类元对象(class literal，类型令牌)中，比如`List.class，String[].class和int.class`都合法，但`List<String.class>`和`List<?>.class`就不合法。而是在使用instanceof操作符中，因为泛型信息会在运行时被擦除。

术语             |                实例
|:-|:-|
参数化的类型     |   `List<String>`
实际类型参数     |   `String`
泛型             |   `List<E>`
形式类型参数     |   `E`
无限制通配符类型 |   `List<?>`
原生态类型       |   `List`
有限制类型参数   |   `<E extend Number>`
递归类型限制     |   `<T extend Comparable<T>`
有限制通配符类型 |   `List<? extend Number>`
泛型方法         |   `static <E> List<E> asList(E[] a)`
类型令牌         |   `string.class`


- 尽量消除非受检警告，如果非要使用注解@SuppressWarnings("unchecked")则需要把范围降低到最小，永远不要在整个类上添加该注解，也不要在方法上添加注解，应该声明一个新的局部变量赋值，然后在其前面添加该注解，因为直接放到return语句中是非法的，它不是一个声明。
- 数组与泛型相比有两个重要的不同点。首先，数组是协变的(convariant)，协变的意思是如果`Sub`为`Super`的子类型，那么数组类型`Sub[]`就是`Super[]`的子类型。而泛型是不可变的(invariant)，即`List<Type1>`不是`List<Type2>`的子类型或超类型。你可能会认为泛型有缺陷，但实际上数组才是有缺陷的。因为泛型列表可以在编译时就发现错误。
{% highlight java %}
{% raw %}
// Fails at runtime!
Object[] objectArray = new Long[1];
objectArray[0] = "I dont'fit in."; // Throws ArrayStoreException
// Aha, won't compile
List<Object> ol = new ArrayList<Long>(); // Incompatible types
ol.add("I don't fit in.");
{% endraw %}
{% endhighlight %}
- 数组和泛型的第二大区别就是数组是具体化的(reified)，因此数组在运行时才知道并检查它们的元素类型约束。而泛型是通过擦除(earsure)来实现，因此泛型只在编译时强化它们的类型信息，并在运行时擦除它们的元素类型信息。擦除就是使泛型可以与没有使用泛型的代码随意进行互用。
- 创建不可具体化(non-reifiable)的泛型数组是非法的，如果合法，则泛型系统便提供不了编译通过后运行时类型转换必然成功的基本保证。当然也有可具体化的泛型，即无限制的通配符类型，如`List<?>`或`Map<?, ?>`，虽不常用，但确实可以创建无限制通配符类型数组。
- 数组和泛型不能很好地混合使用，如果这么做并得到了编译错误或警告时，应该考虑用列表代替数组，牺牲掉一部分性能和简洁性，但却换回来更高的类型安全性和互用性。
- 如果输入参数既是生产者又是消费者，那么使用通配符参数就没有任何意义。否则根据PECS(producer-extends, consumer-super)来帮助记忆使用extends还是super，这个也被称为Get and Put Principle，或者通俗地说就是该参数(某个对象)是无私作贡献来了（生产者），还是想利用某些东西或拿点东西走（消费者）。如所有的Comparable和Comparator都是消费者。
- 不要用通配符类型作为返回类型，否则会强制用户在客户端代码中使用通配符类型。如果使用得当，通配符类型对于类的用户来说几乎是无形的。
- 如果类型参数`E`只在方法声明中出现，那么就可以用通配符`?`来代替它。即用无限制通配符`<?>`代替(无限制)类型参数`<E>`，或用有限制通配符`<? extends Someting>`代替有限制类型参数`<E extends Someting>`。当然这提供更大的灵活性，但需要在使用通配符泛型的方法中调用使用类型参数泛型的方法作为私有的的辅助方法，就是对外接口是通配符，但内部还是需要使用类型参数来实现。因为除了null外的任何值都不能放到`List<?>`中，或者说除了null外没有任何值为?类型。
- 如何构造一个异构(heterogeneous)的Map，它的所有键可以是不同类型的，而普通的Map所有的键为同一类型。
- 类Class在Java1.5以后被泛型化了，类的类型（也称为类型令牌type token）在字面上不再是简单的Class，而是`Class<T>`。例如String.class属于`Class<String>`类型，利用这个特性就可以实现类型安全的异构。
{% highlight java %}
{% raw %}
// Typesafe heterogeneous container pattern
public class Favorites {
    private Map<Class<?>, Object> favarites = new HashMap<Class<?>, Object>();
    public <T> void putFavorites(Class<T> type, T instance) {
        if (type == null) {
            throw new NullPointerException("Type is null");
        }
        // avoid raw form destroy typesafe, like checkedMap, checkedSet, checkedList in java.util.Collections
        favorites.put(type, type.cast(instanc));
    }
    public <T> T getFavorites(<Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
{% endraw %}
{% endhighlight %}
- 注意，你可能认为和之前提到的`List<?>`类似，除null外任何东西都不能放到这个Map中，但其实不然，因为这个通配符类型是嵌套的。另外一点，Map的值只能是Object，即Map不能保证键和值的类型关系，但是getFavorites方法能够并且的确重新建立了这种联系，利用Class的**cast**方法将传入参数**Object的对象**转换为类令牌代表的**参数类型T的对象**，当然类型由大往小转换必须是正确的转换，否则会抛出ClassCastException异常，但在此例中，我们可以确信favorits的值的类型始终与键的类型相匹配。Class还有一种**asSubclass**实例方法，是将**类令牌**转换为作为参数传入的**子类的类令牌**，如果成功返回该子类类令牌，否则抛出异常。
- Favorites类中不能保存不可具体化(non-reifiable)或称为非协变的类，即你可以保存String或String[]，但你不能保存`List<String>`，因为你无法为`List<String>`获取一个Class对象，`List<String>.class`是个语法错误。

---

## 6 枚举和注解

- 枚举类型是功能齐全的类。通过公有的静态final域为每个枚举常量导出实例的类。因为没有可以访问的构造器，枚举类型是真正的final，也就是说枚举类型是实例受控的。另外枚举还是先来Comparable和Serializable接口。可以用静态values方法得到所有的final实例。
- Java的printf方法注意和C语言的区别，`%n`相当于C语言的`\n`
- 介绍一种特定于常量的方法实现。这样你不会用到switch来选择对应的返回值，而且当要添加新的常量时也不会忘记提供对应apply方法，因为是紧随着的，而且如果真的忘记，也会编译不了，因为该枚举类型中所有的抽象方法都必须被覆盖。注意enum前面不需要abstract关键字。特定于常量的方法要优先于启用自有值的枚举。
{% highlight java %}
{% raw %}
// Enum type with constant-specific method implementations
public enum Operation {
    PULS   {double apply(double x, double y) {return x +ｙ;}}
    MINUS   {double apply(double x, double y) {return x -ｙ;}}
    abstract double apply(double x, double y);
}
{% endraw %}
{% endhighlight %}
- 枚举构造器不可以访问枚举的静态域，除了编译时常量域(static final）外。这一限制是有必要的，因为构造器运行的时候这些常量域还没有被初始化。
- 策略模式的枚举(strategy enum pattern)来计算工资，工资包含基本工资和加班工资，因为基本工资的计算方法统一，那么就可以将加班工资的计算转移到一个策略枚举中。如果多个枚举常量同时共享相同的行为，则应该考虑策略枚举。
{% highlight java %}
{% raw %}
// The strategy enum pattern
enum PayrollDay {
    // weekday
    MONDAY(PayType.WEEKDAY),
    TUESDAY(PayType.WEEKDAY),
    WEDNESDAY(PayType.WEEKDAY),
    THURSDAY(PayType.WEEKDAY),
    FRIDAY(PayType.WEEKDAY),
    // weekend
    SATURDAY(PayType.WEEKEND),
    SUNDAY(PayType.WEEKEND);
    private final PayType payType;
    PayrollDay(PayType payType) {
        this.payType = payType;
    }
    double pay(double horsWorked, double payRate) {
        return payType.pay(hoursWorked, payRate);
    }
    // The strategy enum type
    private enum payType {
        WEEKDAY {
            double overtimePay(double hours, double payRate) {
                return hours <= HOURS_PER_SHIFT ？ 0 : (hours - HOURS_PER_SHIFT) * payRate / 2;
            }
        },
        WEEKEND {
            double overtimePay(double hours, double payRate) {
                return hours * payRate / 2;
            }
        };
        private static final int HOUR_PER_SHIFT = 8;
        abstract double overtimePay(double hrs, payRate);
        double pay(double hoursWorded, double payRate) {
            double basePay = hoursWorked * payRate;
            return basePay + overtimePay(hoursWorked, payRate);
        }
    }
}
{% endraw %}
{% endhighlight %}
- 枚举有个小小的性能缺点，即装载和初始化枚举会有空间和时间的成本，但与int常量相比，枚举类型的优势还是不言而喻的。更易读，也更安全，功能更强大。
- 永远不要根据枚举的序数（由枚举的ordinal()方法可得出） 导出与序数关联的值，而是将它保存在一个实例域中。如用SOLO(1), DUET(2)等构造枚举。最好完全避免使用ordinal()方法，除非你在设计像EnumSet和EnumMap这种基于枚举的通用数据结构。
- 用EnumSet来代替位域。位域指的是：
{% highlight java %}
{% raw %}
// Bit field enumeration constants - OBSOLETE!
public class Text {
    public static final int STYLE_BOLD      = 1 << 0; //1
    public static final int STYLE_ITALIC    = 1 << 1; // 2
    // Parameter is bitwise OR of zero or more STYLE_ constants
    public void applyStypes(int styles) { ... }
}
{% endraw %}
{% endhighlight %}
- 这种表示法可以用OR位运算将几个常量并在一个集合中，如`text.applyStyles(STYLE_BOLD | STYLE_ITALIC);`。其实用EnumSet可以轻松实现该功能，而且避免int常量的打印、遍历等诸多缺点。所以可以用枚举来代替int常量，然后EnumSet实现联合或交际的工作，如`text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));`，注意applayStyles方法采用的是`Set<Style>`而非`EnumSet<Style>`，因为最好以接口而非实现类型作为参数以保证更大的灵活性。当然EnumSet有个缺点，截止Java 1.6版本，它都无法创建不可变的EnumSet
- 用EnumMap代替序数索引。 因EnumMap更容易扩展。如果是多维的，就使用`EnumMap<..., EnumMap<...>>`。
- 虽然无法编写可扩展的枚举类型，即枚举类型不能被继承，但可以通过编写接口然后实现该接口来达到相同的目的。如先实现了`Operation`接口的枚举类型，client用该枚举类型作为参数时就可以用`<T extents Enum<T>  & Operation>`确保Class对象既表示枚举有表示Operation的子类型。
- 注解优先于命名模式。命名模式(naming pattern)表明有些程序元素需要通过某种工具或者框架进行特殊处理。自定义注解类型：
{% highlight java %}
{% raw %}
// Marker annotation type declaration
import java.lang.annotation.*;
@Retention(RetentionPlicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Test {
}
{% endraw %}
{% endhighlight %}
- Retention和Target为元注解(meta-annotation)，表明Test注解应该在运行时保留及只能运行在方法中，但目前元注解还做不到只能运行在静态方法中之类的更细化的限制。而自定义Test注解被成为标记注解(marker annotation)，因为它没有参数，只是标记被注解的元素。接着就可以编写简单的测试类，测试被标记Test注解的方法。
{% highlight java %}
{% raw %}
// Program to process marker annotations
import java.lang.reflect.*;
public class RunTests {
    public static void main(String[] args) throws Exception {
        int test = 0;
        int passed = 0;
        Class testClass = Class.forName(args[0]);
        for (Method m: testClass.getDeclaredMethods()) {
            if (m.isAnnotationPresent(Test.class)) {
                test++;
                try {
                    m.invoke(null);
                    passed++;
                } catch (InvocationTargetException wrappedExc) {
                    Throwable exc = wrappedExc.getCause();
                    System.out.println(m + " faild: " + exc);
                } catch (Exception exc) {
                    System.out.println("INVALID @Test: " +　m);
                }
            }
        }
        System.out.println("passed: %d, Faild: %d%n",
                            passed, tests - passed);
    }
}
{% endraw %}
{% endhighlight %}
- 也可用带参数的注解，而不只是标记注解。
{% highlight java %}
{% raw %}
// Anotation type with a parameter
import java.lang.annotation.*;
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    Class<? extends Exception> value();
}
{% endraw %}
{% endhighlight %}
- 参数类型为某个扩展了Exception类的Class对象。那么使用该注解的时候就可以用类似`@ExceptionTest(ArithmeticException.class)`来使用。而在测试类中在判断(`isAnnotationPresent(ExceptionTest.class)`)是该注解的方法，然后运行(`invote`)该方法之后捕获到相应异常之后，可以由`getAnnotation(ExceptionTest.class).value()`来获得注解的参数`Class<? extends Exception>`对象。
- 坚持使用Override注解。可以保证在想覆盖时成功覆盖了方法，因为参数如果不完全相同那就变成了重载。比如想覆盖Object的equals方法，如果参数不是Object对象而写成了其子类，那么就变成了重载(Overloaded)而不是覆盖(Override)，使用HashSet或其他需要比较的集合时就会调用Object的方法造成错误。
- 用标记接口(marker interface)来定义类型。标记接口指的是没有包含方法声明的接口。如Serializable接口，通过实现这个标记接口来表明它的实例可以写到ObjectOutputStream中，即可被序列化，否则ObjectOutputStream.write(Object)方法就会失败，write方法的参应该用Serializable接口但是却用的是Object，所以只会在运行时失败，这个设计也挺令人费解。

---

## 7 方法

- 每当编写方法或构造器之时，应首先考虑它的参数有哪些限制，当然对于开销极大或计算时就会检查的情况可以忽略常位于开头的参数有效性检查。对于公有的方法，要用JavaDoc的@throws标签说明抛出异常的情况。如:
{% highlight java %}
{% raw %}
/**
 * @param m the modules, which must be positive
 * @return this mod m
 * @throws ArithmeticException if m is less than or equal to 0
 */
public BigInteger mod(BigInteger m) {
    if (m.signum() <= 0) {
        throw new ArithmeticException("Modulus <= 0: " + m);
    ... // Do the computation
    }
}
{% endraw %}
{% endhighlight %}
- Java是一门安全的语言(safe language)，意味着对于缓冲区溢出、数组越界、非法指针以及其他内存破坏错误可自动免疫，而这些错误却困扰着如C和C++等不安全语言。无论系统的其他部分发生什么事情，类的约束始终为真，对于那些把所有内存当作一个巨大的数组来看待的语言来说，这是不可能的。
- 即便在安全的语言中也要保护性地设计程序，防止某个类的客户端尽其可能地破坏该类的约束条件。如你给类的实例成员Date对象限制为私有且final，似乎是不可变的，然而Date类本身是可变的，因此很容易就破坏该约束条件。为了避免受到这种攻击，需要**对于构造器的每个可变参数进行保护性拷贝(defensive copy)**，并且使用拷贝的对象作为类实例组件，而不是使用原始的对象。
{% highlight java %}
{% raw %}
// Repaired constructor - make defensive copies of parameters
public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());
    if (this.start.compareTo(this.end) > 0) {
        throw new IllegalArgumentException(start + "after " + end);
    }
}
{% endraw %}
{% endhighlight %}
- 注意检查参数在保护性拷贝之后，这是为了避免有别的线程在此危险阶段(window of vulnerability)改变类的参数。这里的危险阶段指从检查参数开始直到拷贝参数结束之间的时间段，这也被称为Time-Of-Check/Time-Of-Use或TOCTOU攻击。
- 另外因为Date是非final的，即有可能有子类，所以没有用Date的clone方法进行拷贝，否则返回的不一定为java.util.Date对象。它有可能返回处于恶意设计的不可信子类的实例，即传进来的Date实例可能是Date对象不可信子类的实例。对于参数类型可以被不可信子类化的参数，不要使用clone方法进行保护性拷贝。
- 注意返回时也要返回可变内部域的保护性拷贝，即留给外界的都是不同于内部可变域的引用，这样外界就无法改变类内部不想被改变的内部域，即便是可变的内部域。故进来和出去都需要进行保护性拷贝，当然这会造成一定的性能影响。
{% highlight java %}
{% raw %}
// Repaired accessors - make defensive copies of internal fields
public Date start() {
    return new Date(start.getTime());
}
public Date end() {
    return new Date(end.getTime());
}
{% endraw %}
{% endhighlight %}
- 当然在上述返回方法中可以使用clone方法，因为类内部的Date对象保证是真实的java.util.Date，而不是某个潜在的不可信的子类。
- 需要用到保护性拷贝的另一种情况是使用客户提供的对象引用作为内部Set实例的元素，或者内部Map实例的键，这些都是需要保持其唯一性的，如果该对象在插入后再被修改，Set或Map的约束条件就会被破坏。
- 对于boolean参数，要优先使用只含两个元素的枚举类型，更易于阅读和编写，也便于以后添加更多的选项。
- 慎用重载方法，重载方法的选择是在编译时决定的。即重载的方法是静态的，而覆盖的方法是动态的(运行时决定，不管该子类编译时的类型是什么)。**安全而保守的策略是永远不要导出两个具有相同参数数目的重载方法，如果方法使用可变参数，保守的策略是永远不要重载它**。例如ObjectOutputStream类，并没有重载write方法应用于不同的数据类型，而是使用了住院时writeBoolean(boolean)、writeInt(int)和writeLong(long)这样的签名。而对于构造器，可以使用静态工厂。
- 加入不可避免需要重载具有相同参数数目的方法，比如正在改造一个现有的类以实现新的接口，此时就要保证当传递相同的参数时，所有重载方法的行为必须一致，即返回相同的结果。
- 慎用可变参数(Java 1.5)。可变参数方法的每次调用都会导致一次数组的分配和初始化，当然也可以先重载比如0到3个普通参数，当参数数目超过3时，就使用一个可变参数方法。
- 返回零长度的数组(new Something[])或者集合(Collections.emptySet、emptyList和emptyMap)，而不是null。因为零长度的数组或不可变的空集是不可变的，可以被自由的共享。返回null的习惯用法很有可能是从C语言程序设计中沿袭而来的，在C语言中，数组长度和实际的数组是分开返回的，因此如果返回的数组长度为0，那么再分配一个数组就没有任何好处。
- Java 1.5增加了两个重要的Javadoc标签`{@literal}`和`{@code}`。如`{@code index < 0 || index >= this.size()}`，避免被当成HTML的语法做处理。而`{@literal}`除了不以代码格式渲染文本之外，其他和`{@code}`标签一样。

---

## 8 通用程序设计

- 较早的程序设计语言如C语言要求局部变量必须在一个代码块的开头处进行声明。而这个习惯应该改正。Java允许在任何可以出现语句的地方声明变量。要使局部变量的作用域最小化，最有力的做法就是在第一次使用它的地方声明。
- 循环也可使变量的作用域最小化，如果循环结束后就不再需要循环变量的内容，for循环就有先于while循环。for循环独立的可以重用元素变量的名称，另外也比while循环更简短，从而增强了可读性。
- for-each循环优先于传统的for循环。因为for-each会稍有性能优势，因为它对数组索引的边界值只会计算一次。另外在双层循环中不会再有忘在外层循环添加一个变量来保存外层循环的元素的情况。
{% highlight java %}
{% raw %}
// The preferred idiom for iterationg over collections and arrays
for (Element e : elements) {
    doSomething(e);
}
{% endraw %}
{% endhighlight %}
- for-each循环可以应用于任何实现了Iterable接口的对象。当然在需要删除选定元素(过滤)，改变部分或全部元素值(转换)和需要并行遍历多个集合(平行迭代)时就无法使用for-each循环。
- 每个程序员都应该掌握java.lang、java.util和熟悉java.io的内容。至于其他类库的知识可以根据需要随时学习。总而言之，不要重新发明轮子。一般而言，类库的代码可能比你自己编写的代码要好一些，并且会随着时间的推移不断改进。
- 如果需要精确的答案，请避免使用float和double。因为float和double类型不能精确地表示0.1(或10的任何其他负次方值)，所以很不适合用于货币计算。
{% highlight java %}
{% raw %}
// Unfortunately, the result is 0.610000000000001
System.out.println(1.03 - .42)；
{% endraw %}
{% endhighlight %}
- 解决此问题的正确办法是使用BigDecimal、int或者long进行货币计算。使用BigDecimal会调用到其add，substact, compareTo等方法，会比较麻烦，另外也比较慢。当然也可以先换算到最小单位，然后不超过9位十进制数字用int，不超过18位用long。更大的数字则必须使用BigDecimal。
- 基本类型(primitive)优先于装箱基本类型(boxed primitive)。两个装箱基本类型使用`==`时比较，比较的是hashCode的值。当在一项操作中混合使用基本类型和装箱基本类型时，装箱类型几乎都会自动拆箱，但有一种情况例外，就是如果null对象引用被自动拆箱，就会得到一个NullPointerException异常。另外装箱基本类型与基本类型相比，会造成明显的性能下降。当然在特定情况中必须使用基本装箱类型，一是集合中的元素、键和值之中，二是在泛型类型参数中，还有最后一种情况是在进行反射方法调用时。
- 如果其他类型更适合，尽量避免使用字符串。经常被错误地用字符串来代替的类型包括基本数据类型，枚举类型和聚集类型。
- 当心字符串连接符的性能，为连接n个字符而重复地使用字符串连接操作符，需要n的平方级的时间。此时应该考虑使用StringBuilder的append方法。
- 接口优先于反射机制。核心反射机制(core reflection facility) java.lang.reflect，提供了“通过程序来访问关于已加载的类的信息”的能力。给定一个Clas实例，你可以获得Constructor、Method和field实例，这样就可以使你通过反射机制操作它们的底层对等体。例如Method.invoke可以让你调用任何类的任何对象上的任何方法（遵从常规的安全限制）。反射机制丧失了编译时类型检查的好处，所需的代码相当笨拙和冗长，性能损失。
- 核心反射机制最初是为了急于组件的应用创建工具而设计的。另外有一些复杂的应用程序也需要反射机制，例如类浏览器、对象监视器、代码分析工具、解释型的内嵌式系统和RPC(远程系统调用)等。如果适当的构造器不带参数，甚至根本不需要使用java.lang.reflect包，Class.newInstance方法就已经提供了所需的功能。例如下面的程序创建了一个`Set<String>`实例，它的类是由第一个命令行参数指定的，该程序把命令行生育的参数插入到这个集合中，然后打印该集合。
{% highlight java %}
{% raw %}
// Reflective instantiation with interface access
public static void main(String[] args) {
    // Traslate the class name into a Class Object
    Class<?> cl = null;
    try {
        cl = Class.forName(args[0]);
    } catch(ClassNotFoundException e) {
        System.err.println("Class not found.");
        System.exit(1);
    }

    // Instantiate the class
    Set<String> s = null;
    try {
        s = (Set<String>) (cl.newInstance());
    } catch(IllegalAccessException e) {
        System.err.println("Class not accessible.");
        System.exit(1);
    } catch(InstantiationException e) {
        System.err.println("Class not instantiable.");
        System.exit(1);
    }
    // 20 lines to create instance from its ClassName
    // Exercise the set
    s.addAll(Arrays.asList(args)).subList(1, args.length));
    System.ouy.print(s);
}
{% endraw %}
{% endhighlight %}
- 谨慎使用本地方法(native mathod，本地设计语言如C或C++编写的特殊方法)。Java Native Interface(JNI)允许Java应用程序可以调用本地方法。尽可能少用本地方法，并且要进行全面的测试。
- 本地方法主要有三种用途。一是提供访问特定于某种平台的机制的能力，如访问注册表，文件锁，二是提供访问遗留代码库和遗留数据(lagacy data)的能力，最后有可能会提高程序的性能。但也带来一些严重的缺点，因为本地语言不是安全的，另外本地语言是与平台相关的，进入和退出本地语言时需要相关的固定开销。
- 谨慎的进行优化，特别是不成熟的优化。不要因为性能而牺牲合理的结构。在要对优化前后的性能进行测量比较。性能剖析工具有助于你决定应该把优化的重心放在何处，通常再多的底层优化也无法弥补算法选择不当所造成的性能损失。总之，不要费力去编写快速的程序，而应该努力编写好的程序，速度自然会随之而来。
- 努力避免那些限制性能的设计，难以在日后更改的组件有API、线路层(wire-level)协议以及永久数据格式。
- 要考虑API设计决策的性能后果。如使共有的类型成为可变的(mutable)，这可能会导致大量不必要的保护性拷贝。还有在适合使用复合模式的共有类中使用继承，这种耦合性大大限制了子类的性能。此外，在API中使用实现类型而不是接口，会把你限制在一个具体的实现上，难以在将来使用更快的实现。
- 遵守普遍接受的命名惯例。主要分为字面惯例和语法惯例。
- 字面惯例中，关于缩写：包名尽量用有意义的缩写形式，局部变量名可用缩写形式，类和接口名包括枚举和注解类型名称、成员名都应避免使用缩写，除非是通用的缩写如Max、Min、Http、Url等，另外即使是首字母缩写也应该用仅首字母大写的形式，即选择使用HttpUrl而不是HTTPURL。
- 字面惯例中，关于类型参数名称：通常使用下面五个字母，T表示任意的类型，E表示集合的元素类型，K和V表示映射的键和值类型，X表示异常。任何类型的序列可以是T、U、V(按顺序顺延)也可以是T1、T2、T3(加上数字序号)。
- 语法惯例中，返回被调用对象的一个函数或属性，可以用名词、名词短语、或者以get开头的动词短语，如size、hashCode、getTime。并不要求一定是第三种的get开头的形式，前两种往往会产生可读性更好的代码。当然如果所在的类是Bean[JavaBeans，即get和set配套]，就要强制使用以get开头的形式。
- 语法惯例中，静态方法的常用名称为valueOf、of、getInstance、newInstance、getType和newType。

---

## 9 异常

- 异常应该只用于异常的情况下，而不要用于正常的控制流中。如不要用异常控制数组越界情况和取代Iterator的hasNext方法等。
- 对可恢复的情况使用受检异常，对编程错误使用运行时异常。Java提供了三种可抛出(throwable)结构，受检的异常(checked excption)、运行时异常(run-time exception)和错误(error)。其中错误往往被JVM保留用于表示资源不足、约束失败，或者其他使程序无法继续执行的条件，因此最好不要再实现任何新的Error子类。受检的异常往往指明了可恢复的条件。如用户没有储存足够数量的钱，他企图在一个收费电话上进行呼叫就会失败，于是抛出受检的异常。这个异常应该提供一个访问方法，以便客户查询所缺的费用金额，从而可以将这个数值传递给电话用户。
- 优先使用标准的异常，常用的异常有IllegalArgumentException、IllegalStateException、NullPointerException、IndexOutOfBoundsException、ConcurrentModificationException、UnsupportedOpeationException。
- 更高层的实现应该捕获低层的异常，同时抛出可以按照高层抽象解释的异常，称为异常转移(exception traslation)。因为如果高层的实现在后续的发行版本中发生了变化，它所抛出的异常也可能会跟着发生变化，从而可能会破坏现有的客户端程序。
- 每个方法抛出的异常都要有文档，单独声明受检的异常，并且利用Javadoc的@throws标记，准确地记录下抛出抛出的一个或多个异常的条件，包括可能会抛出的未受检异常。这样就可以有效地描述某个方法被成功执行的前提条件(preconditin)。
- 使用JavaDoc的@throws标签记录方法可能抛出的一个受检异常和未受检异常，然后在声明中用throws开头的短语来表示受检异常，而未受检异常则不要出现throws，这样就可以区分这两者。
- 失败的方法调用应该使对象保持在被调用之前的状态，这种属性称为失败原子性(failure atomic)。
- 获得失败原子性的方法有四种，**第一种**是设计一种不可变对象或在执行操作之前检查参数的有效性；**第二种**是调整处理过程的顺序，使得任何可能失败的计算部分都在对象状态被修改前发生；**第三种**是编写一段恢复代码，主要用于永久性(基于磁盘)的操作；**最后一种**是在对象的一份临时拷贝上执行操作，当操作完成后，再用临时拷贝中的结果代替对象的内容。适用于数据保存在临时的数据结构中，计算过程会加速，如Colletions.sort在执行排序之前，会把输入列表转到一个数组中。
- 不要忽略异常，如使用空语句catch。这相当于把火警的预警器关掉了。有一种情况可以忽略，就是关闭FileInputStream的时候，因为你还处于改变文件状态之前。当然即便是被忽略的异常，也要进行额外说明，解释为何忽略该异常。

---

## 10 并发

- 同步不仅可以阻止一个进程看到对象处于不同的状态之中，它还可以保证进入同步方法或者同步代码块的每个线程，都能看到由同一个锁保护的之前所有的修改效果。
- Java语言规范保证读或者写一个变量是原子的(atomic)，除非这个变量的类型为long或double。即便多个线程在没有同步的情况下并发地修改这个变量也是如此。
{% highlight java %}
{% raw %}
// Broken - How long would you expect this program to run?
public class StopThread {
    private static boolean stopRequested;
    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(new Runnable() {
            public void run() {
                int i = 0;
                while (!stopRequested) {
                    i++;
                }
            }
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
{% endraw %}
{% endhighlight %}
- 你可能期望上面这个程序运行大概一秒左右便会停止，但实际上这个程序可能永远不会停止。因为没有同步，就不能保证后台线程何时**“看到”**主线程对stopRequested所做的改变。没有同步，虚拟机改为了：
{% highlight java %}
{% raw %}
if (!stopRequestd){
    while (true) {
        i++;
    }
}
{% endraw %}
{% endhighlight %}
- 这种优化成为提升(hoisting)，正是HopSpot Server VM的工作，结果造成活性失败(liveness failure)，指该程序无法前进。修正该问题的一种方式是同步访问stopRequested域。
{% highlight java %}
{% raw %}
// Properly synchronized cooperative thread termination
public class StopThread {
    private static boolean stopRequested;
    private static synchronized void requestStop() {
        stopRequested = true;
    }
    private static synchronized boolean stopRequested() {
        return stopRequested;
    }
    public static void main(String[] args) {
        Thread backgroundThread = new Thread(new Runnable()) {
            int i = 0;
            while (!stopRequested()) {
                i++;
            }
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        requestStop();
    }
}
{% endraw %}
{% endhighlight %}
- 读和写都同步的情况下同步才会起作用。第二种方法是使用volatile修饰符，它不执行互斥访问，但它可以保证任何一个线程在读取该域的时候都将看到最近刚刚被写入的值。但在使用volatile的时候需要格外小心。考虑下面的方法，假设要产生序列号：
{% highlight java %}
{% raw %}
// Broken - requeires synchronization!
private static volatile int nextSerialNumber = 0;
public static int generateSerialNumber() {
    return nextSerialNumber++;
}
{% endraw %}
{% endhighlight %}
- 然而该方法并不能正常工作，因为增量操作符(`++`)不是原子的，它会执行两个操作，首先它读取值，然后写回一个新值。这样可能在多个线程访问时得到相同的序号值。这就会造成了安全性失败(safety failure)，表示程序会计算出错误的结果。
- 此外还可以使用类AtomicLong，来同步地执行加减操作。
{% highlight java %}
{% raw %}
private static final AtomicLong nextSerialNum = new AtomicLong();
public static long generateSerialNumber() {
    return nextSerialNum.getAndIncrement();
}
{% endraw %}
{% endhighlight %}
- 当然避免上述问题的最佳办法是不共享可变的数据，即把可变数据限制在单个线程中。
- 在一个被同步的区域内部，不要调用设计成要被覆盖的方法，或者是由客户端以函数的形式提供的方法。从包含该同步区域的类的角度来看，这些方法都是外来的(alien)。调用这些方法可能会导致异常、死锁或数据损坏。
- 在同步方法之外调用的方法称为开放调用(open call)，除了可以避免死锁之外，开放调用还可以极大地增加并发性。通常，你应该在同步区域内做尽可能少的工作。
- 不要过度同步，过度同步的实际成本并不是指获取锁所需的时间，而是指失去了并行的机会。当你有足够理由一定要在内部同步类的时候，才应该这么做，并且应该将这个决定清楚的写到文档中。考虑StringBuffer(同步类，几乎被弃用)和StringBuider(非同步类)。
- ExecutorService和task优先于线程。
{% highlight java %}
{% raw %}
ExecutorService executor = Executors.newSingleThreadExecutor();
executor.execute(runnable);
executor.shutdown();
{% endraw %}
{% endhighlight %}
- Excutors类还提供了许多静态工厂，可以提过所需的大多数ExcutorService，还可以直接使用ThreadPoolExcutor类，这个类允许你控制线程池操作的几乎所有方面。
- 如果编写的是小程序或者轻载服务器，使用**Excutors.newCachedThreadPool**通常是不错的选择，在缓存的线程池中，被提交的任务没有排成队列，而是直接交给线程执行。如果是负载很重的服务器则需要选择**Executors.newFixedThreadPool**，提供一个包含固定线程数目的线程池，当然还可以直接使用**ThreadPoolExcutor**类。如果是长时间的定时任务，则应该避免使用**java.util.Timer**，因为timer只用一个线程来执行任务，对于长期运行的任务可能会影响准确性，且容易抛出异常而终止，取而代之的，可以使用**SceduledThreadPoolExecutor**，支持多个线程，并且可以优雅地从抛出未受检异常的任务中恢复。
- 你不仅尽量不要编写自己的工作队列，而且还应该尽量不直接使用线程。以前的线程抽象Thread既充当工作单元，又充当执行机制。现在分成了ExcutorService(抽象的执行机制)和task(抽象的工作单元)。
- 并发工具优先于wait和notify。java.util.concurrent中更高级的工具分为三类：执行器框架(**Excutor Framwork**)、并发集合(**Concurrent Collection**)以及同步器(**Synchronizer**)。
- 并发集合为标准的集合接口(如List、Queue和Map)提供了高性能的并发实现。但这也意味着客户无法原子地对并发集合进行方法调用。因此有些集合接口已经通过依赖状态的修改操作(self-dependent modify operation)进行了扩展。例如ConcurentMap扩展了Map接口，并添加了几个方法，包括PulfAbsent(key, value)，会返回前一个关联的值，如果没有则返回null。这样实现线程安全的Map就容易一些了。
- 有些集合接口已经通过阻塞操作(boocking operation)进行了扩展，他们会一直等待(或者阻塞)到可以成功执行为止。生产者-消费者队列中常会用到，当然大多数ExecutorService的实现(包括ThreadPoolExecutor)都使用了BlockingQueue。
- 同步器(Synchronizer)的作用是使一个线程能够等待另一个线程，并允许它们协调动作。最常用的同步器是**CountDownLatch**和**Semapore**。另外还有**CyclicBarrier**和**Exchanger**。
- 倒计数锁存器(CountDownLatch)是一次性的障碍，允许一个或者多个线程等待一个或者多个其他线程来做这些事情。CountDownLatch的唯一构造器带有一个int的参数，表示所有等待的线程在被允许处理之前，必须在锁存器上调用countDown方法的次数。
- 如果你在维护使用wait和notify代码，无比始终在while内部调用wait，优先使用notifyAll而不是notify。这样可以避免恶意通知和恶意等待，以确保程序的活性(liveness)。
- 线程安全性的级别有：
    1. 不可变的(immutable)，这个类的实例是不可变的，如String、Long和BigInteger。
    2. 无条件的线程安全(unconditionally thread-safe)，这个类有着足够的内部同步。如Random和ConcurrentHashMap。
    3. 有条件的线程安全(conditionally thread-safe)，其中的部分方法需要外部同步。如Collections.synchronized返回的集合，它们的迭代器(iterator)要求外部同步。
    4. 非线程安全(not thread-safe)，如ArrayList和HashMap。
    5. 线程对立(thread-hostile)，这个类即使使用了外部同步也不能安全地被多个并发使用。如System.runFinalizersOnExit，但已经被废弃了。几乎没有人会编写一个线程对立的类。
- 文档中应该清楚地说明一个类所支持的线程安全性级别，并应小心对待有条件的线程安全。你必须指明哪个调用序列需要外部同步，还要指明需要获得哪些锁，通常是作用在实例自身上的那把锁，但也有例外，比如一个对象代表了另一个对象的一个视图(view)，可能是另一个对象实例，以免其他线程修改后台对象。当用interator(语法上可以为隐式的for-each)遍历Collections.synchronizedMap返回的集合视图时，用户必须手动对其进行同步。
{% highlight java %}
{% raw %}
Map<K, V> m = collections.synchronizedMap(new HashMap<K, V>());
...
Set<K> s = m.keySet();  // Needn't be in synchronized block
...
sychronized(m) {   // Synchronizing on m, not s!
    for (K key : s) {
        key.f();
    }
}
{% endraw %}
{% endhighlight %}
- 为避免恶意地长时间持有公有访问锁的拒绝服务(denial-of service)攻击，应该使用一个私有锁对象来代替同步的方法，当然此方法显然只能用于**无条件的线程安全**上。这样还可以防止子类在无意中妨碍基类的操作。
{% highlight java %}
{% raw %}
// private lock object idiom - thwart denial-of-service attack
private final object lock = new Object();
public void foo() {
    synchronized(lock) {
        ...
    }
}
{% endraw %}
{% endhighlight %}
- 延迟初始化(lazy initialization)指的是域的值延迟到需要它时才初始化的行为。如果永远不需要这个值，那这个域永远得不到初始化。这种方法既适用于静态域，也适用于实例域。
- 使用延迟初始化的原则和其他优化一样，因为是一把双刃剑，它降低了初始化类或者创建实例的开销，却增加了访问被延迟初始化的域的开销。如果域只在类的实例部分被访问，并且初始化这个域的开销很大，那么就值得将其延迟初始化。
- 延迟初始化需要考虑同步的问题，如果要对静态域使用延迟初始化，就应该使用静态内部类(Lazy initialization holder class)模式。
{% highlight java %}
{% raw %}
// Lazy initialization holder class idiom for  static fields
private static class FieldHolder {
    static final FieldType field = computeFieldValue();
}
static FieldType getField() {
    return FieldHoder.field;
}
{% endraw %}
{% endhighlight %}
- 当上面例子中的getField方法第一次被调用时，它第一次读取FieldHolder.field，导致FieldHolder类得到初始化。这种模式利用的思想是一个类直到被使用时才被初始化，而类初始化的过程是非并行的,因此实际上并没有增加任何访问成本。
- 如果要对实例域使用延迟初始化，就应该使用双重检查(double-check)模式。这种模式避免了在域被初始化后再访问这个域时的锁定开销。该实例域需要被声明为volatile。
{% highlight java %}
{% raw %}
// Double-check idiom for lazy initialization of instance fields
private volatile FieldType field;
FieldType getField() {
    FieldType result = field;
    if (result == null) {  // First check (no locking)
        synchronized(this) {
            result = field;
            if (result = null) {  // Second check (with locking)
                field = result = computeFieldValue();
            }
        }
    }
    return result;
}
{% endraw %}
{% endhighlight %}
- 解释一下局部变量result的作用是确保field只在已经被初始化的情况下读取一次，防止活性失败(lireness failure)。而且可以提升性能，因为给低级的并发编程应用了一些标准而更优雅，使其比不使用局部变量的方法快了约25%。
- 静态内部类模式和双重检查模式都可以用来构建单例模式，当然构建单例模式最好的方法还是使用枚举。另外对于可以接受重复初始化的实例域，也可以考虑使用单重检查模式，此时该实例域也要被声明为volatile。
- 任何依赖于线程调度器来达到正确性或者性能要求的程序，很有可能都是不可移植的。最好的办法是确保**可运行线程**的平均数量不明显多于处理器的数量。注意线程的总数量 = 可运行线程数量 + 不可运行线程数量。
- 如果线程没有在做有意义的工作就不应该运行，线程不应该一直处于忙-等(busy-wait)的状态，即反复地检查一个共享对象，以等待满足某个条件。不要让程序的正确性依赖于**Thread.yield**或**线程优先级**，否则得到的应用程序既不健壮也不具有可移植性。应该使用Thread.sleep(1)代替Thread.yield来增加线程并发性达到测试的目的，不要使用Thread.sleep(0)，它会立即返回。而调整线程优先级可用于提高已经能够正常工作的程序的服务质量，而不应该用于修复不能正常工作的程序。
- 避免使用线程组ThreadGroup。ThreadGroup的一些基本功能已经被废弃，剩下的也很少使用，请用Java 1.5新增的Thread的setUncaughtExceptionHandler方法代替ThreadGroup的uncaughtException方法以获得未被捕获异常的控制权。其初衷是作为一种隔离applet(小程序)的机制，本身是出于安全的考虑，却在安全方面有着严重缺陷。故可看作一个不成功的试验，直接忽略之。

---

## 11 序列化

- 将一个对象编码成一个字节流，称为将该对象序列化(serializing)，相反的过程称为反序列化(deserilizing)。一旦对象被序列化，它的对象就可以从一台正在运行的虚拟机传递到另一台虚拟机上，或者被存储到到磁盘上，供以后反序列化使用。序列化技术为远程通信技术提供了标准的线路级(wire-level)对象表示法，也为JavaBeans组件结构提供了标准的持久化数据格式。
- 语义上实现序列化只需在类的声明中加上implementes Serializable(实现Serializable标记接口)即可，然而需要注意的是一旦一个类实现了序列化，其字节流编码(serialized form)就变成了它的导出API的一部分。
- 如果你不设计一种自定义的序列化形式(cunstom serialized form)，而仅仅接受了默认的序列化形式，则这个类中私有的和包级私有的实例域都将变成导出的API的一部分，这不符合最低限度的访问域原则。要使用@serial标签告诉JavaDoc工具，指出序列化中包含的所有域。
- 每个可序列的类都有一个唯一标识号与它相关联。如果你没有显示地定义`private static final long serialVersionUID`，系统则会根据类名、接口名以及所有公有的和受保护的成员名自动在运行时生成一个标识号。如果你通过任何方式改变了这些信息，自动产生的序列版本UID也会发生变化，兼容性将会遭到破坏。因此应该显示地定义序列号版本UID。
- 通常情况下，对象是通过构造器创建的，序列化机制是一种语言之外的对象创建机制(extralinguistic mechanism)。反序列化机制是一个隐藏的构造器。而默认的反序列化机制，很容易使对象的约束关系遭到破坏，以及遭受非法访问。
- 随着类发行新的版本，相关的测试负担也增加了。因为要满足兼容性，所以就要检查是否可以在新版本中序列化一个实例，然后在在旧版本中反序列以及相反的情况。
- 根据经验，如Data和BigInteger这样的值类、大多数集合类应该实现Serializable。而为了继承而设计的类、内部类、代表活动实体的类如线程池(Thread Pool)一般不应该实现Serializable。内部类会使用编译器产生的合成域(synthetic field)来保存外围实例(enclosing instance)的引用，以及保存来自外围作用域的局部变量的值。这些域如何对应到类定义中没有明确的规定就像没有指定匿名类和局部类(在方法中的类)的名称一样。即内部类的默认序列化形式是无清晰定义的。如果是静态成员类(static member class)则可以实现Serializable接口。
- 当然也会有例外，在为了继承而设计的类中，真正实现了Serializable接口的有**Throwable**类、**Compoent**抽象类和**HttpServlet**抽象类。对于Thowable类，RMI的异常可以从服务器传到客户端。对于Component类，GUI可以被发送、保存和恢复。对于HttpServlet，会话状态(session state)可以被缓存。为了继承而设计的类在允许子类实现Serializable接口和禁止子类实现Serializable接口两者之间的一个折衷方案是提供一个可访问的无参构造器。这种设计方案允许但不要求子类实现Serializable接口。
- 如果一个对象的物理表示法等同于它的逻辑内容，可能就适用于使用默认的序列形式。即便使用了默认的序列化形式，通常还必须提供一个readObject方法以保证约束关系和安全性。
{% highlight java %}
{% raw %}
// Good condidate for default serialized form
public class Name implements Serializable {
    /**
     * Last Name. Must be non-null.
     * @serial
     */
    private final String lastName;
    /**
     * First Name. Must be non-null.
     * @serial
     */
    private final String firstName;
}
{% endraw %}
{% endhighlight %}
- 从逻辑的角度而言，这个Name类包含两个字符串，分别代表姓和名，实例域精确地反映了它的逻辑内容。此时可以使用默认的序列化形式然后再提供一个readObject方法。@serial标签告诉Javadoc工具，把这些文档信息放到有关序列化形式的特殊文档页中。
- 而下面的类则是另一个极端情况。
{% highlight java %}
{% raw %}
// Awful candidate for default serialized form
public final class StringList implements Serializable {
    private int size = 0;
    private Entry head = null;
    private static class Entry implements Serializable {
        String data;
        Entry next;
        Entry previous;
    }
    ... // Remainder omitted
}
{% endraw %}
{% endhighlight %}
- 从逻辑意义上讲，这个StringList类表示一个字符串序列，但从物理意义上讲，它把该序列表示成了一个双向列表。如果你接受了默认的序列化形式，该序列化形式将不遗余力地镜像出(mirror)链表中的所有项，以及这些项之间的所有双向链表。此时就不能使用默认的序列化形式。否则它会使这个类的导出API永远束缚在该类的内部表示法上，且会消耗过多的时间和空间，还有可能会引起堆栈溢出(因为是递归遍历，具体什么时候会溢出，取决于Heap Size的-Xms和-Xmx的值等)。
- 下面为自定义序列化后的StringList类，transient修饰符表示这个实例域从一个类的默认序列化形式中省略掉。
{% highlight java %}
{% raw %}
// StringList with a reasonable cunstom serialized form
public final class StringList implements Serializable {
    private transient int size = 0;
    private transient Entry head = null;
    // No longer Serializable!
    private static class Entry {
        String data;
        Entry next;
        Entry previous;
    }
    // Appends the specified string to the list
    public final void add(String s) { ... }
    /**
     * Serialize this {@code StringList} instance.
     *
     * @serialData The size of the list (the number of strings
     * it contains) is emitted ({@code int}), followed by all of
     * its elements (each a {@code String)
, in the proper sequance.
     */
    private void writeObject(ObjectOutputStream s) throws IOException {
        s.defaultWriteObject();
        s.writeInt(size);
        // Write out all elements in the proper order
        // for (Entry e = head; e != null; e = e.next) {
            s.writeObject(e.data)
        }
    }
    private void raadObject(ObjectInputStream s)
            throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        int numElements = s.readInt();
        // Read in all elements and insert them in list
        for (int i = 0; i < numElements; i++） {
            add((String) s.readObject());
        }
        ... // Reamainder omitted
    }
}
{% endraw %}
{% endhighlight %}
- 即便所有的实例域都是transient的，也建议调用defaultWriteObject和defaultReadObject，从而极大地增强灵活性，允许以后的版本增加非transient的实例域。注意尽管wirteObject是私有的，但也应该有文档注释，使用了方法的@serialData标签，同域的@serial标签的作用一样，因为该方法定义了一个API，即序列化形式。
- 大多数实例域都应该被标记为transient，通常有，冗余的域，即可以根据其他基本数据域计算出来的，如缓存起来的散列值；其值依赖于JVM某一次运行的域，比如long域代表一个指向本地数据结构的指针。在决定将一个域作为非transient域之前，一定要确保它的值将是该对象逻辑状态的一部分。
- 对于默认的序列化形式，在反序列化的时候会把标记为transient的域初始化为它们的默认值。对象引用域为null，基本数据类型域为0，boolean域为false。通常还必须再提供一个readObject方法，调用readDefaultObject后将这些值恢复为可接受的值，或者可以延迟到第一次被使用时才真正被初始化。
- 如果在读取整个对象状态的某个方法上使用了同步，则必须在对象序列化上writeObject上使用相同的同步。
- 你应该分配足够多的时间来设计类的序列化形式，就好象分配足够多的时间来设计它的导出方法一样。因为它们将永久地保存下去，以确保序列化的兼容性。
- 对于包含可变私有域的对象，原对象已对其进行了保护性拷贝。在将其序列之时，需要编写readObject方法同样进行保护性拷贝。不严格地说，readObject是一个用字节流作为唯一参数的构造器。
{% highlight java %}
{% raw %}
// readObject method with defensive copying and validity checking
private void readObject(ObjectInputStream s) throws IOException {
    s.defaultReadObject();
    // Defensively copy our mutable components
    start = new Data(start.getTime());
    end = new Data(end.getTime());
    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0) {
        throw new InvalidObjectException(start + " after " + end);
    }
}
{% endraw %}
{% endhighlight %}
- 当然这会有限制，就是为了使用上述的readObject方法，start域和end域必须设置为非final的。但这已经是比较好的解决办法。Java 1.4为了阻止恶意的对象引用攻击，同时节省保护性拷贝的开销，在ObjectOutputStream中增加了writeUnshared和readUnshared方法，遗憾的是这些方法很容易受到复杂的攻击。
- 另外对于非final的可序列化的类，readObject方法和构造器类似，不可以调用可以被覆盖的方法。否则被覆盖的方法将在子类的状态被反序列化(被初始化)之前先运行，程序可能会产生错误。
- 对于实例的个数控制，枚举类型优先于readResolve。对于下述的静态单例模式:
{% highlight java %}
{% raw %}
public class Elvis {
    public static final Elvis INSTSANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
{% endraw %}
{% endhighlight %}
- 如果该类实现了序列化，它就不再是一个Singleton，无论是使用默认的还是自定义的序列化模式，无论是否提供了显示的readObject方法。因为readObject方法都会返回一个新建的实例，该实例不同于类初始化时创建的实例。
- readResolve特性允许你用readObject创建的实例代替反序列化新建的实例。在反序列化后，新建对象上的readResolve方法就会被调用，然后该方法返回的对象引用将被返回，取代新建的对象。如果是**final**的类，则可以将readResolve的修饰符设置为private。如果是**非final**的类，且readResolve是受保护或私有的，**且子类没有覆盖它**，则子类进行反序列化时就会产生并返回一个超类实例，此时有可能会导致ClassCassException，而将readResolve设置为private时则子类就无法访问该方法。
{% highlight java %}
{% raw %}
// readResolve for instance control - you can do better
private Object readResolve() {
    // Return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator
    return INSTANCE;
}
{% endraw %}
{% endhighlight %}
- 依赖readResolve进行实例控制的方法有致命的缺点，其他带有对象引用类型的实例域都必须声明为transient。否则该域的内容就可以在Singleton的readResolve方法运行之前被反序列化，此时就可以运行一个精心制作的流(修改字节)盗用指向最初被反序列的Singleton的引用。所以你得保证该类的所有实例域都为基本类型或transient的。
- 使用枚举可以以轻松地实现可序列的单例模式。
{% highlight java %}
{% raw %}
// Enum sigleton - the preferred approach
public enum Elvis {
    INSTANCE;
    private String[] favoriteSongs = {"Hound Dog", "Heartbreak Hotel"};
    public void printFavorites() {
        System.out.println(Array.toString(favoriteSongs));
    }
}
{% endraw %}
{% endhighlight %}
- 考虑使用序列化代理(serialization proxy)代替序列化实例。序列化代理为一个私有的静态嵌套类，可以精确地表示外围类的逻辑状态，外围类要求final的，以确保外围类是真正不可变的。它有一个单独的构造器，其参数类型就是外围类。该构造器从传入的外围类中复制数据，不需要进行任何一致性检查或保护性拷贝。外围类和其序列化代理都需声明实现了Serializable接口。
{% highlight java %}
{% raw %}
// Serialization proxy for Period class
private static class SerializationProxy implements Serializable {
    private final Date start;
    private final Date end;
    SerializationProxy(Period p) {
        this.start = p.start;
        this.end = p.end;
    }
    private static final long serialVersionUID = 234098243823485285L; // Any number will do
}
{% endraw %}
{% endhighlight %}
- 接下来，将下面的writeReplace方法添加到外围类中。writeReplace方法在外围类序列化之前，将外围类的实例转变成了它的序列化代理。
{% highlight java %}
{% raw %}
// writeReplace method for the serilization proxy pattern
private Object writeReplace() {
    return new SerializationProxy(this);
}
{% endraw %}
{% endhighlight %}
- 虽然此时该序列化系统永远无法产生外围序列化实例，但是攻击者有可能伪造破坏其约束条件。所以还需增加一个readObject方法。
{% highlight java %}
{% raw %}
// readObject method for the serialization proxy pattern
private void readObject(ObjectInputStream stream) throws InvalidObjectException {
    throw new InvalidObjectException("Proxy required");
}
{% endraw %}
{% endhighlight %}
- 另外，还需要提供一个readResolve方法，返回一个逻辑上相当的外围类的实例。反序列化时将序列化代理转变回外围类的实例。如果该类的静态工厂或构造器建立了这些约束条件，并且它的实例方法在维持着这些约束条件，你就可以确信序列化也会维持着这些约束条件。所以readResolve所做的事只是返回用构造器新建的已确保满足约束条件的外围类实例即可。
{% highlight java %}
{% raw %}
// readResolve method for Period.SerializationProxy
private Object readResolve() {
    return new Period(start, end); // Uses public constructor
}
{% endraw %}
{% endhighlight %}
- 如保护性拷贝方法一样，序列化代理方法可以阻止伪字节流的攻击以及内部域的盗用攻击。不同点有序列化代理方法的外围类必须为final的，序列化代理允许反序列化实例返回与原始实例不同的类，如可用于返回EnumSet(因其底层实现会有小于等于64个元素时为RegularEnumSet，否则为JumboEnumSet)。
- 序列化代理有两个局限性，它不能与可以被客户端扩展的类兼容，它也不能与对象视图中包含循环的某些类兼容。你不能从readResolve方法内部调用外围对象中的方法，否则会得到一个ClassCastException异常，因为你还没有这个对象，只有它的序列化代理。但序列化所增强的功能和安全性是没有额外的开销的。

---
The End.

zhlinh

Email: zhlinhng@gmail.com

2016-08-23
