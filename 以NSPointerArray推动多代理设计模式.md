
# 以NSPointerArray推动多代理设计模式再出发



## 背景思考

在买买车的项目中，**如果需要一个网络接口请求来的数据来确定哪些内容需要屏蔽，哪些控件需要重新安排位置，或者一些要同步显示更新的数据等**。遇到这样的需求大多数人的做法 拍拍屁股走人，直接用通知`NSNotification` 来实现。包括我之前也是这么Naive...

![img](https://ws4.sinaimg.cn/large/006tNbRwly1fxfual9j0xj308c08c0sx.jpg)

仔细想想，如果涉及页面多，动态同步更改界面或是更新本地数据展示的实现是相对复杂的。坦率的说如果用简单粗暴的方法，前期是简单了，后期无疑是给自己挖坑，埋下了不少隐患。

1、体现在模块数据变更以及界面的同步存在困难；

2、设计上马大哈的情况下，很容易造成极其高的耦合度；

3、实时性和动态性可能不好保证。



## 设计模式



### UML图:

![image-20181122145124134](https://ws2.sinaimg.cn/large/006tNbRwly1fxgude7o2ij31f20k2diu.jpg)

> 如图所示，UI层使用Category来实现代理的方法，通过MMCSwitchManager单例 来管理多个代理，有添加和移除代理等操作，内部通过dispatchDelegates 来分发下达指令。

##### 这样的好处是实现了分层，只需要**代理类（UI）**实现了相关的协议就可以了，这就最大化的减少了**对象之间的耦合度**。

问题的难点在于实现这么多的ViewController的Delegate 需要MMCSwitchManager有数组来管理，在数据更新的时候，通过数组依次的通知到category方法。

当晚吃完饭，就开搞。用了NSMutableArray 于是乎自以为很快的写完了，很满意了。回家了。

第二天早上刷牙的时候，Emmm。。。想想不对劲。

![image-20181122151721272](https://ws4.sinaimg.cn/large/006tNbRwly1fxgv4fbzk4j30r80l27ml.jpg)

刷完牙之后，大概7点半，我就不管他有没有醒，反正劳资就立马玉良书记发了微信。

如图（请忽略错别字。。。）：

![IMG_9077 2](https://ws2.sinaimg.cn/large/006tNbRwly1fxgv1fm7cwj30u013ajw4.jpg)

果然玉良书记的想法跟我不谋而合。



接下来更加亦可赛艇的事情发生了，本来想到Delagate 改成__weak 类型存到数组里就完了。然鹅，这一切都是徒劳。观察到UIViewController的dealloc依然没有被执行。

*`NSArray NSMutableArray` 实现弱引用的方式。【注1】*

##### 为了解决数组 weak reference，最终决定选择了 `NSPointerArray`

## NSPointerArray

> Apple 官方文档是这么解释的：
>
> A collection similar to an array, but with a broader range of available memory semantics.
>
> ## Overview
>
> The pointer array class is modeled after [`NSArray`](https://developer.apple.com/documentation/foundation/nsarray), but can also hold `nil` values. You can insert or remove `nil` values which contribute to the array's [`count`](https://developer.apple.com/documentation/foundation/nspointerarray/1418453-count).
>
> A pointer array can be initialized to maintain strong or weak references to objects, or according to any of the memory or personality options defined by [`NSPointerFunctions.Options`](https://developer.apple.com/documentation/foundation/nspointerfunctions/options).
>
> The [`NSCopying`](https://developer.apple.com/documentation/foundation/nscopying) and [`NSCoding`](https://developer.apple.com/documentation/foundation/nscoding) protocols are applicable only when a pointer array is initialized to maintain strong or weak references to objects.
>
> When enumerating a pointer array with [`NSFastEnumeration`](https://developer.apple.com/documentation/foundation/nsfastenumeration) using `for...in`, the loop will yield any `nil` values present in the array. See [Fast Enumeration Makes It Easy to Enumerate a Collection](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/FoundationTypesandCollections/FoundationTypesandCollections.html#//apple_ref/doc/uid/TP40011210-CH7-SW30) in [Programming with Objective-C](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Introduction/Introduction.html#//apple_ref/doc/uid/TP40011210) for more information.
>
> ### Subclassing Notes
>
> `NSPointerArray` is not suitable for subclassing.



简单的总结：这表明 `NSPointerArray`是可以跟踪集合中的对象内存的，并且它是可以插入删除的，甚至是nil 值。



## 核心代码实现

##### 懒加载初始化NSPointerArray

这里使用`weakObjectsPointerArray`后得到的数组对对象的持有是弱引用。

```objective-c
@property (nonatomic, strong) NSPointerArray *delegates;


- (NSPointerArray *)delegates {
    
    if (!_delegates) {
        _delegates = [NSPointerArray weakObjectsPointerArray];
    }
    
    return _delegates;
    
}

```

##### 实现移除数组中存在nil的元素，值得注意的是代理类的delegate在界面被销毁 进入dealloc的时候，数组中的 delegate 会自动被设置为nil。

```objective-c

- (void)removeAllNilDelegates {
    
    if([self.delegates count] == 0 || !self.delegates ) {
        return;
    }
        
    for (NSUInteger i = 0; i < self.delegates.count; i += 1) {
        
        if (![self.delegates pointerAtIndex:i]) {
            [self.delegates removePointerAtIndex:i];
        }
        
    }
    
    [self.delegates compact];

}

```

##### 添加 代理 到数组

既然在dealloc的时候会被自动释放为nil，在添加代理的时候，我做了个判断来剔除数组中有nil的代理。

以此简化了使用的场景，我们在添加代理的时候，不需要考虑代理的释放问题。

```objective-c
- (void)addSwitchDelegates:(id<MMCSwitchableProtocal>)delegateSwith {

    [self removeAllNilDelegates];
    
    [self.delegates addPointer:(__bridge void*)delegateSwith];

    if ([delegateSwith respondsToSelector:@selector(switchView:)])
    {
        [delegateSwith switchView:@{@"reviewing":@([self reviewing])}];
    }
    
}
```

##### 移除某个delegate

如上功能已经实现，下面的功能暂时没用上的场景，作为预留的接口。

```objective-c
- (NSUInteger)indexOfDelegate:(id)delegateSwith {
    for (NSUInteger i = 0; i < _delegates.count; i += 1) {
        
        if ([self.delegates pointerAtIndex:i] == (__bridge void*)delegateSwith) {
            return i;
        }
        
    }
    return NSNotFound;
}

//移除delegate，现在暂时不需要用到了，当持有的对象被释放，执行dealloc的时候，在执行此方法也没有什么用处了，delegateSwith自动被释放了。
- (void)removeDelegates:(id<MMCSwitchableProtocal>)delegateSwith {
    
    if([self.delegates count] == 0 || !self.delegates ) {
        return;
    }
    
    NSUInteger index = [self indexOfDelegate:delegateSwith];
    
    if (index != NSNotFound) {
        [self.delegates removePointerAtIndex:index];
    }
    
    [self.delegates compact];
    [self removeAllNilDelegates];
    
 }
```

### 核心代码 分发事件

首先，我们存的是__bridge void* 

__bridge ：用作于普通的 C 指针与 OC 指针的转换，不做任何操作。

例如：

```c
void *p;
NSObject *objc = [[NSObject alloc] init];
p = (__bridge void*)objc;
```

这里的 void *p 指针直接指向了 NSObject *objc 这个 OC 类，p 指针并不拥有 OC 对象。

相反 __bridge id 就是取值。

```objective-c
- (void)dispatchDelegates {
    
    // 分发事件
    if([self.delegates count] == 0 || !self.delegates ) {
        return;
    }

    for (NSUInteger i = 0; i < self.delegates.count; i += 1) {
        id deleagteSwitch = (__bridge id)[self.delegates pointerAtIndex:i];
        
        if (deleagteSwitch && [deleagteSwitch respondsToSelector:@selector(switchView:)]) {
            [deleagteSwitch switchView:@{@"reviewing":@([self reviewing])}];
        }
        
    }
    
}
```



接下来就是UIViewController的集成，我们只需要addSwitchDelegates和实现协议方法，就大功告成了。

`@interface MMCSearchCarListCenterView : UIView<MMCSwitchableProtocal>`

`        [[MMCSwitchManager sharedInstance] addSwitchDelegates:self];`        

```objective-c

- (void)switchView:(NSDictionary *)userInfo{
    BOOL reviewing = [[MMCSwitchManager sharedInstance] reviewing];
    UIButton *payButton = (UIButton *)[self viewWithTag:3];

    if (reviewing) {
        //隐藏
      payButton.hidden = YES;
 } else {
        //不隐藏
        payButton.hidden = NO;
    }
    
}
```







------



[^注1]: 实现弱引用的NSArray NSMutableArray https://stackoverflow.com/questions/9336288/nsarray-of-weak-references-unsafe-unretained-to-objects-under-arc https://stackoverflow.com/questions/4692161/non-retaining-array-for-delegates



### NSArray：

```objective-c
NSValue *value = [NSValue valueWithNonretainedObject:myObj];
[array addObject:value];
```

### NSMutableArray：

```objective-c
@implementation NSMutableArray (WeakReferences)
    + (id)mutableArrayUsingWeakReferences {
    return [self mutableArrayUsingWeakReferencesWithCapacity:0];
    }

    + (id)mutableArrayUsingWeakReferencesWithCapacity:(NSUInteger)capacity {
    CFArrayCallBacks callbacks = {0, NULL, NULL, CFCopyDescription, CFEqual};
    // We create a weak reference array
    return (id)(CFArrayCreateMutable(0, capacity, &callbacks));
    }
@end
```

