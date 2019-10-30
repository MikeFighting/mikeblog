---
title: 可能被遗漏的Java细节
date: 2017-09-10 16:15:27
tags: Java
---

### Java中的类型转换

#### 兼容的自动转换

在编程中把一种类型的值赋给另一种类型的变量是合法的。如果这两种类型是兼容的，那么Java会自动对其进行类型转换。例如：把int类型的值赋给long类型的变量就可以。然而如果两种类型不兼容，那么就不会发生这种隐式的类型转换。例如，没有将double类型转化为byte类型的定义。这种不兼容的转化只能自己强制进行。
满足以下两个条件时，Java会自动给你进行类型转换：

* 这两种类型是兼容的。
* 目的类型的范围要比源类型的范围大。

数字类型，包括整数和浮点类型都是彼此兼容的，但是，数字类型和字符类型（char）或布尔类型（bollean）是不兼容的。

#### 不兼容的强制转换

虽然自动转换很好，但是它不能满足所有的需求。例如，你需要将一个int类型的变量付给一个byte类型的变量，你就需要使用(target-type)value这种转换了。下面演示将int转换为byte，如果整数超出了byte类型的取值范围，那么它的值将会因为对byte类型的值取%而减少。

```java
int a;
byte b;
// ...
b = (byte)a;
```

当把浮点数转换为整数类型时会发生一种不同的转换：截断。你知道整数没有小数部分，当你把浮点数转化为整数的时候，其小数部分将会被舍弃。例如，如果将值1.23赋给一个整数，那么其结果是1，0.23被舍弃了。如果浮点数值太而不能适合目标整数类型，那么它的值将会因为对目标类型值域取模而减少。例如：

```java

 // Demonstrate casts.
    class Conversion {
      public static void main(String args[]) {
       byte b;
       int i = 257;
       double d = 323.142;
       System.out.println("\nConversion of int to byte.");
       b = (byte) i;
       System.out.println("i and b " + i + " " + b);
       System.out.println("\nConversion of double to int.");
       i = (int) d;
       System.out.println("d and i " + d + " " + i);
       System.out.println("\nConversion of double to byte.");
       b = (byte) d;
       System.out.println("d and b " + d + " " + b);
} } 
```

该程序的输出如下：

```java
Conversion of int to byte.
i and b 257 1
Conversion of double to int.
d and i 323.142 323
Conversion of double to byte.
d and b 323.142 67
```

当值257被强制转换为byte变量时，其结果是257除以256(256是byte类型的变化范围)的余数。当把变量d转换为int类型时，它的小数部分被舍弃了。当吧变量d转换为byte类型时，它的小数部分被舍弃了，而且它的值减少为256的模，即67。

#### 表达式中的类型提升

在表达式中，有时候中间值的精度会比较高，它有可能超过任何一个操作数的范围。例如，考虑下面的表达式：

```java
 byte a = 40;
 byte b = 50;
 byte c = 100;
 int d = a * b / c;
```

其中间结果`a*b`很容易超出它的任何一个byte型操作数的范围。为了处理这种问题，当分析表达式时，Java会自动提升各个byte或者short型的操作数为int型。这意味着表达式`a*b`是使用整数而不是字节型来运算的。这样，尽管变量a和b都被指定为byte型，50*40的中间表达式的结果2000是合法的。

自动类型提升很好，但是有时候会引起令人疑惑的编译错误。例如，这个看起来正确的程序却会引起问题：

```java
byte b = 50;
b = b * 2; // Error! Cannot assign an int to a byte!
```

这里看上去完全合法，但由于当表达式求值的时候，操作数被自动提升为了int型，所以需要强制转换才可以赋值（但是你必须考虑好溢出的情况）。

```java
byte b = 50;
b = (byte)(b * 2);
```

这样就不会有编译器错误了。

#### 类型提升的约定

除了将byte型和short型提升到int型以外，Java还定义了其它的类型提升规则。如果一个操作数是long型，那么整个表达式将被提升到long型；如果一个操作数是float型，整个表达式将被提升到float型；如果一个操作数是double型，计算结构就是double型。我们用个例子来说明：

```java
class Promote {
      public static void main(String args[]) {
       byte b = 42;
       char c = 'a';
       short s = 1024;
       int i = 50000;
       float f = 5.67f;
       double d = .1234;
       double result = (f * b) + (i / c) - (d * s);
       System.out.println((f * b) + " + " + (i / c) + " - " + (d * s));
       System.out.println("result = " + result);
} } 
```

我们来分析：

```java
 double result = (f * b) + (i / c) - (d * s);
```

