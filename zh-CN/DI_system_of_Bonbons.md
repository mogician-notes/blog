今天是五一假期的第一天，在家睡得很舒服，那就起来写篇文章。

>本文主要讲了Bonbons的依赖注入系统是如何设计的，但思路同样适用于设计任何后台IoC框架

Bonbons一开始便是打算按照IoC模式设计的，那么需要实现一套依赖系统，来解决控制器、服务的注入关系。我对很多著名的IoC框架也不是了解很多，所以在这里只是简单说说Bonbons的依赖注入系统的实现。

>目前Bonbons已经实现依赖注入功能(Dependency Injection)，但是还没有做依赖查找(Dependency Lookup)的相关实现，不过应该很快会完成，这是一个必备功能。

### 基本方式
Bonbons的依赖注入基本上按照接口和构造方法注入来完成。

之所以说是基本上，其实是因为在TypeScript本身来说，接口是一个设计概念，不会反映在JavaScript的运行过程里。那么完全使用TypeScript的接口概念，想要实现DI会有困难，所以Bonbons的DI用到的接口实际上是抽象类(abstract class)。

在TypeScript中抽象类是不可以实例化的，但是可以向接口一样，定义一些列的抽象方法或属性并强制需要在继承类中实现。事实上，抽象类在转译到JavaScript之后，和普通类并没有什么特殊的区别，所以在实现上，不需要额外关注抽象类本身。

### 细节

- 无接口定义直接登记
- 抽象类接口实现依赖分离
- 可注入项的依赖工厂
- 单例模式和范围模式
- 注入现成的的接口实例
- 依赖堆叠与引用解析
- 依赖不可达/循环引用的问题捕获
- DL扩展等支持

那我们来看看细节好了。

#### 1.无接口定义直接登记
虽然Bonbons是按照接口实现的依赖注入，但是同样支持隐式接口提供，即接口来源于当前登记类本身，在使用时通过类自身Constructor获取对应实例。这是Bonbons最基本最便利的依赖注入方式。

比如我们创建了这样的一个服务和控制器：
```TypeScript
@Injectabe()
export class SecService {

    private id = UUID.Create();

    constructor() {}

}

@Controller("api")
export class MainController extends BaseController {

    constructor(private sec: SecService ) {
        super();
    }

}
```
>装饰器部分的实现目前暂不做解释，暂时可以认为@Injectable()和@Controller(...)为运行时提供了设计时的类型源信息。事实上装饰器对于静态类型注入系统非常重要。

接下来我们在应用配置中注入：
```TypeScript
Bonbons.Create()
    .controller(MainController)
    .singleton(SecService)
    .listen(3000)
    .run(() => console.log("Example app listening on port 3000"));
```
> 这是目前Bonbons配置应用实例的方式，未来我将引入BonbonsTemplate模块来模板化声明式地创建应用程序，当然同样会使用装饰器来实现，可以参考Angular的NgModule等实现方式。

这样，在控制器中我们就可以拿到我们按照单例模式注册的SecService实例了(实例如何获得的将在后续说明)。IoC模式让依赖的获取控制权收回到了框架中，在依赖复杂并且高耦合的场景下，可以大幅度降低耦合度，帮助写出高可扩展的应用程序。

#### 2.抽象类接口实现依赖分离
直接注入有时候无法满足一些场景的需要。有时候我们不关心服务具体的业务逻辑实现，我们只是希望明确一个服务需要有哪些行为，因此透明化了服务的细节，因为我们毫不关注。以这样的思路来设计系统，我们可以轻易替换系统中的任何一个服务，只要他的他和替代者严格按照同一接口实现。

所以我们需要接口依赖注入。

之前说了TypeScript没有运行时接口，所以我们需要借助抽象类了，首先我们创建一个抽象类：
```TypeScript

export abstract class ABCService {
    public abstract getMessage(): string;
}

```

然后我们创建一个服务，并实现了抽象类，并且以不同的方式登记注册：
```TypeScript
@Injectabe()
export class MainService implements ABCService {

    private id = “main service id”;

    constructor() { }

    public getMessage(): string {
        return this.id;
    }

}
```
```TypeScript
Bonbons.Create()
    .controller(MainController)
    .singleton(ABCService, MainService)
    .listen(3000)
    .run(() => console.log("Example app listening on port 3000"));
```
>**因为我们把抽象类作为接口使用，所以可以运用TypeScript的接口实现特性，从类强行抽离接口来完成实现，当然你直接extends也没有什么不妥，只是最终在运行时会保留原型继承的关系，如果确定需要的话可以考虑这么做**

然后再控制器里注入就可以得到：
```TypeScript
@Controller("api")
export class MainController extends BaseController {

    constructor(private main: MainService) {
        super();
        main.getMessage();
    }
}
```
代码将按照预期的工作。

#### 3.可注入项的依赖工厂
那么如何实现依赖项的获取呢？没有什么特别的，就是直接采用关联数组来做的DI容器，对应的实现就是ES6的Map。
类型(即ES6 class的构造方法)将作为关联的键，而值则相对复杂一些，使用了工厂方法。
这里是DI容器单个值条目(DIEntry)的接口定义：
```TYPESCRIPT
interface DIEntryContract {
    _instance: any;
    _fac: any;
    getInstance() :any;
}
```
每个entry都至少包含这三个字段。getInstance方法是唯一用于获取结果的方法，另外两个字段从名字上就可以看出，是作为内部成员存在的。_instance字段会在单例模式中起作用，用来保留单例的引用；_fac则保留了工厂方法的引用，用于产生全新的目标实例。

