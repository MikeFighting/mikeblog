---
title: 怎样在UIKit中更好的使用ValueType
date: 2017-08-14 09:06:45
tags: WWDC
---

## ValueType怎样用到UIView中？

**如果两个类没有相同的父类，我们又想对它们写一个相同的逻辑**，怎么办？我们可以让它们遵守同一个协议，该协议中有它们共有的属性，然后就可以复用了。这样我们就依赖Protocol而不是SuperClassl来建立一个多态。
分享代码不必使用继承了：
UIView的组合不是最优的，因为UIView的创建需要很长的时间，并且需要开辟堆空间，需要处理事件。

我们接下来看看怎样利用ValueType将Layout和具体的类进行分开。因为我们可能会用`UIView`也可能用`SKNode`来做其ContentView。
按照之前的情况，我们可能这样写：  

```Swift    
class DecoratingLayoutCell : UITableViewCell {
   var content: UIView
   var decoration: UIView
   // Perform layout...
} 
```       

为了将具体的Cell和其Layout分开，我们创建一个Struct：

```Swift      
struct DecoratingLayout {
   var content: UIView
   var decoration: UIView   
mutating func layout(in rect: CGRect) {
   // Perform layout...
   } 
} 
```          

然后在Cell中我们就可以这样写：

```Swift      
class DreamCell : UITableViewCell {
   ...
   override func layoutSubviews() {
   
   var decoratingLayout = DecoratingLayout(content: content, decoration:decoration)
   decoratingLayout.layout(in:bounds)
} 
```          

同时，我们可以对UIView也用同样的Layout（封装变化）。

```Swift      
class DreamDetailView : UIView {
   ...
   override func layoutSubviews() {
      var decoratingLayout = DecoratingLayout(content: content, decoration:decoration)
   decoratingLayout.layout(in:bounds)
} } 
```      

将Layout分开之后还非常有利于单元测试，比如我们想要测试我们的Layout创建出来的两个View是否符合我们的预期，我们可以这样：
 
```Swift     
 let child1 = UIView()
   let child2 = UIView()
   var layout = DecoratingLayout(content: child1, decoration: child2)
   layout.layout(in: CGRect(x: 0, y: 0, width: 120, height: 40))
} 
XCTAssertEqual(child2.frame, CGRect(x: 0, y: 5, width: 35, height: 30))
XCTAssertEqual(child2.frame, CGRect(x: 35, y: 5, width: 70, height: 30))
```      

因为我们的Layout代码量很小并且和其它代码是隔离的（没有继承，不是Reference Type），那么这样我们就可以很好的做局部分析。

有一天，我们要对SKNode也做同样的操作。所以我们也需要`NodeDecoratingLayout`：

```Swift      
struct ViewDecoratingLayout {
   var content: UIView
   var decoration: UIView   
mutating func layout(in rect: CGRect) {
   // Perform layout...
   } 
} 
struct NodeDecoratingLayout {
   var content: SKNode
   var decoration: SKNode   
mutating func layout(in rect: CGRect) {
   // Perform layout...
   content.frame = ....
   decoration.frame = ...
   } 
} 
```          

我们可以看到这和`VIewDecoratingLayout`是一样的。所以我们要将它封装起来，怎么封装？UIView和SKNode没有共同的父类。因为这两个结构体都有一个属性，那么我们就用一个Layout的`Protocol`来解决这个问题：

```Swift   
protocol Layout {
   var frame: CGRect {get set}
}
struct DecoratingLayout {
   var content: Layout
   var decoration: Layout
   mutating func layout(in rect: CGRect) {
      content.frame = ...
      decoration.frame = ...
   }
} 
```         

然后为了让`UIView`和`SKNode`都是用这个`Layout`，我们只用`Protocol Extension`。

```Swift    
extension UIView : Layout{}
extension SKNode : Layout{}
```      

>我们用Protocol和Extension实现了多态，而没有使用继承!

 这样我们的Layout就不必依赖于UIKit了。

但是上面的代码有一个Bug，我们不能保证`content`和`decoration`是相同的类型，他们可能一个是`UIView`，一个是`SKNode`。我们可以使用**泛型**来解决这个问题。我们可以给`DecoratingLayout`添加泛型：

```Swift
struct DecoratingLayout<Child: Layout> {
   var content: Child
   var decoration: Child
   mutating func layout(in rect: CGRect) {
      content.frame = ...
      decoration.frame = ...
   }
}
```    

这样我么就可以保证了，content和decoration有相同的类型。使用泛型，可以让编译器对代码有一个更好的理解，然后会对代码做很多的优化。

但是如果出现了更加相似的布局，我们该怎样重用我们的代码呢？比如：

