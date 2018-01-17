---
title: Swift中的Dictionary，Set，Range
date: 2017-07-28 09:30:32
tags: Notes
---

# Dictionary，Set，Range

## Dictionay的Key

Swift中`Dictionary`的`Key`都应该是可哈希的，也就是遵守`Hashable`协议，又因为`Hashable`是`Equatable`的扩展，所以又必须遵守`Equatable`协议

在Swift中比较两个Type对象(**引用相等用===**)是否相等，需要遵守`Equatable`协议，并覆盖`==`方法，如：

		   struct Person {
		   var name: String
		   var zipCode: Int
		   var birthday: Date
		}
			extension Person: Equatable {
			static func ==(lhs: Person, rhs: Person) -> Bool {
			return lhs.name == rhs.name
			&& lhs.zipCode == rhs.zipCode
			&& lhs.birthday == rhs.birthday
			}
		}

也就是必须明确一下两点：

  1. 相等的实例必须有相同的哈希值。
  2. 具有相同哈希值的两个对象不一定相等。

由于每次往`Dictionary`中插入数据，都会判断哈希方法，以确定是否覆盖之前`Key`的`Value`。所以这个哈希方法应该具备以下两个特点:

一、应该的执行效率就会影响到对`Dictionary`的操作效率。
二、哈希方法应该尽可能得减少碰撞。
 
 如上面的`Person`所示，如果要做为`Key`就还需要增加下面的代码：
 
 
    extension Person: Hashable {
      var hashValue:Int {
      returen name.hasValue^zipCode.hashValue^birthday.hashValue
      }
    }
注：

1. 标注库中的基础类型如: String,integer,float,boole，以及没有关联值的Enum。
2. 在`Dictionary`中，如果使用可变的引用类型来作为`Key`并且将其改变，这个改变同时又改变了哈希值，那么这个`Key`对应的`Value`就丢失了。但是如果值类型的实例作为了`Key`，因为它执行了一次`Copy`，所以就不用担心被改变了。这也是值类型的一个优势所在。
   
## Set 

`Set`是Swift标准库中唯一遵循`SetAlgebra`协议的类型（Foundation中的:IndexSet和CharacterSet也遵循该协议），`SetAlgebra`中提供了`subtracting`减法，`intersection`取交集，`formUnion`取并集，`isDisjoint`是否有交集等代数方法。`IndexSet`中存储都是正整数，它和Set<Int>相比存储效率更高，因为`IndexSet`是存储范围的，比如10000中的前500个数只需要存储俩个数值：起点和终点。

在过滤某个数组中的相同元素时候我们往往可以将其放到`Set`中，这样就可以将相同元素过滤，但是会遇到一个问题，得打的`Set`和原来的不是顺序不一样，这时，我们可以给`Sequence`添加`Extension`来解决。

    extension Sequence where Iterator.Element: Hashable {
    func unique() -> [Iterator.Element] {
        var seen:Set<Iterator.Element> = []
        return filter{
            if seen.contains($0){
              return false
            }else{
                seen.insert($0)
                return true
            }
        }
    }
	}
	let result = [1,2,3,12,1,3,4,5,6,4,6].unique()
通过上面的方法就可以将元素按照原来的顺序排列下来。

## Range

Swift中管更capable的集合叫countable，countable的边界包含`integer`和指针。但是不包含浮点型，因为`Stride`对	`integer`有限制。如果要遍历浮点型数据，那么需要使用`stride(from:to:by)`，以及`stride(from:through:by)`来新建一个`Sequence`。
将封闭的`Ragne`转化成半封闭的`Range`是不可能的，因为在浮点数的情况下不能确定比最大值小的值是多少。	除非这个`Sequence`是`Strideable`的。如果一个函数接受一个Range座位返回值的话，不可以用`...`来创建。

