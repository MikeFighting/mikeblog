---
title: Vapor中的数据校验
date: 2017-08-10 07:10:13
tags: Server-Swift
---

   ![Validation](http://upload-images.jianshu.io/upload_images/1513759-267da7a2929f0562.jpg)
服务端在有数据请求时需要对数据进行校验然后返回响应的校验结果，比如要求必须输入邮箱，必须输入电话等，Validation工具给我们提供了非常方便的常用操作，接下来就对其使用过程做以总结（*本文用到的是Validation 1.2.0版本*）。

## Vapor中添加Validation依赖包

在`Package.swift`文件中添加如下的依赖，比如：


    .Package(url: "https://github.com/vapor/vapor.git", majorVersion: 2),
    .Package(url: "https://github.com/vapor/validation-provider.git", majorVersion: 1)


>之后要执行`vapor clean`或者`rm -rf .build Package.pins`，然后执行`vapor update`或者`swift package update`，`vapor xcode`，这样才可以安装好依赖。

## 校验实例

### Alphanumeric校验

 接下来我们来做一个简单的校验，校验输入的字符串是否是a-z或者0-9在请求中加入如下代码：
 
 ```Swift   
	get("alpha") { request in
	            guard let input = request.data["input"]?.string else {
	                throw Abort.badRequest
	            }
	            let validInput = try input.tested(by: OnlyAlphanumeric())
	            return "validated:\(validInput)"
	}
```    
     
我们运行程序，然后在PostMan中输入http://localhost:8080/alpha?input=example@github.com，这时会得到下面的返回值：
```Swift    
	{"identifier":"Validation.ValidatorError.failure","reason":"Internal Server Error","debugReason":"OnlyAlphanumeric failed validation: example@github.com is not alphanumeric","error":true}
```      
这也就说明了，我们传输的问本内容不符合`alphanumeric`。
然后我们将URL改为`http://localhost:8080/alpha?input=example123`，然后就会看到我们的返回值


    validated:example
    

### 邮箱校验

我们可以利用`EmailValidator`来做邮箱的校验，方法同上面一样：
```Swift  
get("email") { request in
            guard let input = request.data["input"]?.string else {
                throw Abort.badRequest
            }
            let validaInput = try input.tested(by: EmailValidator())
            return "validated:\(validaInput)"
    }     
```   
然后我们输入URL：http://localhost:8080/email?input=wallaceicdi@outlook.com，然后就会的到：


    Validated: wallaceicdi@outlook.com


### 其余自带校验工具

校验类 | 功能  | 用法
------|------ |-----
Unique | 输入内容是否唯一 | someCharacter.tested(by: Unique())
Compare | 输入内容的数值比较|int.tested(by:Compare.greaterThan(1))
Contains | 输入的内容是否包含某个 | someArray.tested(by: Contains("1"))
Count    | 输入的内容个数 | someArray.tested(by: Count.max(2))
Equals | 输入的内容是否相同| someConent.tested(by: Equals.init("equal"))
In     |输入内容是否被包含| input.tested(by: In.init(["1","2","3"]))

## 创建自己的校验工具

通过参考工具自带的`Equals.Swift`：

```Swift
/// Validates that matches a given input
public struct Equals<T>: Validator where T: Validatable, T: Equatable {
    /// The value expected to be in sequence
    public let expectation: T

    /// Initialize a validator with the expected value
    public init(_ expectation: T) {
        self.expectation = expectation
    }

    public func validate(_ input: T) throws {
        guard input == expectation else {
            throw error("\(input) does not equal expectation \(expectation)")
        }
    }
}
```   

从这里面我们可以看出，只要遵守`Validator`协议，并且实现其`validate`方法即可。

参考文件：
https://github.com/vapor/validation/blob/master/Tests/ValidationTests/ValidationConvenienceTests.swift

