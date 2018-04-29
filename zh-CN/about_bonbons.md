>[Bonbons](https://github.com/ws-node/Bonbons) 是我正在写的一个数据注解风格的MVC+IoC框架，目前按照自己的偏好和编程风格，开发了一些早期的功能。

### 为什么叫这个
**Bonbons**是一个法语词汇，意思是 🍬  **糖果**。之所以取这个名字，是希望Bonbons本身能像糖果一样，成为一套express在MVC方向上的超级甜的语法糖集合，实现高效开发，轻松扩展，同时保证健壮稳定。

>[express](https://github.com/expressjs/express) 是一个基于 Node.js 平台，快速、开放、极简的 web 开发框架。
>`express is a fast, unopinionated, minimalist web framework for node.`

### 基本技术组成
因为Bonbons是一个express ( **4.16.3+** ) 的MVC扩展，那么必定是基于Node和JavaScript的。但是作为一个注解风格的MVC框架，大量的使用了装饰器，以及语义化了请求响应内容格式，在这样的场景下**TypeScript**具有相当大的优势。所以完整的功能需要TypeScript ( **2.8.1+** ) 语言的支持。

### 风格指南 
Bonbons的设计初衷是要成为一个拥有静态类型依赖注入，强力的路由注解，丰富的控制器概念，和更强大的中间件管道的MVC扩展IoC框架，追求高度抽象和一切可替换的扩展性。express提更了很基本的http服务器能力，还提供了基础的中间件模式，所以作为基础是非常合适的。TypeScript和reflect-metadata库提供了设计时信息在运行时反射的支持，支撑实现了依赖注入系统。
简单来说，TypeScript+装饰器+可控制范围的服务+控制器构造方法注入+管道控制组成了Bonbons最基本的程序风格。

### 其他 
更多信息可以阅读Bonbons的[README](https://github.com/ws-node/Bonbons/blob/master/README.md)。

**查看源** : [**issue页**](https://github.com/mogician-notes/blog/issues/3)
