# 异步Rust的现状

随着时间的推移，异步Rust生态系统经历了许多演变，因此很难知道使用什么工具、投资(？_invest in_)什么库或阅读什么文档，而标准库中的`Future`trait和`async`/`await`最近才进入稳定版本，因此整个生态系统正处于向新稳定的API迁移的过程中，在此之后，点流失将显著减少。

就现在来看，Rust异步编程的生态仍处于高速发展的阶段，而其体验还需进一步的打磨。许多库还在使用`futures`crate的0.1版本，也就是说为了实现互操作(？_to interoperate_)，开发者得从0.3版本的`futures`crate中使用`compat`。`async`/`await`的语言特性还处于新生阶段，像`async fn`这样的重要语法扩展也尚未实现，而现阶段编译器的错误也很难解析。

也就是说，Rust正在探索为其异步编程提供最具性能和易于编写的方式，如果您不害怕探索的过程，就请尽情享受在Rust中深入研究异步编程的世界吧！