![SimilarLayout](http://upload-images.jianshu.io/upload_images/1513759-c5236656635ca10c.png)

重用代码，我们最先想到的是继承，但是继承会造成Override及变量的变更，这样我们很难从子类中就明确的推断出代码的含义，那么我们用组合来做这个事情，面对上面的第二张图，我们通常的做法是，创建两个UIView：

![Composition](http://upload-images.jianshu.io/upload_images/1513759-8a14f1f51e4c035c.png)

但是这样做有弊端的：类对象是很消耗性能的：开辟堆空间，内存管理，除此之外，UIView还有：绘制，事件处理。然而Struct是几乎不消耗性能，并且由于其实Value Type，所以它可以做更好的封装，你不需要关系别人对其进行修改。

我们可以使用如下方式：

```Swift
struct CascadingLayout<Child : Layout> {
   var children: [Child]
   mutating func layout(in rect: CGRect) {
... } 
} 
struct DecoratingLayout<Child : Layout> {
   var content: Child
   var decoration: Child
   mutating func layout(in rect: CGRect) {
      content.frame = ...
      decoration.frame = ...
   }
} 
```  

这两种方式组合，这里我们不需要`Layout`协议有frame属性，我们只需要其有`layout()`方法即可

```Swift
protocol Layout {
 mutating func layout(in rect: CGRect)
}
```   

然后，我们的`DecoratingLayout`和`CascadingLayout`遵守该协议即可：

```Swift
struct DecoratingLayout<Child : Layout, ...> : Layout {}
struct CascadingLayout<Child : Layout> : Layout {}
```  
   
这之后我们就可以用组合的方式实现上文提到的UI效果了：

```Swift 
// Composition of Values
let decoration = CascadingLayout(children: accessories)
var composedLayout = DecoratingLayout(content: content, decoration: decoration)
composedLayout.layout(in: rect)
```     

为了实现这种UIView的叠加效果，我们给Layout协议添加`contents`属性：

```Swift
protocol Layout {
  mutating func layout(in rect: CGRect)
  var contents:[Layout]{get}  // 这里可以是UIView或者SKNode
}
```  

但是这回出现和之前所说的一样的Bug，我们没有办法保证这个contents属性里是否既有UIView还有SKNode，为了解决这个问题，我们加上一个关联属性：

```Swift
protocol Layout {
  mutating func layout(in rect: CGRect)
  associatedtype Content
  var contents:[Content]{get}  // 这里可以是UIView或者SKNode
}
```   

然后在`DecoratingLayout`中，我们这样做：

```Swift
struct DecoratingLayout<Child : Layout> : Layout {
   ...
   mutating func layout(in rect: CGRect)
   typealias Content = Child.Content
   var contents: [Content] { get }
```  

如果我们要对其中的内容做以限制，那么我们可以这样做：

```Swift
struct DecoratingLayout<Child:Layout, Decoration:Layout where Child.Content == Decoration.Content> : Layout {
	var content: Child
	var decoration: Decoration
	mutating func layout(in rect: CGRect)
	typealias Content = Child.Content
	var contents: [Content] { get }
}
```  

这样我们的单元测试就不必依赖于具体的UIView或者是SKNode：

```Swift  
  func testLayout() {
   let child1 = TestLayout()
   let child2 = TestLayout()
   var layout = DecoratingLayout(content: child1, decoration: child2)
   layout.layout(in: CGRect(x: 0, y: 0, width: 120, height: 40))
   XCTAssertEqual(layout.contents[0].frame, CGRect(x: 0, y: 5, width: 35, height: 30))
   XCTAssertEqual(layout.contents[1].frame, CGRect(x: 35, y: 5, width: 70, height: 30))
}

struct TestLayout : Layout {
   var frame: CGRect
   ...  
}
```  

## ValueType怎样用到Controller中

我们增加了Favoriate功能之后，我们的撤销功能消失了。为了隔离我们的Model，我们将其变为一个Struct，并且拥有之前的属性：

```Swift
class DreamListViewController : UITableViewController {
   var model: Model
... 
} 
struct Model : Equatable {
    var dreams: [Dream]
    var favoriteCreature: Creature
} 
```  

然后将Model和View隔离的做法极其容易出bug，因为我们的Model的任何改变都要对应相应View的位置。 
我们可以将其合并起来：

```Swift   
/// Diffs the model changes and updates the UI based on the new model.
    private func modelDidChange(diff: Model.Diff) {
        // Check to see if we need to update any rows that present a dream.
        if diff.hasAnyDreamChanges {
            switch diff.dreamChange {
                case .inserted?:
                    let indexPath = IndexPath(row: diff.from.dreams.count, section: Section.dreams.rawValue)
                    tableView.insertRows(at: [indexPath], with: .automatic)

                case .removed?:
                    let indexPath = IndexPath(row: diff.from.dreams.count - 1, section: Section.dreams.rawValue)
                    tableView.deleteRows(at: [indexPath], with: .automatic)

                case .updated(let indexes)?:
                    let indexPaths = indexes.map { IndexPath(row: $0, section: Section.dreams.rawValue) }
                    tableView.reloadRows(at: indexPaths, with: .automatic)

                case nil: break
            }
        }
```    

如果我们的UI有各种状态，并且我们要处理这种状态，这时，该怎样处理才比较好呢？比如我们的这个App中有:展示、选择、分享三种状态。这时如果遇到状态的切换就会遇到问题，因为Cell是复用的，所以就会出现忘记清除前一种Cell的状态而引起的Bug。
这种情况出现的原因就是我们的ViewController持有了这些不同的状态，如果状态逐渐变多，那么就会有非常复杂的业务逻辑需要处理。

```Swift
class DreamListViewController : UITableViewController {
    var isInViewingMode: Bool
    var sharingDreams: [Dream]?
    var selectedRows: IndexSet?
    ... 
} 
```  

因为UI只能出现一种状态，所以每次我们切换状态，我们就需要给其它的两种状态进行置空，这时就很容易引起Bug，为了解决这个问题，我们可以使用`Enum`来讲这些状态进行封装。

```Swift
enum State {
	case viewing
	case sharing(dreams:[Dream])
	case selecting(selectedRows: IndexSet)
}
``` 

这样我们就可以避免因为状态转换而产生的Bug了。



   



 



