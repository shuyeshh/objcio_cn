本文由 [**Migrant**](http://objcio.com) 翻译自 [Custom Controls](http://www.objc.io/issue-3/custom-controls.html)，转载请注明出处。

本文将讨论一些自定义视图和控件的诀窍和技巧。我们先对UIKit已经提供给我们的控件做一个概览，介绍一些渲染技巧。随后我们会深入到视图和它们的所有者之间的通信策略，并简略探讨辅助功能，本地化和测试。

## 视图层次概览

看一下UIView的子视图，可以看到3个基本类:响应者，视图和控件。我们快速重温一下它们。

### UIResponder

`UIResponder`是`UIView`的父类。响应者能够处理触摸，手势，远程控制等事件。之所以它是一个单独的类而没有合并到`UIView`中，是因为`UIResponder`有更多的子类，最明显的就是`UIApplication`和`UIViewController`。通过重写`UIResponder`的方法，可以决定一个类是否可以成为第一响应者(例如当前输入焦点元素)。

当触摸或手势等交互行为发生，它们被发送给第一响应者(通常是一个视图)。如果第一响应者没有处理，则该行为沿着响应者链到达视图控制器，如果行为仍然没有被处理，则它继续传递给应用。如果想检测晃动手势，可以根据需要在这3层中的任意位置处理。

`UIResponder`还允许自定义输入方法，通过`inputAccessoryView`向键盘添加辅助视图或使用`inputView`提供一个完全自定义的键盘。

### UIView

`UIView`的子类处理所有跟内容绘制有关的事情和触摸。只要写过"Hello, World"应用的人都知道视图，但我们重申一些技巧点:

一个普遍的错误概念是视图的区域是由它的frame定义的。实际上frame是一个派生属性，主要由center和bounds合成而来。不使用Auto Layout时，大多数人使用frame来定位改变视图的大小。作为警告，[官方文档](https://developer.apple.com/library/ios/#documentation/UIKit/Reference/UIView_Class/UIView/UIView.html#//apple_ref/occ/instp/UIView/frame)特别详细说明了一个注意事项:

> If the transform property is not the identity transform, the value of this property is undefined and therefore should be ignored.

另一个允许向视图添加交互的方法是使用手势识别器。注意它们并不工作在响应者上，而只在视图及其子类上工作。

### UIControl

`UIControl`建立在视图上，增加了更多的交互支持。最重要的是，它增加了target/action模式。看一下具体的子类，可以看到按钮，日期选择器，文本输入框，等等。创建交互控件时，你通常想要子类化一个`UIControl`。一些常见但并不是控件的类是工具栏按钮(虽然也支持target/action)和文本视图(这里需要你使用代理来获得通知)。

## 渲染

现在，我们接下去来到可见的部分:自定义渲染。正如Daniel在他的[文章](http://www.objc.io/issue-3/moving-pixels-onto-the-screen.html)中提到的，你可能想避免在CPU上做渲染而将其丢给GPU。这里有一条经验:尽量避免`drawRect:`，而使用现有的视图构建自定义视图。

通常最快速的渲染方法是使用图片视图。例如，假设你想画一个带有边框的圆形头像，像下面图片中这样:

![](/images/posts/2013-12-04-understanding-frame-05.png)

为了实现这个，我们用以下的代码创建了一个图片视图的子类:

```objc
// called from initializer
- (void)setupView
{
    self.clipsToBounds = YES;
    self.layer.cornerRadius = self.bounds.size.width / 2;
    self.layer.borderWidth = 3;
    self.layer.borderColor = [UIColor darkGrayColor].CGColor;
}
```

我鼓励各位读者深入了解`CALayer`及其属性，因为你用它能实现的大多数事情会比用Core Graphics自己画要快。然而一如既往，剖析自己的代码是十分重要的。

把可拉伸的图片和图片视图一起使用也可以极大的提高效率。在[Taming UIButton](http://robots.thoughtbot.com/post/33427366406/designing-for-ios-taming-uibutton)这个帖子中，Reda Lemeden探索了几中不同的绘图方法。在文章结尾处有一个很有价值的帖子:[a comment by Andy Matuschak](https://news.ycombinator.com/item?id=4645585)，解释了可拉伸图片是这些技术中最快的。原因是可拉伸图片在CPU和GPU之间的数据转移量最小，并且这些图片的绘制是经过高度优化的。

处理图片时，你也可以让GPU为你工作来代替使用Core Graphics。使用Core Image，你不必用CPU做任何的工作就可以在图片上建立复杂的效果。你可以直接在OpenGL上下文上直接渲染，所有的工作都在GPU上完成。

### 自定义绘制

如果决定了采用自定义绘制，有几种不同的选项可供选择。如果可能的话，看看是否可以生成一张图片并在内存和磁盘上缓存起来。如果内容是动态的，也许你可以使用Core Animation，如果还是行不通，使用Core Graphics。如果你真的想要接近底层，使用GLKit和原生OpenGL也不是那么难，但是需要做很多工作。

如果你真的选择了重写`drawRect:`，确保检查内容模式。默认的模式是将内容缩放以填充视图的范围，并且当视图的frame改变时并不会重新绘制。

## 自定义交互

正如之前所说的，自定义控件的时候，你几乎一定会扩展一个UIControl的子类。在你的子类里，可以使用目标-动作机制触发事件，如下面的例子:

```objc
[self sendActionsForControlEvents:UIControlEventValueChanged];
```

为了响应触摸，你可能更倾向于使用手势识别。然而如果想要更接近底层，仍然可以重写`touchesBegan`，`touchesMoved`和`touchesEnded`方法来访问原始的触摸行为。但虽说如此，创建一个手势识别的子类来把手势处理相关的逻辑从你的view或者view controller中分离出来，在很多情况下都是一种更合适的方式。

创建自定义控件时所面对的一个普遍的设计问题是向拥有它们的类中回传返回值。比如，假设你创建了一个绘制交互饼状图的自定义控件，想知道用户何时选择了其中一个部分。你可以用很多种不同的方法来解决这个问题，比如通过目标-动作模式，代理，block或者KVO，甚至通知。

### 目标-动作

老式的，通常也是最方便的方法是使用目标-动作。在用户选择后你可以在自定义的视图中做类似这样的事情:

```objc
[self sendActionsForControlEvents:UIControlEventValueChanged];
```

如果有一个视图控制器在管理这个视图，需要:

```objc
- (void)setupPieChart
{
    [self.pieChart addTarget:self 
                      action:@selector(updateSelection:)
            forControlEvents:UIControlEventValueChanged];
}
 
- (void)updateSelection:(id)sender
{
    NSLog(@"%@", self.pieChart.selectedSector);
}
```

这么做的好处是在自定义视图子类中需要做的事情很少，并且自动获得多目标支持。

### 代理

如果你需要更多的控制从视图发送到视图控制器的消息，通常使用代理模式。在我们的饼状图中，代码看起来大概是这样:

```objc
[self.delegate pieChart:self didSelectSector:self.selectedSector];
```

在视图控制器中，你要写如下代码:

```objc
@interface MyViewController <PieChartDelegate>

 ...

- (void)setupPieChart
{
    self.pieChart.delegate = self;
}
 
- (void)pieChart:(PieChart*)pieChart didSelectSector:(PieChartSector*)sector
{
    // Handle the sector
}
```

当你想要做更多复杂的工作而不仅仅是通知所有者值发生了变化时，这么做显然更合适。不过虽然大多数开发人员可以非常快速的实现自定义代理，但这种方式仍然有一些缺点:你有必要检查代理是否实现了你想要调用的方法(使用`respondsToSelector:`)，最重要的，通常你只有一个代理(或者需要创建一个代理数组)。也就是说，一旦视图所有者和视图之间的通信变得稍微复杂，我们几乎总是会采取这种模式。

### Block

另一个选择是使用block。再一次用饼状图举例，代码看起来大概是这样:

```objc
@interface PieChart : UIControl

@property (nonatomic,copy) void(^selectionHandler)(PieChartSection* selectedSection);

@end
```

在选取行为的代码中，只需要执行它。在此之前检查一下block是否被赋值非常重要，因为执行一个未被赋值的block会使程序崩溃。

```objc
if (self.selectionHandler != NULL) {
    self.selectionHandler(self.selectedSection);
}
```

这种方法的好处是可以把相关的代码在视图控制器中整合:

```objc
- (void)setupPieChart
{
    self.pieChart.selectionHandler = ^(PieChartSection* section) {
        // Do something with the section
    }
}
```

就像代理，每个动作通常只有一个block。另一个重要的限制是不要形成引用循环。如果你的视图控制器持有饼状图的强引用，饼状图持有block，block又持有视图控制器，就形成了一个引用循环。只要在block中引用self就会造成这个错误。所以通常代码会以这样结束:

```objc
__weak id weakSelf = self;
self.pieChart.selectionHandler = ^(PieChartSection* section) {
    MyViewController* strongSelf = weakSelf;
    [strongSelf handleSectionChange:section];
}
```

一旦block中的代码要失去控制(比如block中要处理的事情太多，导致block中的代码过多)，你还可以将它们抽离成独立的方法，而且你可能也已经使用了代理。

### KVO

如果喜欢KVO，你也可以用它来观察。这有一点神奇而且没那么直接，但当应用中已经使用，它是很好的解耦的设计模式。在饼状图类中，编写代码:

```objc
self.selectedSegment = theNewSelectedSegment;
```

当使用合成属性，KVO会拦截到该变化并发出通知。在视图控制器中，编写类似的代码:

```objc
- (void)setupPieChart
{
    [self.pieChart addObserver:self forKeyPath:@"selectedSegment" options:0 context:NULL];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context 
{
    if(object == self.pieChart && [keyPath isEqualToString:@"selectedSegment"]) {
        // Handle change
    }
}
```

根据你的需要，在`viewWillDisappear:`或`dealloc`中，还需要移除观察者。对同一个对象设置多个观察者很容易造成混乱。有一些技术可以解决这个问题，比如[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)或者更轻量级的[THObserversAndBinders](https://github.com/th-in-gs/THObserversAndBinders)。

### 通知

作为最后一个选择，如果你想要一个非常松散的耦合，可以使用通知来使其他对象得知变化。对于饼状图来说你几乎肯定不想这样，不过为了讲解的完整，这里介绍如何去做。在饼状图的的头文件中:

```objc
extern NSString* const SelectedSegmentChangedNotification;
```

在实现文件中:

```objc
NSString* const SelectedSegmentChangedNotification = @"selectedSegmentChangedNotification";

...

- (void)notifyAboutChanges
{
    [[NSNotificationCenter defaultCenter] postNotificationName:SelectedSegmentChangedNotification object:self];
}
```

现在订阅通知，在视图控制器中：

```objc
- (void)setupPieChart
{
    [[NSNotificationCenter defaultCenter] addObserver:self 
                                           selector:@selector(segmentChanged:) 
                                               name:SelectedSegmentChangedNotification
                                              object:self.pieChart];

}

...

- (void)segmentChanged:(NSNotification*)note
{
}
```

当添加了观察者，你可以不将饼状图作为参数`object`，而是传递`nil`，以接收所有饼状图对象发出的通知。就像KVO通知，你也需要在恰当的地方退订这些通知。

这项技术的好处是完全的解耦。另一方面，你失去了类型安全，因为在回调中你得到的是一个通知对象，而不像代理，编译器无法检查通知发送者和接受者之间的类型是否匹配。

## 辅助功能

苹果官方提供的标准iOS控件均有辅助功能。这也是推荐用标准控件创建自定义控件的另一个原因。

这或许可以作为一整期的主题，但是如果你编写自定义视图，[Accessibility Programming Guide](http://developer.apple.com/library/ios/#documentation/UserExperience/Conceptual/iPhoneAccessibility/Accessibility_on_iPhone/Accessibility_on_iPhone.html#//apple_ref/doc/uid/TP40008785-CH100-SW3)说明了如何创建控件辅助功能。最为值得注意的是，如果有一个视图中有多个需要辅助功能的元素，但它们并不是该视图的子视图，你可以让视图实现`UIAccessibilityContainer`协议。对于每一个元素，返回一个描述它的`UIAccessibilityElement`对象。

## 本地化

创建自定义视图时，本地化也同样重要。如辅助功能一样，这个可以作为一整期的话题。本地化自定义视图的最直接工作就是字符串内容。如果使用`NSString`，你不必担心编码问题。如果在自定义视图中展示日期或数字，使用日期和数字格式化类来展示它们。使用`NSLocalizedString`本地化字符串。

另一个本地化过程中很有用的工具是Auto Layout。例如，有在英文中很短的词在德语中可能会很长。如果根据英文单词的长度对视图的尺寸做硬编码，那么当翻译成德文的时候几乎一定会遇上麻烦。通过使用Auto Layout，让标签控件自动调整为内容的尺寸，并向依赖元素添加一些其他的限制以确保重新设置尺寸，使这项工作变得非常简单。苹果为此提供了一个很好的[introduction](http://developer.apple.com/library/ios/#referencelibrary/GettingStarted/RoadMapiOS/chapters/InternationalizeYourApp/InternationalizeYourApp/InternationalizeYourApp.html)。另外，对于类似希伯来语这种顺序从右到左的语言，如果你使用了leading和trailing属性，整个视图会自动按照从右到左的顺序展示，而不是硬编码的从左至右。

## 测试

最后，让我们考虑测试视图的问题。对于单元测试，你可以使用Xcode自带的工具或者其它第三方框架。另外，可以使用UIAutomation或者其它基于它的工具。为此，你的视图完全支持辅助功能是必要的。UIAutomation并未充分得到利用的一个功能是截图;你可以用它[自动对比](http://jeffkreeftmeijer.com/2011/comparing-images-and-creating-image-diffs/)视图和设计以确保两者每一个像素都分毫不差。(另一个无关的功能:你还可以使用它来为应用[自动生成截图](http://www.smallte.ch/blog-read_en_29001.html)，这在你有多个多国语言的应用时特别有用)。