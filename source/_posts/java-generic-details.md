---
title: Java泛型
date: 2017-10-17 21:18:59
tags: Notes
---
## 为什么需要泛型？

在Java1.5版本之前是没有泛型的。这时如果我们要实现一个Array类，它里面可以存储任何的对象，我们该怎样做呢？显然，我们可以通过多态来实现：

```java
class Array {
void add(Object e);
// ....
Object get(int i);
}
```

因为Java中的对象都继承自Object，所以就可以往这个Array中添加任何的对象，并且取出任何的对象了。但是这样做会有很大的风险：

* 我们取出的对象都是Object类型的，所以如果要使用这个对象就必须做一次强制转换，因为我们使用Collection框架的频率是很高的，所以这种转换就显得比较麻烦
* 不安全，因为是Object，所以我们可以往里面放任何的对象，比如Animal对象，Plant对象，Plane对象，这样如果我们在调用Array中对象的方法时就可能Crash

第二点对于Java这种追求安全性的语言来说，显然是不可以接受的，所以就出现了泛型。

## 泛型的基本用法

### 泛型类

我们可以定义一个泛型的类，具体的方法如下：

```java
class Pair<T> {
   T first;
   T second;
   // ....
}
```

这样在我们创建对象的时候就可以使用`Pair<Integer> pair = new Pair<Integer>()`，这样我们就可以明确的指出这里的`Pair`中存放的就是`Integer`类型的对象，其它对象如果要放到Pair中编译器就会报错，这样就在很大程度上增加了安全性。并且在取出的时候也不必再进行一次强制转换了。
如果某个类中有多个泛型，那么这些泛型用`,`好分割开就好了。`public class Pair<T, U> { . . . }`。*一般我们会用大写字母来表示泛型的元素，在Java框架中`E`用来表示一个元素，`K`和`V`用来表示一个table的key value值。*

同样，我们可以定义一个泛型的方法，比如：

```java
class ArrayAlg {
public static <T> T getMiddle(T... a) {
return a[a.length / 2]; }
}
```

调用的时候不需要进行强制转换

```java
String middle = ArrayAlg.getMiddle("John", "Q.", "Public");
double middle = ArrayAlg.getMiddle(3.14, 1729, 0);
```

在第二个例子中我们传入的参数是原始数据类型，这时编译器会给我们自动打包，在从类中取出的时候编译器会给我们自动的拆包。

## 泛型的界

有时候我们需要对输入的参数做一些限制，比如说要好处两个数值中较小的一个，那么我们就要求进入方法的参数是实现了`Comparable`接口的，或者是某个类的子类，这样我们就可以做如下的限制，来让泛型有界：

```java
public static <T extends Comparable> T min(T[] a) . . .
```

如果某个类型没有实现`Comparable`接口的话，让它作为参数会产生一个编译期的警告，在运行时就会Crash。如果一个泛型需要限制很多个接口的话，那么很多个接口之间用`,`隔开（但是这其中只能有一个是对类的限制，并且如果有对类的限制，那么这个限制一定要写在第一个的位置）。

## 泛型和JVM

在Java虚拟机中是没有任何泛型类的，所有的对象都是普通的Java类。其实编译器会对泛型实行一次擦除的过程。也就是说，当你定义一个泛型的时候，系统会自动给你创建一个原始类型给你。原始类型变量的名字和泛型时候取的名字是一样的，但是泛型类型的参数类型被移除了。这些类型被移除之后，取而代之的是它们的边界类型（如果没有边界，那么它的边界就是Object，这也是为了和Java之前的版本做兼容。这也就是上文提到的，为什么对于原始数据类型有一个装包和拆包的过程）。比如，如果你创建上文中的`Pair<T>`，在编译之后Pair类就成了下面的样子：

