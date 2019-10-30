---
title: RunTime应用实例：MustOverride
date: 2017-06-05 11:41:00
tags: Objective-C
---

## 常用做法

在IOS开发中，我们的基类往往会写一些空方法，然后让子类去实现，基类控制主要流程(这其实就是模板方法模式)，这时我们往往这样写：

```objc
 - (void)mustBeOverriddenMethod {
    [NSException raise:@"Method did not be overridden" format:@"you must override this method in the subclass"];
    }
```

  这样该方法如果直接被父类调用就会报异常，并且提示一定要被子类所覆盖。但是该方法存在如下弊端：
  
  1. 该方法一定要被调用才可以报异常，如果子类没有调用该方法，也没有覆盖该方法，父类在某些特定的情况下才调用该方法，那么就会出错。
  2. 不可以在该方法内部做一个基本的实现，然后被子类继承并且调用`[super mustBeOverriddenMethod]`
  3. 如果项目中存在一个子类，但是暂时没有用到，并且其没有覆写这个方法，那么没有提示。以后其他人用这个类，很可能就会出错。
  
## 优雅的做法及疑问

   以上这些问题都可以通过[MustOverride](https://github.com/nicklockwood/MustOverride)框架来实现。
   先来看下其用法，然后我们逐步分析其实现方式。
   只要在父类需要被实现的方法内容添加一个宏：`SUBCLASS_MUST_OVERRIDE`即可：

```objc
 - (void)someMethod {
     SUBCLASS_MUST_OVERRIDE;
     }  
```

这样就可以了，并且更加神奇的是：

1. 没有类调用该方法也可以报异常。
2. 就算子类没有被用到也会报异常。
3. 父类中可以做简单的实现，子类可以调用`super`来扩展该实现。

这时你可能产生如下疑问:

   1. 这个类没有用到为啥可以报异常？
   2. 它是怎样找到这个类的被标记了`SUBCLASS_MUST_OVERRIDE`的方法的？

## 对问题的剖析

   一切都要从这个宏说起，进入宏的定义可以发现：

```objc
     #define SUBCLASS_MUST_OVERRIDE __attribute__((used, section("__DATA,MustOverride" \
    ))) static const char *__must_override_entry__ = __func__
```

  是不是感觉有些长？我们可以将该宏拆分：

```objc
      #define SUBCLASS_MUST_OVERRIDE static const char *__must_override_entry__ = __func__
 __attribute__((used, section("__DATA, MustOverride" )))
```

 首先定义了一个静态常量指针`__must_override_entry__`，这个指针指向`__func__`，也就是该宏所在方法的方法名。然后利用`__attribute__`(编译器指令，可以在声明时做一些错误检查，或者一些优化)，将其放入指定的`section`中（关于section的定义会在后续章节中加以说明）,我们可以在`loader.h`中看到section是这样一个结构体：

```objc
    struct section { /* for 32-bit architectures */
		char		sectname[16];	/* name of this section */
		char		segname[16];	/* segment this section goes in */
		uint32_t	addr;		/* memory address of this section */
		uint32_t	size;		/* size in bytes of this section */
		uint32_t	offset;		/* file offset of this section */
		uint32_t	align;		/* section alignment (power of 2) */
		uint32_t	reloff;		/* file offset of relocation entries */
		uint32_t	nreloc;		/* number of relocation entries */
		uint32_t	flags;		/* flags (section type and attributes)*/
		uint32_t	reserved1;	/* reserved (for offset or index) */
		uint32_t	reserved2;	/* reserved (for count or sizeof) */
    };
```

  关于[used的用法我们要到ARM的指令说明中查询](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0491c/BABCJJID.html)
  
![ARM中关于used的说明](http://upload-images.jianshu.io/upload_images/1513759-4a3f19f66fd72bd4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  从上面可以看出，used的意思是告诉编译器该**静态变量**要在该对象文件中被保留（尽管该变量是没有被引用的）。被标注的静态变量将会按照声明的顺序，放到指定的一个section中。使用`__attribute__((section("name")))`可以指明该section.
  那么放到section中的静态变量是怎样被使用的呢？
  我们可以看到在load方法中，其调用了`CheckOverrides`函数，也就是在该类加载到Runtime中的时候就被调用，不论其是否被使用。

```objc
    Dl_info info;
    dladdr((const void *)&CheckOverrides, &info);

    const MustOverrideValue mach_header = (MustOverrideValue)info.dli_fbase;
    const MustOverrideSection *section = GetSectByNameFromHeader((void *)mach_header, "__DATA", "MustOverride");
    if (section == NULL) return;

    NSMutableArray *failures = [NSMutableArray array];
    for (MustOverrideValue addr = section->offset; addr < section->offset + section->size; addr += sizeof(const char **))
    {
        NSString *entry = @(*(const char **)(mach_header + addr));
        NSArray *parts = [[entry substringWithRange:NSMakeRange(2, entry.length - 3)] componentsSeparatedByString:@" "];
        NSString *className = parts[0];
        NSRange categoryRange = [className rangeOfString:@"("];
        if (categoryRange.length)
        {
            className = [className substringToIndex:categoryRange.location];
        }

        BOOL isClassMethod = [entry characterAtIndex:0] == '+';
        Class cls = NSClassFromString(className);
        SEL selector = NSSelectorFromString(parts[1]);

        for (Class subclass in SubclassesOfClass(cls))
        {
            if (!ClassOverridesMethod(isClassMethod ? object_getClass(subclass) : subclass, selector))
            {
                [failures addObject:[NSString stringWithFormat:@"%@ does not implement method %c%@ required by %@",
                                     subclass, isClassMethod ? '+' : '-', parts[1], className]];
            }
        }
    }
```

   从中可以看到其从`Dl_info`中获取了section，
   什么是`Dl_info`，`dladdr`？[我们要从Linux指令集中去查找](http://man7.org/linux/man-pages/man3/dladdr.3.html)，
   ![Linux中关于dladdr的说明](http://upload-images.jianshu.io/upload_images/1513759-635cfd5c1231b6d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  从其中的解释可以看出来，`dladdr`可以用来确定`addr`指明的地址是否存在于公用的对象中，这些对象是被调用程序所加载的。如果存在那么`dladdr`会返回公用对象及重叠`addr`的表示。该信息被封装到了`Dl_info`结构体中。取出`Dl_info`结构体中的`dli_fbase`,然后调用`getsectbynamefromheader_64`，就可以获取之前存储数据的`section`。然后遍历该`section`以找到所有被标识的方法。接下来利用`RunTime`找到所有的子类：

```objc
      static NSArray *SubclassesOfClass(Class baseClass)
{
    static Class *classes;
	    static unsigned int classCount;
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	      classes = objc_copyClassList(&classCount); // 获取项目中所有用到的类
	    });
    NSMutableArray *subclasses = [NSMutableArray array];
    for (unsigned int i = 0; i < classCount; i++)
    {
        Class cls = classes[i];
        Class superclass = cls;
        while (superclass)
        {
            if (superclass == baseClass)
            {
                [subclasses addObject:cls];
                break;
            }
            superclass = class_getSuperclass(superclass);
        }
    }
    return subclasses;
     }
```

  判断某个类是否覆盖了方法：

```objc
	  static BOOL ClassOverridesMethod(Class cls, SEL selector)
	{
	    unsigned int numberOfMethods;
	    Method *methods = class_copyMethodList(cls, &numberOfMethods);
	    for (unsigned int i = 0; i < numberOfMethods; i++)
	    {
	        if (method_getName(methods[i]) == selector)
	        {
	            free(methods);
	            return YES;
	        }
	    }
	    free(methods);
	    return NO;
	}
```

如果没有覆盖则报异常。
小结：MustOverrid在编译期利用`__attribute__((used,section("__DATA, MustOverride")))`来将方法名放到`section`中，然后在文件加载到runtime的时候找到这个`section`，进而找到对应地方法，找到所有的子类，利用`runtime`判断其是否覆盖了父类的方法。

## 附：
  
  关于load方法的几点说明：
  在类或者分类被加载到Runtime的时候，会触发`load`方法；并且只会在第一次被加载的时候被调用，所以只会调用一次。
  load方法的调用顺序：
  1. 父类先调用`+load`方法，然后子类再调用。
  2. 分类调用`+load`方法要晚于原类。
  
### 延伸阅读

http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0474e/BABHIIEF.html
http://tech.meituan.com/DiveIntoCategory.html
http://man7.org/linux/man-pages/man3/dladdr.3.html
https://www.bignerdranch.com/blog/inside-the-bracket-part-5-runtime-api/
http://nshipster.com/__attribute__/
