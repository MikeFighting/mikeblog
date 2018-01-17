---
title: 理解Foundation框架
date: 2017-09-04 10:13:00
tags: WWDC
---

本文源于对WWDC：UnderStanding Foundation的总结。
 
## 什么是Foundation?

* Foundation提供了构建基础类的框架
  * 所有应用都是用的基础类型
  * 它们供软件的更高层来组合使用

## Dictionry

Dictionary中提供了`objectEnumerator`和`keyEnumerator`两个方法，可以直接取到`key`或者`value`的Enumerator，然后就直接可以用while循环了。

```objc   
NSEnumerator *e = [dictionary keyEnumerator];

while(id key = [e nextObject]){

 id value = [e objectForKey:key];
 ....
}
```           

### Fast Enumeration 

如果想获取某个Dictionary中的key，那么直接用Fast Enumeration就可以：

```objc   
    NSDictionary *someDic = @{@"key":@"value"};
    for (id key in someDic) {
      
        NSLog(@"key:%@",key);   
    }
```      

如果想要获取Key及其对应的Value，那么直接是用Block就可以：

```objc
    [someDic enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
        
    }];
```       

### NSArray排序

对一个NSArray进行排序，有以下几种方法：

1. C function
2. Objective-C method
3. NSSortDescriptor
4. Blocks

使用Bock遍历数组的方法：

```objc
NSMutableArray *names = [NSMutableArray arrayWithObjects:@"11",@"22", nil];
    [names sortUsingComparator:^NSComparisonResult(id  _Nonnull obj1, id  _Nonnull obj2) {
        NSComparisonResult result;
        NSUInteger lLen = [obj1 length], rLen = [obj2 length];
        
        if (lLen < rLen) {
            result = NSOrderedAscending;
        }else if (lLen > rLen){   
            result = NSOrderedDescending;
        }else{
            result = NSOrderedSame;
        }        
        return result;
    }];
```         

### Collection的过滤

**遍历一个Collection的同时再改变它，会引发异常。**
Collection的过滤步骤：

1. 将要筛选的Collection改为可变类型的
2. 筛选出需要移除的项
3. 调用可变类型的响应方法进行移除

```objc
    NSMutableArray *files = [NSMutableArray arrayWithObjects:@"file0",@"file1", nil]; // array of NSString objcects;
    NSIndexSet *toRemove = [files indexesOfObjectsPassingTest:^BOOL(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        
        if ([obj hasPrefix:@"."]) {return YES;}
        return NO;
        
    }];
    [files removeObjectsAtIndexes:toRemove];
```         

### Collection更多的特性

* 查找
* 对每个元素都调用某个方法
* `NSArray`：切开和串联
* NSSet：交集，合并和子集

## Strings

String其实是一个Unicode字符集的数组，你可以将它看做非透明的容器。对它的常用方法有：

* 比较
* 查找
* 转换编码

### 字符串比较的方法：

```objc
- (NSComparisonResult)compare:(NSString *)string; // 1
- (NSComparisonResult)localizedStandardCompare:(NSString *)string NS_AVAILABLE(10_6, 4_0); // 2
- (NSComparisonResult)localizedCompare:(NSString *)string; // 3
- (NSComparisonResult)localizedCaseInsensitiveCompare:(NSString *)string; // 4
- (NSComparisonResult)compare:(NSString *)string options: (NSStringCompareOptions)mask range:(NSRange)rangeOfReceiverToCompare locale:(nullable id)locale; // 5
```           

第二个方法和第三个方法是对那些做了本地化的字符串进行比对。第四个可以对字符串的一部分进行比较，并且可以指定是否是大小写敏感等。如果数组中有字符串需要排序，那么我们可以用到上文中提到的，让数组中的元素分别调用其自身的方法。

```objc
    NSArray *strings = [NSArray arrayWithObjects:@"Larry",@"Curly",@"Moe", nil];
    NSArray *sortedArray = [strings sortedArrayUsingSelector:@selector(localizedCompare:)];
```    