```java
public class Pair {
private Object first; private Object second;
public Pair(Object first, Object second) {
this.first = first;
this.second = second; }
public Object getFirst() { return first; }
public Object getSecond() { return second; }
public void setFirst(Object newValue) { first = newValue; }
public void setSecond(Object newValue) { second = newValue; } }
```

这其实和没有泛型时候创建的类是一样的了。

这里有一个特殊情况，如果有两个限制，那么会以第一个限制为准：

```java
public class Interval<T extends Comparable & Serializable> implements Serializable {
private T lower;
private T upper;
...
public Interval(T first, T second) {
if (first.compareTo(second) <= 0) { lower = first; upper = second; }
else { lower = second; upper = first; } }
}
```

这时`Interval`的基本类型就是：

```java
public class Interval implements Serializable {
private Comparable lower;
private Comparable upper;
...
public Interval(Comparable first, Comparable second) { . . . }
}
```

这种情况下，如果第一种类型要转化为第二种类型的时候编译器会自动的加上强制转换。

例如：

```java
Pair<Employee> buddies = . . .;
Employee buddy = buddies.getFirst();
```

因为在编译期进行了类型擦除，所以在调用`buddies.getFirst()`的时候返回的是一个Object，所以编译器就自动添加了一层强制转换。

## 泛型方法的翻译

上文提到过泛型会被编译器在编译的阶段进行擦除，并且将边界替换为泛型的类型。这种泛型的擦除同时也带来了复杂性，比如下面的例子：

```java
class DateInterval extends Pair<LocalDate> {
public void setSecond(LocalDate second) {
if (second.compareTo(getFirst()) >= 0) super.setSecond(second);
}
//...
}
```

因为我们要保证`Pair`中的第二个元素要始终不小于第一个元素，所以我们就继承了`Pair`，并且重写了其`setSecond`方法，这样经过泛型的擦除，最后将会变为：

```java
class DateInterval extends Pair // after erasure 
{
public void setSecond(LocalDate second) { . . . }
//...
}
```

而原来的`Pair`类中的`setSecond`方法是这样子的：

```java
public void setSecond(Object second)
```

很显然，这是两个不同的方法，然而我们不想让它们是不同的方法，我们想让它走我们新写的方法，因为我们在这里面新增加了我们自己的业务逻辑，比如下面的方法：

```java
DateInterval interval = new DateInterval(. . .); Pair<LocalDate> pair = interval; // OK--assignment to superclass
pair.setSecond(aDate);
```

在这里很显然，我们想走我们新的方法，这时编译器其实会在我们的参数为`Object`的方法里面重新调用我们新写的方法：

```java
public void setSecond(Object second) {
    setSecond((Date) second);
    }
```

也就是这种转换让系统调用到了我们新写的方法。这样就可以始终调用到我们自己的方法里面了。

## Java泛型中需要注意的问题

### 不能使用基本数据类型初始化泛型参数

### Runtime类型检查不起作用

### 不能使用参数类型来创建Array

### 可变参数警告

### 不能初始化类型变量

### 泛型类中的静态环境中不允许使用类型变量

### 不能抛出或者捕获泛型类对象

### 注意泛型擦除之后的冲突

## 其它

在遗留代码中往往有非泛型的类，这时，如果用一个泛型类去接，那么往往会产生一个警告，比如：

```java
Dictionary<Integer, Components> labelTable = slider.getLabelTable(); // Warning
```

这时如果你检查了`labelTable`中数据的类型，并且确定了其中的key value为`Integer`和`Components`，那么就可以使用`@SuppressWarnings("unchecked")`来忽略这个警告。

```java
@SuppressWarnings("unchecked")
Dictionary<Integer, Components> labelTable = slider.getLabelTable(); // No warning
```

当然也可以在外层的方法上添加。

Array有特殊的保护，不能对存有父类的Array中添加子类元素，否则在运行时虚拟机会抛出ArrayStoreException的异常。

## 参考内容

https://www.zhihu.com/question/20400700
http://blog.csdn.net/claram/article/details/51943742





