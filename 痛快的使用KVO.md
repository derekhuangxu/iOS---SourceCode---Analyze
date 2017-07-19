## 前言

KVO是iOS开发当中必不可少的一个工具，可以说是使用最广泛的工具之一。无论你是要在检测某一个属性变化，还是构建viewmodel双向绑定UI以及数据，KVO都是一个十分使用的工具。


然而！！

KVO用起来太TMD麻烦了，要注册成为某个对象属性的观察者，要在适当的时候移除观察者状态，还要写毁掉函数，更蛋疼的是对象属性还要用字符串作为表示。其中任何一个地方都要注意很多点，而且因为Delegate回调函数的原因，导致代码分离，可读性极差，维护起来异常费劲。

所以说，对于我来说，能不用的时候，尽量绕过去用其他的方法，直到我发现了Facebook的开源框架[KVOController](https://github.com/facebook/KVOController)。

***
## 基本介绍

#### 1、主要结构

![屏幕快照 2017-07-19 上午12.51.20.png](http://upload-images.jianshu.io/upload_images/711112-37dfa51ee5ca534b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

事实上**KVOController**的实现只有2各类，第一个是NSObject的Category是我们使用的类，第二个则是具体的实现方法。


#### 2、NSObject + FBKVOController 分析

在Category的.h文件中有两个属性，根据备注可知区别在意一个是持有的，另一个不是。

    /**
     @abstract Lazy-loaded FBKVOController for use with any object
     @return FBKVOController associated with this object, creating one if necessary
     @discussion This makes it convenient to simply create and forget a FBKVOController, and when this object gets dealloc'd, so will the associated     controller and the observation info.
     */
    @property (nonatomic, strong) FBKVOController *KVOController;

    /**
     @abstract Lazy-loaded FBKVOController for use with any object
     @return FBKVOController associated with this object, creating one if necessary
     @discussion This makes it convenient to simply create and forget a FBKVOController.
     Use this version when a strong reference between controller and observed object would create a retain cycle.
     When not retaining observed objects, special care must be taken to remove observation info prior to deallocation of the observed object.
     */
    @property (nonatomic, strong) FBKVOController *KVOControllerNonRetaining;


Category的.m文件和其他文件类似，写的都是**setter**以及**getter**方法，并且在getter方法中对别对两个属性做了对于 FBKVOController 的初始化。


    - (FBKVOController *)KVOController
    {
      id controller = objc_getAssociatedObject(self, NSObjectKVOControllerKey);
  
      // lazily create the KVOController
      if (nil == controller) {
        controller = [FBKVOController controllerWithObserver:self];
        self.KVOController = controller;
      }
  
      return controller;
    }

    - (void)setKVOController:(FBKVOController *)KVOController
     {
      objc_setAssociatedObject(self, NSObjectKVOControllerKey, KVOController, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }

    - (FBKVOController *)KVOControllerNonRetaining
    {
      id controller = objc_getAssociatedObject(self, NSObjectKVOControllerNonRetainingKey);
  
      if (nil == controller) {
        controller = [[FBKVOController alloc] initWithObserver:self retainObserved:NO];
        self.KVOControllerNonRetaining = controller;
      }
  
      return controller;
    }

    - (void)setKVOControllerNonRetaining:(FBKVOController *)KVOControllerNonRetaining
    {
      objc_setAssociatedObject(self, NSObjectKVOControllerNonRetainingKey, KVOControllerNonRetaining,     OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }



### 3、FBKVOController分析


   #### 1）几个基本API

    /**
     @abstract Creates and returns an initialized KVO controller instance.
     @param observer The object notified on key-value change.
     @return The initialized KVO controller instance.
     */
    + (instancetype)controllerWithObserver:(nullable id)observer;


    /**
     @abstract Registers observer for key-value change notification.
     @param object The object to observe.
     @param keyPath The key path to observe.
     @param options The NSKeyValueObservingOptions to use for observation.
     @param block The block to execute on notification.
     @discussion On key-value change, the specified block is called. In order to avoid retain loops, the block must avoid referencing the KVO controller or an owner thereof. Observing an already observed object key path or nil results in no operation.
     */
    - (void)observe:(nullable id)object keyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options block:(FBKVONotificationBlock)block;


    /**
     @abstract Registers observer for key-value change notification.
     @param object The object to observe.
     @param keyPath The key path to observe.
     @param options The NSKeyValueObservingOptions to use for observation.
     @param action The observer selector called on key-value change.
     @discussion On key-value change, the observer's action selector is called. The selector provided should take the form of -propertyDidChange, -    propertyDidChange: or -propertyDidChange:object:, where optional parameters delivered will be KVO change dictionary and object observed. Observing nil or observing an already observed object's key path results in no operation.
     */
    - (void)observe:(nullable id)object keyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options action:(SEL)action;


    /**
     @abstract Block called on key-value change notification.
     @param observer The observer of the change.
     @param object The object changed.
     @param change The change dictionary which also includes @c FBKVONotificationKeyPathKey
     */
    typedef void (^FBKVONotificationBlock)(id _Nullable observer, id object, NSDictionary<NSKeyValueChangeKey, id> *change);


     
 * 第一个很简单了，是创建KVOController的实例
 * 第二个是注册键值变化的观察者，返回一个有固定参数的Block。需要注意的是，为了避免循环引用，尽量避免使用KVOController及其持有者。
 * 第三个和第二个一样，也是注册键值变化的观察者，但是返回的是一个选择子SEL，API介绍中还对选择子SEL进行了建议。
 * 第四个很简单，是第二个回调函数的Block。值得注意的是，observer以及object分别是变化的观察者以及属性变化的对象，所以我们书写的时候可以改成我们需要的样式，以此来免去另加的转换过程。



***
## 主要的实现逻辑

KVOController的实现需要有两个私有的成员变量：
  *  NSMapTable<id, NSMutableSet<_FBKVOInfo *> *> *_objectInfosMap;
  *  pthread_mutex_t _lock;

以及另一个暴露在外只读的属性：
  * @property (nullable, nonatomic, weak, readonly) id observer;

在实现过程中，作为 KVO 的管理者，其必须持有当前对象所有与 KVO 有关的信息，而在 KVOController 中，用于存储这个信息的数据结构就是 NSMapTable。为了保证线程安全，需要持有pthread_mutex_t锁，用于在操作NSMapTable时候使用。

#### 1、下面让我们看初始化方法：

    - (instancetype)initWithObserver:(nullable id)observer retainObserved:(BOOL)retainObserved
    {
      self = [super init];
      if (nil != self) {
        _observer = observer;
        NSPointerFunctionsOptions keyOptions = retainObserved ? NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPointerPersonality :   NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality;
        _objectInfosMap = [[NSMapTable alloc] initWithKeyOptions:keyOptions valueOptions:NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPersonality capacity:0];
        pthread_mutex_init(&_lock, NULL);
      }
      return self;
    }

很简单，主要工作是持有了传进来的**Observer**，初始化了**NSMapTable**以及初始化了**pthread_mutex_t**锁。
值得一提的是初始化** NSMapTable**，我们回看第二部分，在属性的区分就在于是否是持有，根据属性的名字也能看出，不持有的话，引用计数就不会加一。所以在初始化的时候明显的区分就是在创建**NSPointerFunctionsOptions**的时候，是StrongMemory还是WeakMemory。
通过方法**+ (instancetype)controllerWithObserver:(nullable id)observer**初始化的时候，默认为持有。


#### 2、注册观察者

通常情况下我们会使用可以回调Block的API，但是也有少数情况下会选择传递选择子SEL的API，我们这里只拿传递Block的方法举例子。

    - (void)observe:(nullable id)object keyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options block:(FBKVONotificationBlock)block
    {
      NSAssert(0 != keyPath.length && NULL != block, @"missing required parameters observe:%@ keyPath:%@ block:%p", object, keyPath, block);
      if (nil == object || 0 == keyPath.length || NULL == block) {
        return;
      }

      // create info
      _FBKVOInfo *info = [[_FBKVOInfo alloc] initWithController:self keyPath:keyPath options:options block:block];

      // observe object with info
      [self _observe:object info:info];
    }

在这里传递进来的一些参数会被封装成为私有的**_FBKVOInfo**，那我们来简单看一下**_FBKVOInfo**的主要实现：

    {
    @public
      __weak FBKVOController *_controller;
      NSString *_keyPath;
      NSKeyValueObservingOptions _options;
      SEL _action;
      void *_context;
      FBKVONotificationBlock _block;
      _FBKVOInfoState _state;
    }

    - (instancetype)initWithController:(FBKVOController *)controller
                               keyPath:(NSString *)keyPath
                               options:(NSKeyValueObservingOptions)options
                                 block:(nullable FBKVONotificationBlock)block
                                action:(nullable SEL)action
                               context:(nullable void *)context
    {
      self = [super init];
      if (nil != self) {
        _controller = controller;
        _block = [block copy];
        _keyPath = [keyPath copy];
        _options = options;
        _action = action;
        _context = context;
      }
      return self;
    }

由此可以看出，**_FBKVOInfo**的主要作用就是起到了一个类似Model一样存储主要数据的作用，并储存了一个**_FBKVOInfoState**作为表示当前的 KVO 状态。
需要注意的是，成员变量都是用了**@public**修饰。
另外，对**- (NSString *)debugDescription**以及**- (NSString *)debugDescription**两个方法做了重写，方便了使用以及Debug。


之后执行了私有方法**- (void)_observe:(id)object info:(_FBKVOInfo *)info**

    - (void)_observe:(id)object info:(_FBKVOInfo *)info
    {
      // lock
      pthread_mutex_lock(&_lock);

      NSMutableSet *infos = [_objectInfosMap objectForKey:object];

      // check for info existence
      _FBKVOInfo *existingInfo = [infos member:info];
      if (nil != existingInfo) {
        // observation info already exists; do not observe it again

        // unlock and return
        pthread_mutex_unlock(&_lock);
        return;
      }

      // lazilly create set of infos
      if (nil == infos) {
        infos = [NSMutableSet set];
        [_objectInfosMap setObject:infos forKey:object];
      }

      // add info and oberve
      [infos addObject:info];

      // unlock prior to callout
      pthread_mutex_unlock(&_lock);

      [[_FBKVOSharedController sharedController] observe:object info:info];
    }


1）首先先进行的是对于自身持有的 **_objectInfosMap**这个成员变量的操作，一切都需要在先**锁定**，执行结束再**解锁**的过程。

  *  首先获取了对于当前观察者的注册的关注列表。
  *  判断是否当前需要关注的信息是否在此列表中，如果有则return出去，不再进行关注。
  *  如果当前的关注列表不存在则此时创建一个
  *  将关注的信息储存在关注列表中。

2）然后是获取了 **_FBKVOSharedController**单例并且执行了单例的 **- (void)observe:(id)object info:(nullable _FBKVOInfo *)info**方法。

     - (void)observe:(id)object info:(nullable _FBKVOInfo *)info
    {
      if (nil == info) {
        return;
      }

      // register info
      pthread_mutex_lock(&_mutex);
      [_infos addObject:info];
      pthread_mutex_unlock(&_mutex);

      // add observer
      [object addObserver:self forKeyPath:info->_keyPath options:info->_options context:(void *)info];

      if (info->_state == _FBKVOInfoStateInitial) {
        info->_state = _FBKVOInfoStateObserving;
      } else if (info->_state == _FBKVOInfoStateNotObserving) {
        // this could happen when `NSKeyValueObservingOptionInitial` is one of the NSKeyValueObservingOptions,
        // and the observer is unregistered within the callback block.
        // at this time the object has been registered as an observer (in Foundation KVO),
        // so we can safely unobserve it.
        [object removeObserver:self forKeyPath:info->_keyPath context:(void *)info];
      }
    }

加锁，对于当前单例的**NSHashTable**进行添加操作的信息，并执行**Foundation**的

    - (void)addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context;

然后对信息中的state进行更改。

#### 3、观察并回调

    - (void)observeValueForKeyPath:(nullable NSString *)keyPath
                      ofObject:(nullable id)object
                        change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change
                       context:(nullable void *)context
    {
      NSAssert(context, @"missing context keyPath:%@ object:%@ change:%@", keyPath, object, change);

      _FBKVOInfo *info;

      {
        // lookup context in registered infos, taking out a strong reference only if it exists
        pthread_mutex_lock(&_mutex);
        info = [_infos member:(__bridge id)context];
        pthread_mutex_unlock(&_mutex);
      }

      if (nil != info) {

         // take strong reference to controller
        FBKVOController *controller = info->_controller;
        if (nil != controller) {

          // take strong reference to observer
          id observer = controller.observer;
          if (nil != observer) {

            // dispatch custom block or action, fall back to default action
            if (info->_block) {
              NSDictionary<NSKeyValueChangeKey, id> *changeWithKeyPath = change;
              // add the keyPath to the change dictionary for clarity when mulitple keyPaths are being observed
              if (keyPath) {
                NSMutableDictionary<NSString *, id> *mChange = [NSMutableDictionary dictionaryWithObject:keyPath forKey:FBKVONotificationKeyPathKey];
                [mChange addEntriesFromDictionary:change];
                changeWithKeyPath = [mChange copy];
              }
              info->_block(observer, object, changeWithKeyPath);
            } else if (info->_action) {
    #pragma clang diagnostic push
    #pragma clang diagnostic ignored "-Warc-performSelector-leaks"
              [observer performSelector:info->_action withObject:change withObject:object];
    #pragma clang diagnostic pop
            } else {
              [observer observeValueForKeyPath:keyPath ofObject:object change:change context:info->_context];
            }
          }
        }
      }
    }

这个就相对简单了，主要是根据关注信息内是Block还是Action来执行，如果两者都没有就会调用观察者 KVO 回调方法。


#### 4、注销观察

事实上，注销是在执行dealloc的时候执行的，同时也去掉了锁：

    - (void)dealloc
    {
      [self unobserveAll];
      pthread_mutex_destroy(&_lock);
    }

因为KVO事件都由私有的 **_KVOSharedController** 来处理，所以当每一个 **KVOController** 对象被释放时，都会将它自己持有的所有 KVO 的观察者交由**  _KVOSharedControlle** r的方法处理，我们再来看下代码：

    - (void)unobserve:(id)object infos:(nullable NSSet<_FBKVOInfo *> *)infos
    {
      if (0 == infos.count) {
        return;
      }

      // unregister info
      pthread_mutex_lock(&_mutex);
      for (_FBKVOInfo *info in infos) {
        [_infos removeObject:info];
      }
      pthread_mutex_unlock(&_mutex);

      // remove observer
      for (_FBKVOInfo *info in infos) {
        if (info->_state == _FBKVOInfoStateObserving) {
          [object removeObserver:self forKeyPath:info->_keyPath context:(void *)info];
        }
        info->_state = _FBKVOInfoStateNotObserving;
      }
    }

该方法会遍历所有传入的 **_FBKVOInfo** ，从其中取出 **keyPath**  并将 **_KVOSharedController** 移除观察者。

当然，假如你需要手动的移除某一个的观察者，**_KVOSharedController** 也提供了方法：

    - (void)unobserve:(id)object info:(nullable _FBKVOInfo *)info
    {
      if (nil == info) {
        return;
      }

      // unregister info
      pthread_mutex_lock(&_mutex);
      [_infos removeObject:info];
      pthread_mutex_unlock(&_mutex);

      // remove observer
      if (info->_state == _FBKVOInfoStateObserving) {
        [object removeObserver:self forKeyPath:info->_keyPath context:(void *)info];
      }
      info->_state = _FBKVOInfoStateNotObserving;
    }

***
## 总结
这套框架提供了丰富的结构，基本能够满足我们对于KVO的使用需求。
只需要一次代码，就可以完成对一个对象的键值观测，同时不需要处理移除观察者，也可以在同一处代码进行键值变化之后的处理，从恶心的回调方法中解脱出来，不仅提供了使用方便，也不需要我们手动主要观察者，避免了各种问题，绝对算的上一个完善好用的框架。

***
## Refrence

  *  [如何优雅地使用 KVO](http://draveness.me/kvocontroller.html)
  *  [FBKVOController](https://github.com/facebook/KVOController)

***
## 另外

  *  [简书地址](http://www.jianshu.com/u/ac41d8480d04)
  *  [掘金地址](https://juejin.im/user/5730b373f38c840067d0d602)
