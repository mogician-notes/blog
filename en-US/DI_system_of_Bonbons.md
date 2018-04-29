Today is the first day of the Labor Day holiday. I slept well at home and decided to write an article.

>This article focuses on how Bonbons's dependency injection system is designed, and the same ideas apply to designing any background IoC framework.

Bonbons was originally planned to be designed in accordance with the IoC model, so it is necessary to implement a set of dependent systems to solve the injection relationship between controllers and services. I don't know much about many well-known IoC frameworks, so I'm just going to talk about Bonbons' implementation of the dependency injection system.

>Currently Bonbons has implemented Dependency Injection, but it has not yet implemented Dependency Lookup, but it should be completed soon. This is an essential feature.

### Basic 
Bonbons' dependency injection is essentially done in terms of interface and constructor injections.

The reason for this is basically because in TypeScript itself, the interface is a design concept that will not be reflected in the running environment of JavaScript. Then using the concept of TypeScript interface completely, it would be difficult to implement DI, so the interface used by Bonbons DI is actually abstract classes.

Abstract class are not instantiable in TypeScript, but you can define abstract methods or properties for columns as they are for interfaces and force them to be implemented in inherited classes. In fact, after the abstract class is translated into JavaScript, there is no special difference with the ordinary class, so there is no need to pay extra attention to the abstract class itself.

### Details

- Direct registration without interface definition
- Abstract class interface implementation dependency separation
- Dependent factory that can inject items
- Singleton mode and scoped mode
- Inject ready-made interface instances
- Dependent heap and reference resolution
- Problem capture of dependency unreachable/cyclic reference
- DL extension support

So let's take a look at the details.

#### 1.Direct registration without interface definition
Although Bonbons implements dependency injection according to the interface, it also supports implicit interface providing, that is, the interface is derived from the current registration class itself, and the corresponding instance is obtained by using the class's own constructor when it is used. This is Bonbons' most basic and convenient method of dependency injection.

For example, we created such a service and controller:
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
>The implementation of the decorator part is currently not explained. For the time being, **@Injectable()** and **@Controller(...)** can be regarded as providing the design-time type source information for the runtime. In fact, decorators are very important for static type injection systems.

Next we inject in the application configuration:
```TypeScript
Bonbons.Create()
    .controller(MainController)
    .singleton(SecService)
    .listen(3000)
    .run(() => console.log("Example app listening on port 3000"));
```
> This is the way Bonbons configures an application instance. In the future, I will introduce the BonbonsTemplate module to template-declaratively create an application. Of course, this will also be implemented by using a decorator. You can refer to Angular's NgModule implementation.

In this way, we can get the SecService instance that we registered according to the singleton mode in the controller (how to obtain the instance will be described later). The IoC mode relies on retrieving the acquired control rights into the framework. In the case of complex and high-coupling scenarios, the coupling degree can be greatly reduced, helping to write highly scalable applications.。

#### 2.Abstract class interface implementation dependency separation
Direct injection sometimes fails to meet the needs of some scenarios. Sometimes we don't care about the service-specific business logic implementation. We just want to be clear about what actions a service needs, and therefore transparently the details of the service because we don’t care. By designing the system in such a way, we can easily replace any service in the system, as long as his and his alternates are implemented in strict accordance with the same interface.

So we need interface dependency injection.

Before saying that TypeScript does not have a runtime interface, we need to use abstract classes. First we create an abstract class:
```TypeScript

export abstract class ABCService {
    public abstract getMessage(): string;
}

```

Then we create a service and implement the abstract class and register the registration in different ways:
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
>**Because we use the abstract class as an interface, we can use TypeScript's interface implementation features to force the interface to complete the implementation from the class, of course, you can still choose to extend the abstract class, but this will eventually retain the prototype inheritance relationship at runtime, If you really want to, you can consider doing so**

Then in the controller can be injected:
```TypeScript
@Controller("api")
export class MainController extends BaseController {

    constructor(private main: MainService) {
        super();
        main.getMessage();
    }
}
```
The code will work as expected.

#### 3.Dependent factory that can inject items
So how do you achieve the dependency? There is nothing special about DI containers that use associative arrays directly. The corresponding implementation is the ES6 Map.
The type (ie, the constructor of the ES6 class) will be the associated key, and the value will be a bit more complex, using factory methods.
Here is the interface definition of the DI container single value entry (DIEntry):
```TYPESCRIPT
interface DIEntryContract {
    _instance: any;
    _fac: any;
    getInstance() :any;
}
```
Each entry contains at least these three fields. The getInstance method is the only method used to get the result. The other two fields can be seen from the name and exist as internal members. The _instance field will work in singleton mode to keep the singleton reference; _fac will keep the reference to the factory method to generate a brand new target instance.

Through the associative array, we can obtain the corresponding type of DIEntry, and get the required instance object through the getInstance method.
>As for the implementation of _fac and getInstance, it will continue to expand in the singleton/scoped modes that follow.

#### 4.Singleton mode and scoped mode
The usual IoC framework also has different control over the scope of the dependency. For example, Asp.Net Core, a service can exist at least as scoped or as a singleton.

In Bonbons, a singleton service means that there will only be a single instance of the entire application life cycle, and its construction method depends on what type it will exist as a singleton. A scoped service ensures that it stays unique during a request, and its constructor dependencies show the correct behavior based on the scope type of the dependency itself.

Whether the service needs singleton or scoped is often differentiated according to actual needs. For example, a database's context object is usually scoped because we don't want it to have side effects on different requests because it's just a proxy to the database.

