---
title: 如何优雅地使用DateFormatter？
date: 2017-08-27 09:48:45
tags: Objective-C
---

## 性能对比

之所以要聊`DateFormatter`是因为某次给项目做性能检测，发现创建DateFormatter太消耗性能，我们来做个对比，新建100000个日期。我们使用两种方式：第一种每次创建日期的时候新建一个NSDateFormatter，第二种共用一个NSDateFormatter，来生成日期：

```swift
public func testWithMultipleInstantiation() ->CFTimeInterval {
    
        var dateStrings:[String] = []
        dateStrings.reserveCapacity(100000)
        let startTime = CACurrentMediaTime()
        for _ in 0..<100000 {
            let df = DateFormatter()
            df.dateStyle = .medium
            df.timeStyle = .full
            dateStrings.append(df.string(from: Date()))
        }
        let endTime = CACurrentMediaTime()
        return endTime - startTime
    }
    
    public func testWithSingleInstance() ->CFTimeInterval {
    
        var dateStrings: [String] = []
        dateStrings.reserveCapacity(100000)
        let startTime = CACurrentMediaTime()
        let df = DateFormatter()
        df.dateStyle = .medium
        df.timeStyle = .full
        for _ in 0..<100000 {
            dateStrings.append(df.string(from: Date()))   
        }   
        let endTime = CACurrentMediaTime()
        return endTime - startTime
}
```

然后我们调用这两个方法：
     
```swift
print("testWithMultipleInstantiation--\(testWithMultipleInstantiation())")
print("testWithSingleInstance--\(testWithSingleInstance())")
```

打印结果是：
            
```swift
testWithMultipleInstantiation--7.83139349098201
testWithSingleInstance--0.742719032976311
```

从中可以明显看到创建`DateFormatter`是很消耗性能的，多次创建DateFormatter比单次创建大约要慢11倍。如果我们要用DateFormatter，那么尽量创建一次，然后多次使用。

然后我们再做进一步的实验：创建一次`DateFormatter`，但是改变这个NSDateFormatter的`dateStyle`和`timeStyle`。

```swift
public func testWithSingleInstanceChangeFormatter() ->CFTimeInterval {
        
        var dateStrings: [String] = []
        dateStrings.reserveCapacity(100000)
        let startTime = CACurrentMediaTime()
        let df = DateFormatter()
        for _ in 0..<100000 {
            
            df.dateStyle = .medium
            df.timeStyle = .full
            df.dateStyle = .full
            df.timeStyle = .medium
            dateStrings.append(df.string(from: Date()))
        }
        
        let endTime = CACurrentMediaTime()
        return endTime - startTime
    }
```

然后调用这个方法：

```swift
print("ChangeFormatter--\(testWithSingleInstanceChangeFormatter())")
```

这时输出的结果是：

```swift
ChangeFormatter--5.77827541399165
```

从中我们可以看到，其对性能的消耗和多次创建DateFormatter相差并不多。最后我们得到这样一个结论：

> 1. 每次使用DateFormatter时都新建是最消耗性能的
> 2. 创建一个DateFormatter然后改变其`dateStyle`和`timeStyle`等和1中的性能消耗差不多
> 3. 为每一种日期类型创建一种DateFormatter并且不改变其`dateStyle`和`timeStyle`等属性是性能最优的

## 解决方案

通过上面的结论，我们发现如果对DateFormatter做成单例，那么就必须保证每个DateFormatter的格式是相同的，因为改变DateFormatter的格式也是很消耗性能的。我们要做多个单例，每种单例是一种formatter，然后分别使用吗？显然太过于麻烦。**我们可以使用缓存策略，将每种格式的DateFormatter缓存一份，下次如果有相同格式的Formatter，直接从缓存中取就可以了，这就避免了多次创建和多次改变格式的问题。**为了解决这个问题，我使用NSCache做了一个DateFormatter的缓存池：[MFDateFormatterPool](https://github.com/MikeFighting/MFDateFormatterPool)，已经上传到了GitHub上，分为OC和Swift两个版本，如有问题可以联系我（Swift版稍后会加上）。

*其它：NSDateFormatter在IOS7之前是非线程安全的，多线程可能引起崩溃，*

### 延伸阅读： 

https://www.raywenderlich.com/31166/25-ios-app-performance-tips-tricks#reuseobjects
http://www.chibicode.org/?p=41
https://stackoverflow.com/questions/18195051/crash-in-nsdateformatter-setdateformat-method



