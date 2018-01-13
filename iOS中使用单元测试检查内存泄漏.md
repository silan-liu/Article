# iOS中使用单元测试检查内存泄漏

在任何的程序开发中，内存泄漏都是个需要令人重视的问题，因为它直接影响着程序的性能与质量，同时也影响着用户体验。要是用户用着用着，app内存占用过多被系统杀死，用户懵了，不知道咋回事就闪退了，及其不好。

所以解决app中的内存泄漏问题，显得尤其重要。检查内存泄漏问题，可以试用IDE自带的工具，这里不做介绍。下面主要介绍一种比较新奇有意思的思路，通过单元测试来检测。

## 主要原理

* weak修饰的对象在释放后会自动置为nil
* 给对象新赋值后，之前指向的对象会释放掉

解释下第二点

```
id obj = [NSObject new];

// 重新赋值，之前new的对象会被释放
obj = [NSObject new];
```

所以，在单元测试中，可结合这两个原则，来思考。

## delegate

一般的，我们会让delegate是weak的，如果不小心写成了strong，会引起循环引用。那么如何用单元测试来检查呢？

根据上面的原则，可以先创建一个对象obj后，赋给delegate，然后将obj重新赋值，由于之前的对象会释放掉，就意味着delegate所指向的对象也应该释放了，这时候如果delegate是weak修饰的，即可判断其是否为nil。如果不为nil，因为这里obj没有被其他对象引用，可以判定为会存在内存泄漏。


```
- (void)testDelagte {
	id obj = [Test new]; 
	ViewController *vc = [ViewController new]; 
	vc.delegate = obj; 
	   
	// obj被赋值，之前的对象会销毁，如果vc.delegate是weak修饰，则会变成nil
	obj = [Test new]; 
	   
	XCTAssertNil(vc.delegate);
}
``` 

## Observer

在我们自己封装观察者模式时，会将观察者的实例保存起来，等有事件发生时，再逐一通知。这时候，是不希望强引用实例的。这里，也可以通过单元测试检测一波~。

```
- (void)testObserver {
	NotificationCenter *center = [NotificationCenter new];
	
	Observer *observer = [Observer new];
	
	weak weakObserver = observer;
	
	[center addObserver:observer];
	
	observer = [Observer new];
	
	XCTAssertNil(weakObserver)
}
```

这里center是直接强引用observer的，所以测试不会通过。可以采用包装对象的方式，来避免强引用oberver。

```
@interface WrapperObj: NSObject
@property (nonatomic, weak) id obj;
@end
```

## block

block会capture内部使用的变量，进行retain，也是最容易造成循环引用的地方。一般的，在异步调用中，我们很多时候会使用block，保存回调，事情处理完后，再调用回调。比如网络请求。如果在网络请求结束后，还持有block，那么block内部的变量也不会释放掉，造成内存泄漏。所以，需要检测block是否有被清掉。

虽然使用weak可以避免，但是这里的检查重点是block是否清除的问题。

可以用上面的方式来写，只不过需要单独将block先赋给一个变量，稍显麻烦。

```
void (^completion)(void) = ^{
        NSLog(@"hello");
    };
    
__weak void (^weakCompletion)(void) = completion;
    
completion = ^{
   
};
    
NSLog(@"%@", weakCompletion);
```

还有另外一种方式。我们知道，在block被释放后，它持有的对象，引用计数会减1。因此，可以通过有意的让block持有对象，然后检查对象是否为nil，来得出block是否被移除。

```
id obj = [NSObject new];
    
__weak id weakObj = obj;
    
NSString *url = @"xxx";
NSData *data = [NSData dataWithContentsOfURL:[NSURL URLWithString:@"xxx"]];

// mock response
NetworkResponse *response = [NetworkManager mockResponse:url data:data];

[NetworkManager loadUrl:url completion:^{
	NSLog(@"%@", obj);
}];

// response回来后，看是否会清除completion
[response send];

obj = [NSObject new];
    
XCTAssertNil(weakObj);
```




