---
title: 代码整洁之道-继承，多态和测试
date: 2017-09-06 07:50:00
tags: Drafts
---

### 代码整洁的讨论，条件和多态

![Conclusion](http://upload-images.jianshu.io/upload_images/1513759-0ac66a6395d8b9ff.png)

条件和多态是怎样影响测试的？

1. 大多数的if可以被多态所取代

为什么要取代ifs

1. 没有if的代码更容易阅读
2. 没有if的代码更容易测试
3. 多态更容易维护：容易扩展

什么时候使用多态：

1. 如果一个对象基于其不同的状态，而表现不一样
2. 如果你要在不同的地方来检测这种条件

对于基本数据类型，我们使用`>,<,==,!=`来做比较。我们要避免使用`if`。

尽量不适用if
不要返回null（使用null之后不能够Dispatch这个错误）,而是返回一个Null对象，比如一个空的list
不要返回error代码，而是抛出异常

### 尽量少用继承

多态需要使用继承
小心继承的层级太深


### 将条件用多态替代

你有一个条件，该条件是基于某个对象的类型来选择不同的行为。

将这些条件的判断一到一个子类的覆盖方法中。然后将原来的的方变成一个抽象方法。
我们用下面的例子：

```java
double getSpeed() {
  
  switch (_type) {
   
   case ENROPEAN:
        return getBaseSpeed();
        
   case AFRICAN:
        return getBaseSpeed() - getLoadFactor() * _numberOfCoconuts;
    
   case NORWEGIAN_BLUE:
        return (_isNailed) ? 0 : getBaseSpeed(_voltage);
  }
  
  throw new RuntimeException("Should be unreachable");
}
```    

比如我们有一个`1+ 2 * 3`的表达式，如果用树来表示应该是这样的： 

![1 + 2 * 3](http://upload-images.jianshu.io/upload_images/1513759-44f337f613e5c3f2.png)


这时候大多情况下我们会写下面的的代码：

![Implement](http://upload-images.jianshu.io/upload_images/1513759-4efa0dbac30dd19d.png)


那么大多数情况下会出问题，比如：

![Evaluate UML](http://upload-images.jianshu.io/upload_images/1513759-3ee95ff9e5c8dd78.png)

从中我们可以看到叶子节点的左右分支都是null，上文已经提到null是很不好的。比如叶子节点我们需要的只是数值，没有function，没有left和right数值。

那么怎么办呢？我们需要将原来的Node创建两个子类，一个ValueNode，一个OpNode。UML图如下：

![SubClass UML](http://upload-images.jianshu.io/upload_images/1513759-e32bbbf0adf5e116.png)

创建子类之后：

![Node Subclass](http://upload-images.jianshu.io/upload_images/1513759-92adea90f4f9f44d.png)

Java只针对某个类编译。

这样最终我们的UML图是这样的：

![ValueNode](http://upload-images.jianshu.io/upload_images/1513759-d12731bff0df6db3.png)

用多态替代switch case。

![多态UML](http://upload-images.jianshu.io/upload_images/1513759-8e09c9eb66adc448.png)

然后系统的操作类变为：
![Operation Class](http://upload-images.jianshu.io/upload_images/1513759-f3b681a924bba620.png)

然后我们再创建两个`OpNode`的子类：

![OpNodeSubClass](http://upload-images.jianshu.io/upload_images/1513759-d6699abc050608e6.png)

然后我们有了除去Operation字段的UML图：

![NoOperationUML](http://upload-images.jianshu.io/upload_images/1513759-1ebf2e28e6c0c310.png)


## 进一步探讨

定义一个`toString()`方法来打印表达式的前缀，并且在适当的时候加上括号。
添加一些新的数学操作：添加方程，阶乘，算法，三角学。

## 小结      
  
 多态的结局方案通常来说是更好的，因为；
 
 * 新功能的添加不需要改动原来的代码
 * 每个操都被放到一个单独的文件中了，这样易于测试和理解，并且易于扩展

多用多态而不是条件语句：

* **switch**语句就意味着你应该使用多态
* **if**更加很精妙，但是有时候一个**if**仅仅是一个**if**


### 重复条件

```java
class Update {
    execute (){
    if (FLAG_i18n_ENABLED) {
    
       // DO A;
    }else {
       // DO B;
     }
   }
}
```   

但是后来又有了相似的代码：

```java  
class Update {
   render (){
    if (FLAG_i18n_ENABLED) {
    
       // Render A;
    }else {
       // Render B;
     }
   }
}
```    

在测试的时候，我们还需要写两套代码，比如：

```java
void testExecuteDoA {
	FLAG_i18n_ENABLED = true;
	Update u = new Update();
	u.execute();
	assertX();
}
void testExecuteDoB {
  FLAG_i18n_ENABLED = false;
  Update u = new Update();
  u.execute();
  assertX();
}
```   
   
应该怎样做才好呢？我们应该经条件语句用多态来替代：

*你有一种条件，这种条件会基于某个对象的类型类选择不同的行为。*
将判断条件一到一个子类的覆盖方法中。将原来类的方法变成抽象方法。

```java

abstract class Update {
 //...
}

class I18NUpdate extends Update {
   execute() {
    // Do A;
   }
   render() {
   // Render A;
   }
}

class NoNI18NUpdate extends Update {
  execute() {
   // Do B;
  }
  render() {
   // Render A;
  }
}
```   

测试的时候我们就方便多了：

```java
void testExecuteDoA {
   Update u = new MyI18NUpdate();
   u.execute();
   assertX();
}

void testExecuteDoB {
   Update u = new MyNonI18NUpdate();
   u.execute();
   assertX();
}
```    

**if**s去哪里了？

```java
class Consumer {
  Consumer(Update u){...}
}
```   

两种方式：

第一种方式：

对象的堆砌

* 业务逻辑
* 有趣的东西
* 目的是建造逻辑，主要的抽象
* 假如需要很多的合作者

第二种方式：

构造方法的堆砌

* 工厂
* 建造者
* Provier<T>
* 目的是建造对象表
* 创建和提供合作者（依赖注入）

将创建某个具体对象的操作放到工厂方法中去性能会更高，因为我们只需要判断一次就可以对这个对象做一系列的操作。而之前，我们需要尽心一系列的判断。将If else放到工厂中还有一个好处就是便于测试和调试Bug，因为尽管我们有if else，但是这是在创建对象的过程中，具体的业务逻辑我们是放到了子类中去做的，而经常出错的地方往往是业务逻辑，而不是创建对象的过程。举个例子：

```java
class Consumer {
  Consumer(Update u){...}
}

class Factory {
  Consumer build() {
    Update u = FLAG_i18n_ENABLED ? new I18NUpdate()
                                 : new NonI18NUpdate();
    return new Consumer(u);
  }
}
```  

使用依赖注入：

![Injection](http://upload-images.jianshu.io/upload_images/1513759-d44fe1f5cbe6fe8f.png)

这样做的好处是：

1. 所有的判断条件都被放到了一个地方
2. 没有很多的重复代码
3. 将责任分离，将全局状态分离

## 总结优势

1. 公用代码放到了一个地方
2. 很容易独立测试和平行测试
3. 关注子类让它更加明白不同点在哪里

什么时候用继承？

不同的状态产生不同的行为
不同的地方有平行的条件




