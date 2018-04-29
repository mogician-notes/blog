>[Bonbons](https://github.com/ws-node/Bonbons) is a data annotation style MVC-IoC framework that I am writing. Currently, I have developed some basic features based on my preferences and programming style.

### Why name is bonbons
**Bonbons** is a French word meaning 🍬 **candy**. The reason of using this name is that I wish Bonbons can be like a candy, as a super-sweet syntactic sugar collection of express in the MVC direction, to achieve efficient development, easy expansion, while ensuring robust and stable.

>[express](https://github.com/expressjs/express) is a fast, unopinionated, minimalist web framework for node.

### Tech stack
Because Bonbons is an MVC extension to express ( **4.16.3**+ ), it must be based on Node and JavaScript. However, as an annotation-style MVC framework, many decorators being used, and the request response content format is semantically represented. TypeScript has a considerable advantage in this scenario. So the full functionality requires the support of the TypeScript ( **2.8.1**+ ).

### Guide
Bonbons was originally designed to be an MVC extended IoC framework with static type dependency injection, powerful routing annotations, rich controller concepts, and more powerful middleware pipelines, pursuing high abstraction and replaceable extensibility. Express provides a very basic http server capability and also provides a basic middleware model, so as a basis is very appropriate. The TypeScript and reflect-metadata library provides support for design-time information reflection at runtime, supporting the implementation of dependency injection systems.
In simple terms, the TypeScript+Decorator+Controllable Range Service+Controller Construction Method Injection+Pipe Controls form the most basic programming style for Bonbons.

### More
For more information, you could want to read Bonbons' [README.md](https://github.com/ws-node/Bonbons/blob/master/README.md).

**Source** : [**issue page**](https://github.com/mogician-notes/blog/issues/3)
