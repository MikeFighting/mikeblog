---
title: Vapor制作JSON Model
date: 2017-08-13 18:16:34
tags: Server-Swift
---

上篇博客中我们讲到了怎样配置数据库，以及怎样将数据库中的数据读出并且返回给客户端，本文将说明怎样将数据库中的数据用Model来表示，并且怎样对Model进行各种操作。

## 建立Class 

我们建立以个简单的`User`类，如下：

```Swift 
final class User {
 var id: String
    var name: String
    var age: Int
    var gender: String
    var imageUrl: String
    var oatch: String
    
    init(id: String, name: String, age:Int, gender: String, imageUrl: String, oatch: String) 
    {
        
        self.id = id
        self.age = age
        self.name = name
        self.gender = gender
        self.imageUrl = imageUrl
        self.oatch = oatch
    }
}
```  
 
## 遵守NodeRepresentable和NodeRepresentable协议

从之前返回JSON时，JSONS的生成过程`JSON(node:someNode)`，那么Node是什么呢？其实就是一个遵守`NodeRepresentable`的对象，我们让User遵守`NodeRepresentable`协议，兵且实现其`makeNode`方法，这样就可以传入`JSON(node:user)`了：

```Swift
func makeNode(in context: Context?) throws -> Node {
        
        return try Node(node:[
            "id":id,
            "name":name,
            "age":age,
            "gender":gender,
            "imageUrl":imageUrl,
            "oatch":oatch
            ])
    }
```    
  
我们这样不调用`JSON`的方法，只使用我们的Model就可以创建出来JSON对象呢？我们只需要遵守`JSONRepresentable`对象即可，然后添加其需要实现的协议方法。

```Swift         
    func makeJSON() throws -> JSON {
        return try JSON(node: self)
    }
```   

然后我们就可以使用`user.makeJSON()`来创建JSON对象了。

