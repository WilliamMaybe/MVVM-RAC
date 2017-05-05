# ReactiveCocoa

## 如何运作
```
[[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@"hello maybe"];
        return nil;
    }]
     subscribeNext:^(id x) {
         NSLog(@"%@", x);
    }];
```
从上述代码中，RAC到底是如何运作的呢?
## createSignal:
这个步骤会生成一个信号量，一般情况下会生成DynamicSignal，并且将subscriber(订阅者)的实现代码已block的形式保存下来。

`在这个阶段，只是单纯的存储了一个block而已`

```
// RACDynamicSignal.m
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	RACDynamicSignal *signal = [[self alloc] init];
	signal->_didSubscribe = [didSubscribe copy];
	return [signal setNameWithFormat:@"+createSignal:"];
}
```
## subscribeNext:
这一步骤里，生成了一个实现了RACSubscriber协议的类`RACSubscriber`，*不用奇怪，协议和类的名字是一样的*。这个就是订阅者，不过我们平时是不会用到的，这是RAC内部使用的类。

然后将我们真正需要执行的代码已block的形势保存下来。

```
// RACSignal.m
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock {
	NSCParameterAssert(nextBlock != NULL);
	
	RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:NULL completed:NULL];
	return [self subscribe:o];
}
```

这个函数最终返回的是RACDiposable，该类主要是控制订阅者是否取消接下来的工作。使用[RACDisposable dispose]即可取消掉。
## subscribe:
这个方法是RACSignal(基类)的方法，然后本身不实现，交给派生类实现该方法。本例子是RACDynamicSignal的使用。

```
// RACDynamicSignal.m
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCParameterAssert(subscriber != nil);

	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];

	if (self.didSubscribe != NULL) {
		RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
			RACDisposable *innerDisposable = self.didSubscribe(subscriber);
			[disposable addDisposable:innerDisposable];
		}];

		[disposable addDisposable:schedulingDisposable];
	}
	
	return disposable;
}
```
在代码中，subscriber会重新生成，转移给RACPassthroughSusbcriber (passthrough查了下字典 转移)

在最后会去使用RACScheduler进行处理`self.didSubscribe(subscriber)`

## RACDisposable
## RACScheduler

## bind:

```
typedef RACStream * (^RACStreamBindBlock)(id value, BOOL *stop);

- (instancetype)bind:(RACStreamBindBlock (^)(void))block;
```

将block与订阅者返回的value绑定在一起

返回类型其实是RACDynamicSignal

flattenMap:正是实用bind来巧妙的转换signal
### 实现方案

1. 创建一个控制信号的dispoable来管理生命周期
2. 实现2个block，一个completeSignal，一个addSignal。
3. completeSignal用于当切换的而来的block() -> RACStream信号量在complete的时候使用，通知完成逻辑
4. addSignal则是在被订阅的时候获取到block() -> RACStream信号量，让信号量被订阅，然后将实现next、error、complete，做出不同的处理
5. 完成上述准备之后，开始真正进行订阅状态的处理，处理自己的next、error、complete情形下的逻辑。在next中使用block获取到接下来要转换的signal，使用addSignal进行处理；error和complete则直接使用初始时的disposable，并且使用completeSignal结束周期


# Tip
- RAC_signalForSelector:并不适合使用有返回值的SEL
- RAC_signalForSelector:只会返回sendNext:，所以不能使用then等
- - 调用该方法的时候，会使用runtime替换掉消息转发的一系列的方法实现，然后返回Signal
- - 当类触发了Selector的时候，会走消息转发，最终来到RACForwardInvocation(...)中
- - 在RACForwardInvocation中只使用了sendNext，把函数的各个变量用Tuple的形势返回出来
- RAC_signalForSelector:fromProtocol:有时候会失灵，需要在设置完之后再设置下delegate
- - // 不知道为什么要这样做
- - self.scrollView.delegate = nil; 
- - self.scrollView.delegate = self;
- RACCommand可以使用execute:anything，然后使用signal时就会传出anything。
- RACSignal createSignal时block内返回的是dispose，通常我们都直接返回nil，在特定情况下，*`可以返回[RACDisposable disposablexxxxx]，然后在这个disposable中加入一些取消操作`* 
- `switchToLatest`是在[signal sendNext:otherSignal]时使用的，可以间接订阅到otherSignal
- [RACCommand execute:]后如果想在触发前操作使用executing
- 初始信号如果想能够持续响应，在flattenMap中返回的信号不能sendError，否则会直接dispose

#ViewModel-Based的导航操作
[MVVM With ReactiveCocoa](http://blog.leichunfeng.com/blog/2016/02/27/mvvm-with-reactivecocoa/)
##步骤
- 创建一个NavigationProtocol，申明push，pop，present，dismiss 等方法
- 创建一个ViewModelServices继承于NavigationProtocol，该协议除了继承之外，主要是整合加入子ViewModelServices
- 创建一个ViewModelServicesImpl来实现协议，并不需要实现具体算法，我们要利用RAC来解耦
- 创建一个NavigationControllerStack来统一管理整个app的Controller
- stack中拥有一个id<ViewModelServices>，下面我们就可以使用RAC来监听push，pop等方法了
- 最后，创建一个Router来管理ViewModel相对应的ViewController

## [GitBucket iOS](https://github.com/leichunfeng/MVVMReactiveCocoa)
在app代码中重点看`MRCNavigationControllerStack`,`MRCViewModelServices`,`MRCViewModelServicesImpl`,`MRCNavigationProtocol`,`MRCRouter`