通过关联数组，我们可以获得对应类型的DIEntry，并通过getInstance方法拿到需要的实例对象即可。
>至于_fac和getInstance的实现，将在后面的多重模式中继续展开。

#### 4.单例模式和范围模式
通常的IoC框架对于依赖项的范围也有区别可控制，比如Asp.Net Core，一个服务至少可以作为scoped存在，也可以作为singleton存在。

在Bonbons中，一个singleton服务就意味着整个应用生命周期中只会存在唯一的实例，并且它的构造方法依赖无论是什么类型的也都将作为单例存在。一个scoped服务则会确保在一次请求过程中保持唯一，并且他的构造方法依赖会按照依赖项自身的范围类型表现出正确的行为。

服务需要singleton还是scoped往往是按照实际需要来区分的。比如一个数据库的上下文对象往往是scoped的，因为我们不希望它在不同请求中产生副作用，因为它只是数据库的一个代理而已。

如何注入不同范围特性的服务呢？很简单，就像下面这样：
```TYPESCRIPT
Bonbons.Create()
    .controller(MainController)
    .singleton(SecService)
    .scoped(SuperService)
    .listen(3000)
    .run(() => console.log("Example app listening on port 3000"));
```
在框架内我定义了一个枚举用以区分不同范围的服务，目前暂时实现了上面两种类型：
```TYPESCRIPT
export enum InjectScope {
    Singleton = "__singleton",
    Scoped = "__scoped"
}
```

按照这个枚举，我重建了DIEntry的实现，实现了通过getInstance方法区分生产不同的范围对象：
```TYPESCRIPT
class DIEntry {
    _instance: any;
    _fac?: any;
    constructor(private scope: InjectScope) { }
    getInstance() {
        return this.scope === InjectScope.Singleton ? 
            (this._instance || (this._instance = this._fac())) : 
            this._fac();
    }
}
```
可以看见的是，scoped模式下，每一次请求实例都会通过工厂方法重建一个全新的，而singleton范围下，最终的结果总是相同的。

#### 5.注入现成的的接口实例
有一种场景需要额外关注，那就是提供一个现成的实例，提供接口然后自动实现单例模式。我们可能会初始化好一个接口实现，然后交给框架；框架确保在任何地方诸如得到的都是这个实例本身。

这在Bonbons里也是可行的。

```TYPESCRIPT
Bonbons.Create()
    .controller(MainController)
    .singleton(SecService, new SecService())
    .listen(3000)
    .run(() => console.log("Example app listening on port 3000"));
```
值得注意的是，无论你使用何种方式注入(singleton/scoped)，最终能够的效果总是singleton，即使使用scoped方法也是一样，下面的内部代码清晰的展示了这个逻辑：
```typescript
this.queue.push({ 
    el, 
    realel: registerValue, deps, 
    scope: isConstructor ? scope : InjectScope.Singleton
});
```

配置之后，实例化的对象同样按照之前的工厂方式构建，最终的DI行为也将保持一致。

#### 6.依赖堆叠与引用解析
当我们的所有可注入单体都是简单的，我们根本无需考虑他们的依赖关系，因为每一个实例的解析都非常简单：他们的构造方法参数是空的。

但如果一个服务需要注入另一个服务的时候，问题就会变得复杂起来了。至少有一点是很明确的，A依赖于B，那B就应该先于A被解析构造。这就涉及到一个依赖解析顺序的问题，一旦不能正确解析依赖，那应用程序可能会在运行时期引发异常。

所以我在这里引入了依赖堆叠的概念(这是我我自己用来描述Bonbons的DI关系的)，堆叠的每层单位数并非总是依次递减的，而是不确定的，但层于层之间的依赖总是单向的。

##### 依赖堆叠的结构：
* **堆底**：即第一层，所有的可注入服务中，构造方法参数列表为空的，都会被归纳到塔基中。这很好理解，这些服务作为所有其他服务的祖先，是整个依赖树的基石。
* **次级**：即第二层，所有本层的服务中，构造函数的参数类型至来源于堆底元素，且不能为空。堆底总是第二层的依赖的直接来源。
* **高层**：即三层+，第n(n>2)层的服务中，每个服务的构造方法依赖，总是来源于n层之下所有层级的总和，也就是说，以来总是来源于本层之下。

>严格来说，次级和高级层次可以合并，甚至包括堆底也是，但是考虑到算法实现上有明显的特异化，所以特殊为堆底和次级设立独立的级别。

框架需要做的是就是为所有注入的服务整理依赖，构建这样的依赖堆叠，排定依赖解析的顺序。构造依赖堆叠的时候，还能顺带解决其他不应该在运行时再去面对的问题，比如错误依赖(不可达问题)和循环引用的问题。

> 本篇Blog未完成，正在施工中...

**查看源** : [**issue页**](https://github.com/mogician-notes/blog/issues/4)