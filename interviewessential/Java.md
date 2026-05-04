# Java

# 接口和抽象类的区别?

从职责分析上来讲:

1. 接口主要强调的指定规范
2. 而抽象类强调是复用性,如果多个实现类中有相同的可以复用的代码,就可以加一层抽象类

从区别上来讲:

1. 方法定义: 接口抽象类,最明显的就是接口中定义了一些方法,接口中只有抽象方法,java8可以有默认实现
2. 修饰符: 抽象类中可以有public protected private 等修饰符,而接口中全部都是public修饰符,接口中如果定义了变量,就需要初始化;
3. 构造器: 抽象类可以有构造器,接口中不能有构造器;

使用方式上:

接口可以被实现并且是多实现,抽象类可以被继承但只能单继承;

#

# Polymorphism:

essential condition:

1. **<font style="color:rgb(34, 34, 34);">继承或接口实现</font>**<font style="color:rgb(34, 34, 34);">：建立父子类或接口与实现类的关系。</font>
2. **<font style="color:rgb(34, 34, 34);">方法重写</font>**<font style="color:rgb(34, 34, 34);">：子类/实现类需覆盖父类/接口的方法。</font>
3. **<font style="color:rgb(34, 34, 34);">向上转型</font>**<font style="color:rgb(34, 34, 34);">：父类/接口引用指向子类对象（如</font><code><font style="color:rgb(34, 34, 34);">Animal a = new Dog()</font></code><font style="color:rgb(34, 34, 34);">）。</font>

# 抛出异常和捕获异常的区别

1. **<font style="color:rgb(34, 34, 34);">抛出异常</font>**<font style="color:rgb(34, 34, 34);"> </font><font style="color:rgb(34, 34, 34);">→ 汽车油表亮红灯（</font>**<font style="color:rgb(34, 34, 34);">问题发生</font>**<font style="color:rgb(34, 34, 34);">）</font>
   * **<font style="color:rgb(34, 34, 34);">结果</font>**<font style="color:rgb(34, 34, 34);">：车辆无法继续行驶（</font>**<font style="color:rgb(34, 34, 34);">当前方法中断</font>**<font style="color:rgb(34, 34, 34);">）</font>
2. **<font style="color:rgb(34, 34, 34);">捕获异常</font>**<font style="color:rgb(34, 34, 34);"> </font><font style="color:rgb(34, 34, 34);">→ 您停车加油（</font>**<font style="color:rgb(34, 34, 34);">处理问题</font>**<font style="color:rgb(34, 34, 34);">）</font>
   * **<font style="color:rgb(34, 34, 34);">结果</font>**<font style="color:rgb(34, 34, 34);">：加油后您继续驾驶（</font>**<font style="color:rgb(34, 34, 34);">程序恢复执行</font>**<font style="color:rgb(34, 34, 34);">）</font>

<font style="color:rgb(34, 34, 34);">❌</font><font style="color:rgb(34, 34, 34);"> </font>**<font style="color:rgb(34, 34, 34);">未捕获的异常</font>**<font style="color:rgb(34, 34, 34);"> = 无视油表红灯强行驾驶 → 最终抛锚（</font>**<font style="color:rgb(34, 34, 34);">程序崩溃</font>**<font style="color:rgb(34, 34, 34);">）</font>

# ArrayList的扩容机制:

```java
首先是他的构造方式
JDK8
默认创建一个长度为10的空数组
JDK17
无参构造函数的时候,不会创建一个10的空数组,只会给他赋值一个空数组,当添加add()第一个元素的时候
会创建一个10容量的数组长度,同时完成更新引用 完成扩容

public  ArrayList(){

    
}
```

## ArrayList和LinkedList的区别?

1. 内存分析
2. 插入速度/删除速度?
3. 随机访问的效率
4. 底层数据结构

# HashMap的底层实现结构

> 什么是hash冲突???
>
> 当我们通过put(key,val)向hashmap中添加元素,需要通过散列函数确定元素究竟放置在数组中的哪一个位置

JDK1.8之前

:::info
HashMap 使用的是数组加链表. HashMap 通过key的hashcode 得到处理的hash值, 然后通过(数组长度-1) & hash 判断当前元素存放的位置.

如果当前位置存在元素的话,就判断该元素与要存入的元素hash值以及key是否相同,如果相同,直接覆盖,不相同就通过拉链法解决冲突.

:::

# 异常:

异常分为:

Throw : Exception, Error

Exception 又分为: 编译时异常,非编译时异常(也就是运行时异常)

见下图:

[processon](https://www.processon.com/embed/6655aff7e5710259963d66df?cid=6655aff7e5710259963d66e2)

## Throwable类常用方法有哪些??

* `String getMessage()`:返回异常时发生得简要描述
* `String toString()`:返回异常发生的详细信息
* `String getLocalizedMessage()`:
* `void printStackTrace()`:在控制台上打印`Throwable`对象封装异常信息

# 反射

1. 创建对象

```java
Class.forName("类的全路径名");
```

2. 获取方法

```java
clazz.getDeclaredMethods();
clazz.getMethod();

```

3. 获取构造器

```java
clazz.getConstructor();
```

# 创建对象的几种方式?

```java
1. 使用new关键字
2. Class对象的newInstance()方法
构造函数对象的newInstance()方法
对象反序列化
Object对象的clone()方法

在使用反射的时候: 先利用反射获取类信息也就是clazz,然后通过获得构造器,
创建实例newInstance() 得到自己想要创建的对象
也就是 : clazz.getConstructor().newInstance(); //Obejct o = 
如果想要调用方法先要获取方法对象:  clazz.getMethod(String name,参数类型.class)
然后通过方法对象去调用invoke()方法,参数中放执行的对象,以及参数!
```

# 深拷贝与浅拷贝

浅拷贝 他会创建一个新的对象

> 如果属性是基本类型,拷贝的就是基本类型的值,如果属性是引用类型,拷贝的就是内存地址, 因此如果有其中一个对象改变了地址,就会影响到另一个对象.

深拷贝会拷贝所有的属性,并拷贝属性指向动态分配的内存. 序列话属于深拷贝

# String底层

## String为什么是final

1. 为了实现字符串池
2. 为了线程安全
3. 为了实现String可以创建HashCode不可变性

## 为什么equals要重写hashCode方法?

因为仅仅通过两个对象的属性值相等,就判断两个对象相等的话,而不去判断hashCode(),这样再插入哈希表的时候,两个相同对象却拥有不同的hashCode,集合错误的认为他们是不同的对象,从而在删除或者查找的时候效率降低.会导致bug,所以为了维护一致性原则,必须要重写hashCode方法

# 包装类

数据类型不再多说:

包装类的缓冲池技术: `Byte` `Short``Integer` `Long` (-128,127) `Character` 是(0,127)引用常量池对象

NPE问题

# 泛型:

编译器可以对泛型参数进行校验.通过泛型参数可以指定传入得对象类型\
**泛型类,泛型方法,泛型接口**

**泛型类:**

```java
public class Generics<T>{

    //逻辑业务

    
}
```

**泛型接口:**

```java
public interface Genertor<T>{

    
}
```

**泛型方法:**

```java
import java.util.List;

public class GenericMethodExample {

    // 定义一个泛型方法 findFirstIndex，它接受一个List集合和一个Predicate（条件判断函数）
    public static <T> int findFirstIndex(List<T> list, Predicate<T> predicate) {
        for (int i = 0; i < list.size(); i++) {
            if (predicate.test(list.get(i))) {
                return i;
            }
        }
        return -1; // 如果没有找到满足条件的元素，返回-1
    }

    public static void main(String[] args) {
        // 使用泛型方法的例子
        List<Integer> numbers = List.of(1, 2, 3, 4, 5);
        
        // 定义一个条件判断函数，用于查找偶数
        Predicate<Integer> isEven = num -> num % 2 == 0;
        
        // 调用泛型方法，传入numbers列表和isEven条件
        int index = findFirstIndex(numbers, isEven);
        
        System.out.println("第一个偶数的索引是: " + index); // 输出：1
    }
}
```

# 什么是序列化,什么是反序列化??

\*\*序列化:\*\*将数据结构/对象转成二进制字节流的过程

**反序列化:** 将在序列化所生成的二进制字节流转换成数据结构或者对象的过程.

目的是:通过网络传输对象或者说是将对象存储到文件系统,数据库,内存中.

如果有些字段不想进行序列化怎么办??

使用`transient`关键字进行修饰

`transient`只能修饰变量,不会修饰类和方法

`transient`修饰的变量,在反序列化之后的变量值将会被设置成默认值,比如说`int`

反序列化之后的值就是0

`static`变量不属于任何对象,所以有没有`transient`都不会被序列化.

# IO流简介

# 集合:

### 数组和List的区别?

存储方式,数据类型,方法功能,性能特点.

二者互相转换->数组转集合 Arrays.toList(array)     .

集合转数组 -> toArray

### 在列表循环中删除元素的方法有哪几种?

使用Stream.filter(里面是Predicate接口).toList()

还有使用removeif方法

HashMap

1. **<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">目标</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">：要一个O(1)的Map。</font>
2. **<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">核心</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">：用</font>**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">哈希函数</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">直接定位数组下标。</font>
3. **<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">冲突</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">：用</font>**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">链地址法</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">（链表）解决哈希冲突。</font>
4. **<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">退化</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">：数据多了，链表变长，性能退化到O(N)。</font>
5. **<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">扩容</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">：当元素超过</font>**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">容量×加载因子</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">时，</font>**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">扩容</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">并</font>**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">Rehash</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">，打散链表，恢复性能。</font>
6. **<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">优化</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">：为了防止极端长链表，Java 8引入</font>**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">红黑树</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">，在链表长度>8时，将查找性能从O(N)优化到O(logN)。</font>
7. **<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">并发</font>**<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">：以上所有设计都是单线程的。多线程下，</font><code>**<font style="color:rgb(235, 87, 87);background-color:rgb(236, 236, 236);">HashMap</font>**</code><font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);"> 会有数据丢失和死循环问题。必须使用 </font><code>**<font style="color:rgb(235, 87, 87);background-color:rgb(236, 236, 236);">ConcurrentHashMap</font>**</code>