### 字符串查找

字符串查找的方法如下：

```objc
- (NSRange)rangeOfString:(NSString *)searchString;
- (NSRange)rangeOfString:(NSString *)searchString options:(NSStringCompareOptions)mask range:(NSRange)rangeOfReceiverToSearch locale:(nullable NSLocale *)locale NS_AVAILABLE(10_5, 2_0);
```        

*注：如果有特殊字符，比如`Ó`，那么它在数组中是`O`，`´`两个分开存储，是占两个存储单位的，所以它自身rang的length是2。

字符串的查找中还支持正则表达式：

```objc
    NSString *str = @"Going going gone!";
    NSRange found = [str rangeOfString:@"go(\\w*)"
                               options:NSRegularExpressionSearch
                                 range:NSMakeRange(0, str.length)];
```       

这样我们就可以得到found的值：found.location = 6, found.length = 5;

### 字符串编码

字符串和Data之间编码的相互转换：

```objc
    NSData *someData = [NSData dataWithContentsOfFile:@""];
    NSString *inString = [[NSString alloc]initWithData:someData encoding:NSUTF8StringEncoding];
    NSString *outString = @"For Windows";
    NSData *converted = [outString dataUsingEncoding:NSUTF16StringEncoding];
```   

如果要将某个和文件系统相独立的字符串表示成文件系统调用时所指定的字符串表示，可以使用：

```objc
const char *fileName = [outString fileSystemRepresentation];
```   

将它表示成一个C类型的字符串，这个字符串在outString销毁的时候也跟着自动销毁。

更多的特性：

* 打印格式
* 遍历逐个子字符串遍历，逐行遍历，逐段遍历
* 替换某个子字符串
* 路径补全

### NSDateFormatter：

如果不想将一个`NSDate`转换成字符串的时候出现时间，那么可以将`timeStyle`设置为：`NSDateFormatterNoStyle`。

```objc
    NSDateFormatter *fmt = [[NSDateFormatter alloc]init];
    [fmt setTimeStyle:NSDateFormatterNoStyle];
    [fmt setDateStyle:NSDateFormatterLongStyle];
```   

### Dates和Formatter小结

* 将NSDate和NSCalendar混合使用来计算时间
* 当展现日期和数字的时候使用foramtter

## 数据持久化

### 将数据以Plist的形式存储

将数据转换成Plist：

```objc
    NSDictionary *colors = [NSDictionary dictionaryWithObjectsAndKeys:@"Verde",@"Green",@"Rojo",@"Red",@"Amarillo",@"Yellow", nil];
    NSError *error = nil;
    NSData *plist = [NSPropertyListSerialization dataWithPropertyList:colors format:NSPropertyListXMLFormat_v1_0 options:0 error:&error];
    if (!plist) {
        NSLog(@"本地化失败");
    }
    [plist writeToFile:@"filePath" atomically:YES];
```    

将Plist转化成NSData：

```objc
    NSData *readData = [NSData dataWithContentsOfURL:urlOfFile];
    NSDictionary *newColors = [NSPropertyListSerialization propertyListWithData:readData options:0 format:nil error:&error];
```  

### NSFileManager

NSFileManager支持文件的复制，移动，链接，删除等。

```objc
    NSFileManager *mgr = [[NSFileManager alloc]init];
    BOOL res;
    res = [mgr copyItemAtURL:src toURL:des error:&error];
    res = [mgr moveItemAtURL:src toURL:des error:&error];
    res = [mgr linkItemAtURL:src toURL:des error:&error];
    res = [mgr removeItemAtURL:src error:&error];
```    

`linkItemAtURL`就是创建一个硬链接，硬链接就是给一个已经存在的文件重新创建另外一个名字，如果原来文件被删除了，那么硬链接的名字就会失效了。

NSFileManager还可以遍历某个目录下所有的内容：

```objc
    NSArray *stuff = [mgr contentsOfDirectoryAtURL:dirURL includingPropertiesForKeys:[NSArray array] options:0 error:&error];
```     




