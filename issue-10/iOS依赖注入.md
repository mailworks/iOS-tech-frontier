iOS依赖注入
---

> * 原文链接 : [Dependency Injection: Give Your iOS Code a Shot in the Arm
June 05, 2015](https://corner.squareup.com/2015/06/dependency-injection-in-objc.html)
* 原文作者 : [Square Engineering Blog](https://corner.squareup.com)
* 译文出自 : [开发技术前线 www.devtf.cn](http://www.devtf.cn)
* 译者 : [HarriesChen](https://github.com/mrchenhao) 

##什么是依赖注入呢?
依赖注入（DI）是一种非常流行的设计模式在许多的语言之中，比如说Java和C#,但是它似乎并没有在Objective-C广泛的被采用，这篇文章的目的就是简要的介绍一下在Objective-C中使用依赖注入的例子。虽然代码是用Objective-C写的，但是也可以应用到Swift。

依赖注入的概念很简单：一个对象应该被依赖而不是被创建。强烈建议阅读Martin Fowler的[excellent discussion on the subject](http://martinfowler.com/articles/injection.html)作为背景阅读。

注入可以通过对象的初始化（或者可以说构造器）或者通过特性（setter），这些就指的是构造器注入和setter方法注入。

##构造器注入

```
- (instancetype)initWithDependency1:(Dependency1 *)d1 
                        dependency2:(Dependency2 *)d2;
                        
```

##Setter 注入

```
@property (nonatomic, retain) Dependency1 *dependency1;
@property (nonatomic, retain) Dependency2 *dependency2;
```

根据福勒的描述，[首选构造器注入](http://martinfowler.com/articles/injection.html#ConstructorVersusSetterInjection)，一般来说只有在构造器注入不可行的情况下才会选择settter注入。在构造器注入中，你可以还是会使用`@property`来定义这些依赖，但是在接口中你应该让他们为只读，

##为什么使用依赖注入
依赖注入提供了一系列的好处，其中重要的是一下几点：

* **依赖定义清晰** 可以很明显的看出对象需要什么来进行操作，从而可以避免依赖类似（dependencies—like），全部变量消失（globals—disappear）。
* **组合** 依赖注入提倡[组合而不是继承](https://en.wikipedia.org/wiki/Composition_over_inheritance), 提高代码的可重用性。
* **定制简单** 当创建一个对象的时候，可以轻松的定制对象的每一个部分来面对最糟糕的情况。
* **所有权清晰** 特别是当使用构造函数注入，严格执行对象的所有权规则，帮助建立一个[有向无循环对象图](https://en.wikipedia.org/wiki/Directed_acyclic_graph)。
* **可测试性** 更重要的是，依赖注入提高了你的对象的可测性。因为他们可以被非常简单的创建通过初始化方法，没有隐藏的依赖关系需要管理。此外，可以简单的模拟出依赖让你关注于被测试的对象。

##使用依赖注入

你的代码可能还没有使用依赖注入的设计模式来设计，但是很容易上手。依赖注入的一个很好的方面是，你不需要采用“全或无”。相反，你可以将它应用到代码的特定区域并从那里展开。

###类型注入

首先，让我们把类分成两种：基本类和复合类。基本类是没有依赖的，或只依赖于其他基础类。基本类被继承的可能性极低，因为它们的功能是明确的，不变的，不引用外部资源。Cocoa本身有很多的基本类，如`NSString` `NSArray`，`NSDictionary` `NSNumber`……。

复杂类则相反，他们有复杂的依赖关系，包括应用级别的逻辑（可能被改变），或者访问外部资源，比如磁盘，网络或者一些全局的服务。在你应用内的大多数类都是复杂的，包括大多数的控制器对象和模型对象。需要Cocoa类也食非常复杂的，比如说`NSURLConnection `或者`UIViewController `

判断这个的最简单方法就是拿起一个复杂类然后看它初始化其他复杂类的地方（搜索"alloc] init" or "new]"），介绍了类的依赖注入，改变这个实例化的对象是通过作为一个类的初始化参数而不是类的实例化对象本身。

###初始化中的依赖分配

让我们来看一个例子，一个子对象（依赖）被初始化作为父对象的一部分。一般代码如下：

```
@interface RCRaceCar ()

@property (nonatomic, readonly) RCEngine *engine;

@end


@implementation RCRaceCar

- (instancetype)init
{
   ...

   // Create the engine. Note that it cannot be customized or
   // mocked out without modifying the internals of RCRaceCar.
   _engine = [[RCEngine alloc] init];

   return self;
}

@end

```

依赖注入版本的有一些小小的不同

```
@interface RCRaceCar ()

@property (nonatomic, readonly) RCEngine *engine;

@end


@implementation RCRaceCar

// The engine is created before the race car and passed in
// as a parameter, and the caller can customize it if desired.
- (instancetype)initWithEngine:(RCEngine *)engine
{
   ...

   _engine = engine;

   return self;
}

@end

```

###依赖惰性初始化

一些对象有时会在很后面才会被用到，有时甚至在初始化之后不会被用到，举个例子（在依赖注入之前）也许看起来是这样子的：

```
@interface RCRaceCar ()

@property (nonatomic) RCEngine *engine;

@end


@implementation RCRaceCar

- (instancetype)initWithEngine:(RCEngine *)engine
{
   ...

   _engine = engine;

   return self;
}

- (void)recoverFromCrash
{
   if (self.fire != nil) {
      RCFireExtinguisher *fireExtinguisher = [[RCFireExtinguisher alloc] init];
      [fireExtinguisher extinguishFire:self.fire];
   }
}

@end

```

在这种情况下，赛车希望不会出事，而我们也不需要使用灭火器。由于需要该对象的机会很低，我们不想每一个赛车的创建的时候慢下来。另外，如果我们的赛车需要从多个事故恢复，就需要创建多个灭火器。针对这些情况，我们可以使用工厂。

工厂标准的Objective-C块不需要参数和返回一个对象的实例。一个对象可以控制当其依赖的是使用这些块而不需要详细了解如何创建。

这里是一个例子使用了依赖注入和工厂模式来创建灭火器。

```
typedef RCFireExtinguisher *(^RCFireExtinguisherFactory)();


@interface RCRaceCar ()

@property (nonatomic, readonly) RCEngine *engine;
@property (nonatomic, copy, readonly) RCFireExtinguisherFactory fireExtinguisherFactory;

@end


@implementation RCRaceCar

- (instancetype)initWithEngine:(RCEngine *)engine
       fireExtinguisherFactory:(RCFireExtinguisherFactory)extFactory
{
   ...

   _engine = engine;
   _fireExtinguisherFactory = [extFactory copy];

   return self;
}

- (void)recoverFromCrash
{
   if (self.fire != nil) {
      RCFireExtinguisher *fireExtinguisher = self.fireExtinguisherFactory();
      [fireExtinguisher extinguishFire:self.fire];
   }
}

@end

```

工厂模式在我们创建未知数量的依赖的时候也是非常有用的，即使是在初始化期间，例子如下：

```
@implementation RCRaceCar

- (instancetype)initWithEngine:(RCEngine *)engine 
                  transmission:(RCTransmission *)transmission
                  wheelFactory:(RCWheel *(^)())wheelFactory;
{
   self = [super init];
   if (self == nil) {
      return nil;
   }

   _engine = engine;
   _transmission = transmission;

   _leftFrontWheel = wheelFactory();
   _leftRearWheel = wheelFactory();
   _rightFrontWheel = wheelFactory();
   _rightRearWheel = wheelFactory();

   // Keep the wheel factory for later in case we need a spare.
   _wheelFactory = [wheelFactory copy];

   return self;
}

@end

```

##避免麻烦的配置

如果对象不应该在其他对象内部分配，他们在哪里分配？是不是所有的这些依赖关系很难配置？大多不都是一样的吗？这些问题的解决在于类的便利初始化（想一想+[NSDictionary dictionary]）。我们会把我们的对象图的配置移出我们正常的对象，把他们抽象出来，测试，业务逻辑。

在我们添加便利初始化方法之前，先确认它是有必要的。如果一个对象只有少量的参数在初始化方法之中，这些参数没有规律，那么简单初始化方法是没有必要的。调用者应该使用标准初始化方法。

我们需要从四个地方搜集依赖来配置我们的对象：

* **没有明确默认值的变量**	这些包括布尔类型和数字类型的有可能在每个实例中是不同的。这些变量应该在便利初始化方法的参数中。
* **共享对象** 这些应该在便利初始化方法中作为参数（例如电台频率），这些对象先前可能被访问作为一个单例或者通过父指针。
* **新创建的对象** 如果我们的对象不与另一个对象分享这种依赖关系，则一个合作者对象应该在类的便利初始化方法中实例化。这些对象事先在对象的实现中直接创建。
* **系统单例** Cocoa提供了很多单例可以直接被访问，例如`[NSFileManager defaultManager]`，那里有一个明确的目的只需要一个实例被使用在应用中，还有很多单例在系统中。

一个对于赛车的简单初始化看起来如下：

```
+ (instancetype)raceCarWithPitRadioFrequency:(RCRadioFrequency *)frequency;
{
   RCEngine *engine = [[RCEngine alloc] init];
   RCTransmission *transmission = [[RCTransmission alloc] init];

   RCWheel *(^wheelFactory)() = ^{
      return [[RCWheel alloc] init];
   };

   return [[self alloc] initWithEngine:engine 
                          transmission:transmission 
                     pitRadioFrequency:frequency 
                          wheelFactory:wheelFactory];
}
```

你的类的简单初始化方法应该是最合适的。一个经常被用到的（可重用）的配置在一个.m文件中，而一个特定的配置文件应该在在`@interface RaceCar (FooConfiguration)`的类别文件中比如`fooRaceCar `。

##系统单例

在Cocoa的许多对象中，只有一种实例将一直存在。
例如`[UIApplication sharedApplication]`, `[NSFileManager defaultManager]`, `[NSUserDefaults standardUserDefaults]`, 和 `[UIDevice currentDevice]`，如果一个对象对他们存在依赖，那么应该在初始化的参数中包含他们，即使也许在你的代码中只有一个实例，你的测试可能要模拟实例或创建一个实例每个测试避免测试相互依存。

建议避免创建全局引用单例，而是创建一个对象的单个实例当它第一次被需要的时候并把它注入到所有依赖它的对象中去。

##不可变构造器

偶尔在某个类的构造器或者初始化方法不能被改变或者调用的时候，我们可以使用setter注入。例如：

```
// An example where we can't directly call the the initializer.
RCRaceTrack *raceTrack = [objectYouCantModify createRaceTrack];

// We can still use properties to configure our race track.
raceTrack.width = 10;
raceTrack.numberOfHairpinTurns = 2;

```

setter注入允许你配置对象，但是它也引入了易变性，你必须进行测试和处理，幸运的是，有两个场景导致初始化方法不能改变和调用，他们都是可以避免的。

###类注册

使用类注册的工厂模式意味着不能修改初始化方法。

```
NSArray *raceCarClasses = @[
   [RCFastRaceCar class],
   [RCSlowRaceCar class],
];

NSMutableArray *raceCars = [[NSMutableArray alloc] init];
for (Class raceCarClass in raceCarClasses) {
   // All race cars must have the same initializer ("init" in this case).
   // This means we can't customize different subclasses in different ways.
   [raceCars addObject:[[raceCarClass alloc] init]];
}
```

一个简单的替代方法就是使用工厂块而不是类的声明列表。


```
typedef RCRaceCar *(^RCRaceCarFactory)();

NSArray *raceCarFactories = @[
   ^{ return [[RCFastRaceCar alloc] initWithTopSpeed:200]; },
   ^{ return [[RCSlowRaceCar alloc] initWithLeatherPlushiness:11]; }
];

NSMutableArray *raceCars = [[NSMutableArray alloc] init];
for (RCRaceCarFactory raceCarFactory in raceCarFactories) {
   // We now no longer care which initializer is being called.
   [raceCars addObject:raceCarFactory()];
}
```

###故事版

故事提供了一个方便的方式来设计你的用户界面，但也带来一些问题的时候。特别是依赖注入，实例化初始视图控制器在故事板中不允许你选择初始化方法。同样，当下面的事情就是在故事板中定义，目标视图控制器被实例化也不允许你指定的初始化方法。

解决这个问题的办法是避免使用故事板。这似乎是一个极端的解决方案，但是我们发现故事板有其他问题在大型团队的开发中。此外，没有必要失去大部分故事版的好处。xibs提供所有相同的好处和故事板，除了`segues`，但仍然让你自定义初始化。

##公共和私有

依赖注入鼓励你暴露更多的对象在你的公共接口。如前所述，这有许多好处。但当你构建框架，它会明显的膨胀公共API。使用依赖注入之前，公共对象A可能使用私有对象B（这反过来使用私有对象C），但对象B和C分别从未曝光的外部框架。使用依赖注入的对象A会使对象B暴露在公共初始化方法，对象B反过来将对象C暴露在公共初始化方法。。

```
// In public ObjectA.h.
@interface ObjectA
// Because the initializer uses a reference to ObjectB we need to
// make the Object B header public where we wouldn't have before.
- (instancetype)initWithObjectB:(ObjectB *)objectB;
@end

@interface ObjectB
// Same here: we need to expose ObjectC.h.
- (instancetype)initWithObjectC:(ObjectC *)objectC;
@end

@interface ObjectC
- (instancetype)init;
@end
```

对象B和对象C都是具体的实现，但你并不需要框架的使用者来担心他们，我们可以通过协议的方式来解决。

```
@interface ObjectA
- (instancetype)initWithObjectB:(id <ObjectB>)objectB;
@end

// This protocol exposes only the parts of the original ObjectB that
// are needed by ObjectA. We're not creating a hard dependency on
// our concrete ObjectB (or ObjectC) implementation.
@protocol ObjectB
- (void)methodNeededByObjectA;
@end
```

##结束语

依赖注入在 Objective-C(同样在Swift)，合理的使用会使你的代码更加易读、易测试和易维护。