**<font style="color:rgb(235, 87, 87);background-color:rgb(236, 236, 236);"></font>**

<font style="background-color:rgb(236, 236, 236);">这个地方使用@EqualAndHashCode(callSuper=true) </font>

<font style="color:rgba(0, 0, 0, 0.86);">Lombok </font>**<font style="color:rgba(0, 0, 0, 0.86);">只会考虑当前类中声明的非 </font>**<code>**<font style="color:rgba(0, 0, 0, 0.86);background-color:rgba(0, 0, 0, 0.06);">static</font>**</code>**<font style="color:rgba(0, 0, 0, 0.86);"> 和非 </font>**<code>**<font style="color:rgba(0, 0, 0, 0.86);background-color:rgba(0, 0, 0, 0.06);">transient</font>**</code>**<font style="color:rgba(0, 0, 0, 0.86);"> 字段</font>**<font style="color:rgba(0, 0, 0, 0.86);">，来生成 </font><code><font style="color:rgba(0, 0, 0, 0.86);background-color:rgba(0, 0, 0, 0.06);">equals()</font></code><font style="color:rgba(0, 0, 0, 0.86);"> 和 </font><code><font style="color:rgba(0, 0, 0, 0.86);background-color:rgba(0, 0, 0, 0.06);">hashCode()</font></code><font style="color:rgba(0, 0, 0, 0.86);"> 方法</font>

<font style="color:rgba(0, 0, 0, 0.86);">主要在继承关系中使用;</font>

<font style="color:rgba(0, 0, 0, 0.86);"></font>

<font style="color:rgba(0, 0, 0, 0.86);"></font>

<font style="color:rgba(0, 0, 0, 0.86);"></font>

<font style="color:rgba(0, 0, 0, 0.86);"></font>

<font style="color:rgba(0, 0, 0, 0.86);">栈: 存储方法调用/局部变量 </font><font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">它的生命周期与线程和方法绑定，特点是存取速度快但空间小，生命周期固定</font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">堆: 用于存储所有的对象实例和数据, 它的生命周期由垃圾回收器管理，特点是空间大但存取速度相对慢。</font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">方法区/元空间 存储类信息,常量 静态变量等</font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);"></font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);"></font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);"></font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);"></font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);"></font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);"></font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);"></font>

<font style="color:rgb(78, 78, 78);background-color:rgb(244, 246, 248);">引用</font>

```java
public static void changeName(Person p) {
p.setName("New Name"); // 修改对象内容 → 外部可见
}

public static void main(String[] args) {
    Person person = new Person("Old");
    changeName(person);
    System.out.println(person.getName()); // "New Name"
}

```

```java
public static void reassign(Person p) {
    p = new Person("Another"); // p 现在指向新对象
}

public static void main(String[] args) {
    Person person = new Person("Old");
    reassign(person);
    System.out.println(person.getName()); // 仍然是 "Old"！
}

```

* <font style="color:rgba(38, 36, 76, 0.88);">你能通过引用</font>**<font style="color:rgba(38, 36, 76, 0.88);">修改对象的内容</font>**
* <font style="color:rgba(38, 36, 76, 0.88);">但不能通过参数</font>**<font style="color:rgba(38, 36, 76, 0.88);">改变外部引用本身</font>**


> 更新: 2025-12-08 15:40:26  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/gem9b1fk0uifoi52>