How to inject different range of features? It's simple, like this:
```TYPESCRIPT
Bonbons.Create()
    .controller(MainController)
    .singleton(SecService)
    .scoped(SuperService)
    .listen(3000)
    .run(() => console.log("Example app listening on port 3000"));
```
In the framework I define an enumeration to distinguish between different scopes of services. Currently these two types are temporarily implemented:
```TYPESCRIPT
export enum InjectScope {
    Singleton = "__singleton",
    Scoped = "__scoped"
}
```
>It is worth noting that the **session** mode is not implemented because the express-session module has not been included in Bonbons by me. This part of the function is still planning, in the future may follow the framework to upgrade and change, expand the range of types of injectable items.

In accordance with this enumeration, I rebuilt the DIEntry implementation and realized the different scoped objects produced by the getInstance method:
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
It can be seen that in the scoped mode, each request instance will be rebuilt by the factory method to a completely new one. Under the singleton scope, the final result is always the same.

#### 5.Inject ready-made interface instances
One scenario that requires extra attention is to provide a ready-made instance that provides an interface and then automatically implement the singleton mode. We may initialize an interface implementation and then hand it off to the framework; the framework ensures that everything that we get, for example, is the instance itself.

This is also feasible in Bonbons.

```TYPESCRIPT
Bonbons.Create()
    .controller(MainController)
    .singleton(SecService, new SecService())
    .listen(3000)
    .run(() => console.log("Example app listening on port 3000"));
```
It is worth noting that no matter which way you use (singleton/scoped), the ultimate effect is always a singleton, even if you use the scoped method. The following internal code clearly shows this logic:
```typescript
this.queue.push({ 
    el, 
    realel: registerValue, deps, 
    scope: isConstructor ? scope : InjectScope.Singleton
});
```

After the configuration, the instantiated objects are constructed in the same way as the previous factory, and the final DI behavior will remain the same.

#### 6.Dependent heap and reference resolution
When all our injectable monomers are simple, we don't need to consider their dependencies at all because the parsing of each instance is very simple: their constructor parameters are empty.

However, if a service needs to be injected into another service, the problem becomes complicated. At least one thing is clear, A depends on B, and B should be constructed before A is parsed. This involves a problem that depends on the order of the parsing. Once the dependency cannot be parsed correctly, the application may throw an exception at runtime.

So I've introduced the concept of dependency heap here (this is my own description of Bonbons' DI relationship). The number of stacked units per layer is not always decreasing, but is uncertain, but layered Interdependence is always one-way.

##### Depends on the heap structure：
* **Bottom**：That is, in the first layer, all of the injectable services, the construction method parameter list is empty, will be summarized in the tower base. It is well understood that these services, as the ancestors of all other services, are the cornerstone of the entire dependency tree.
* **Secondary**：That is, the second layer, all the services of this layer, the parameter types of the constructor function to the end of the heap element, and can not be empty. The bottom of the pile is always the direct source of dependence on the second level.
* **Higher**：That is, in the service of level 3+, in the **n** th (n>2) layer, the construction method of each service depends on the sum of all levels below the n level, that is, the dependency always comes from the level of the layer. under.

>Strictly speaking, the secondary and higher levels can be combined, even including the bottom of the heap, but given the obvious specialization of the algorithm implementation, special levels are set up for the bottom of the heap and the secondary level.

What the framework needs to do is to organize dependencies for all injected services, build such a heap of dependencies, and order the dependencies resolved. When constructing stacks of dependencies, it is also possible to solve other problems that should not be faced at runtime, such as error dependencies (inaccessible problems) and circular references.

#### 7.Problem capture of dependency unreachable/cyclic reference
The above explained the idea of parsing dependencies. Now it is time to look at the code. Because the code is written at the beginning, it will face many real problems, such as the handling of some errors.

The problem with two scenarios is very common: 1) Adding a service error injects an Object constructor. Obviously Object will not normally be registered for injection; 2) Two services inject themselves into each other.

Bonbons needs to circumvent these two issues (at least) before the app instance begins.

In order to solve the first problem, it is necessary to find the target item when parsing each dependency. If no search is found, it means that the dependency is unreachable:
```TYPESCRIPT
// sourceQueue : all dependencies registered by the application (either singleton or scoped)
// checkArr : when the level > 2, represents all the following set of dependencies, refer to above the dependency stack
// node : current checkable injectable item
const isresolved = node.deps.every(i => checkArr.map(m => m.el).includes(i));
if (!isresolved && !node.deps.every(i => sourceQueue.map(m => m.el).includes(i))) {
    throw resolveError(node.realel, node.deps);
}
```
>Since dependency checking is a process that is used when an application is started, it does not incur additional overhead for each request. Therefore, the algorithm implemented here is rough and does not take care of performance.

Another problem is relatively not difficult, because the circular reference page above a situation, only need to interrupt the upper logic and check in advance. Bonbons does not currently handle circular references, so circular references can cause dependant interrupts.

#### 8.DL extension support
The Dependency Lookup feature is not fully implemented, but it is actually not completely publicly configurable. At present, the DI function has been separated into components. It only needs to create an injector, and injects into the system in the same manner as the DI. Then it can inject as usual dependencies and manually obtain the instance through the injector's API. The design of the injector API will be added after consideration.

### Postscript
Although the Bonbons DI system is not powerful, the basic functions are also completed. In a simple test, the operation is very stable. It can be used as a reference, and where it does not work well, I hope that the you can also give me some advice.

**Source** : [**issue page**](https://github.com/mogician-notes/blog/issues/4)