的类型提升过程，在第一个子表达式`f*b`中，变量b被 升为float类型，该子表达式的结果当然是float类型。 接下来，在子表达式i/c，中，变量c被 升为int类型，该子表达式的结果当然是int类型。然 后，子表达式`d*s`中的变量s被 升为double类型，该子表达式的结果当然也是double类型。 最后，考虑三个中间值，float类型，int类型，和double类型。float类型加int类型的结果是float 类型。然后float类型减去 升为double类型的double类型，该表达式的最后结果是double型。

### Java中的break

Java中的break除了可以用来在switch或者循环中终止某个条件或者循环，还可以作为goto语句的一种形式来使用。Java中没有goto语句，因为goto让程序流程变得非结构化，可能让程序难以理解和维护，并且可以阻止某些编译器优化。但是，有些地方使用goto语句有助于流程控制，并且是合法的。例如，从嵌套很深的循环中退出来，goto语句就很有帮助。因此，Java定义了break语句的一种扩展形式来处理这种情况。通过给某个代码块加上标签，那么**其内部的break语句**就可以在某些情况下跳转到该标签。这个代码块不必非要是循环或者switch，它可以是任意的代码块。标签的指定如下所示：

```java
// Using break as a civilized form of goto.
    class Break {
      public static void main(String args[]) {
       boolean t = true;
       first: {
         second: {
           third: {
            System.out.println("Before the break.");
            if(t) break second; // break out of second block
            System.out.println("This won't execute");
} 
           System.out.println("This won't execute");
         }
         System.out.println("This is after second block.");
       }
} } 
```

改程序的输出如下：

```bash
Before the break.
This is after second block.
```

这个示例中有三个嵌套的代码块，每一个都有它自己的标签。break语句使得循环往外层跳转，直接跳过了second标签的的代码块，直接执行了first标签的代码块。

### Java中的方法重载

方法重载就是说如果两个方法有相同的方法名字，但是其参数不同（参数个数不同或者参数类型不同），那么Java会根据调用时候不同的参数类型来匹配不同的方法来调用。然而有时候其类型匹配不是很精确：比如：

```java
    // Automatic type conversions apply to overloading.
    class OverloadDemo {
      void test() {
       System.out.println("No parameters");
      } 
      // Overload test for two integer parameters. 
      void test(int a，int b) { 
        System.out.println("a and b: " + a + " " + b);
      }
      // overload test for a double parameter
      void test(double a) {
        System.out.println("Inside test(double) a: " + a);
      }
  } 
   class Overload {
      public static void main(String args[]) {
        OverloadDemo ob = new OverloadDemo();
        int i = 88;
        ob.test(); ob.test(10，20); 
        ob.test(i); // this will invoke test(double)
        ob.test(123.2); // this will invoke test(double)
      }
  }```

输出结果是：

```bash
   No parameters
   a and b: 10 20
   Inside test(double) a: 88
   Inside test(double) a: 123.2
 ```

这里，我们的`test(i)`中i是int型，但是在调用时，我们发现调用`test(double)`类型，这是因为在调用`test(int)`型的时候Java找不到相应的类型，所以就将int扩大为了double，然后就调用了`test(double)`，如果我们这里定义了一个`test(int)`，那么就会调用这个`test(int)`而不会将int扩大为double。

### super关键字

在继承中，super可以用来表示父类，可以调用父类特有的方法。还有一种应用情况是：它可以用来调用父类中被子类所隐藏的属性，比如：

```java
class A {
int i; } 
    // Create a subclass by extending class A.
    class B extends A {
      int i; // this i hides the i in A
      B(int a, int b) {
       super.i = a; // i in A
       i = b; // i in B
} 
      void show() {
       System.out.println("i in superclass: " + super.i);
       System.out.println("i in subclass: " + i);
} } 
    class UseSuper {
      public static void main(String args[]) {
       B subOb = new B(1, 2);
       subOb.show();
      }
} 
```

该程序的输出为：

```bash
i in superclass: 1
i in subclass: 2
```

尽管B中的实例变量i隐藏了A中的i，使用super就可以访问超类中定义的i。其实， super也可以用来调用超类中被子类隐藏的方法。 

### 构造函数

在类的层次结构中，构造函数的调用顺序是从父类到子类。并且父类的构造方法的调用要放在子类构造方法的第一行中。如果子类中没有用到super()，那么每个父类默认的或者无参数的构造函数将执行。例如：

```java
// Create a super class.
    class A {
      A() {
       System.out.println("Inside A's constructor.");
} } 
    // Create a subclass by extending class A.
    class B extends A {
      B() {
       System.out.println("Inside B's constructor.");
} } 
    // Create another subclass by extending B.
    class C extends B {
      C() {
       System.out.println("Inside C's constructor.");
} } 
    class CallingCons {
      public static void main(String args[]) {
C c = new C(); } 
} 
```

该程序的输出如下：

```bash
 Inside A’s constructor
 Inside B’s constructor
 Inside C’s constructor
```

由此可见，构造函数以派生的顺序被调用。





