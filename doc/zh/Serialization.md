---
title: Serialization
layout: documentation
documentation: true
---
序列化本文阐述了 Storm 0.6.0 以上版本的序列化机制。在低于 0.6.0 版本的 Storm 中使用了另一种序列化系统，详细信息可以参考 Serialization (prior to 0.6.0) 一文。
Storm 中的 tuple 可以包含任何类型的对象。由于 Storm 是一个分布式系统，所以在不同的任务之间传递消息时 Storm 必须知道怎样序列化、反序列化消息对象。
Storm 使用 Kryo库 对对象进行序列化。Kryo 是一个灵活、快速的序列化库。Storm 默认支持基础类型、string、byte arrays、ArrayList、HashMap、HashSet 以及 Clojure 的集合类型的序列化。如果你需要在 tuple 中使用其他的对象类型，你就需要注册一个自定义的序列化器。
Dynamic typing在 tuple 中没有对各个域（field）的直接类型声明。你需要将对象放入对应的域中，然后 Storm 可以动态地实现对象的序列化。在学习序列化接口之前，我们先来了解一下为什么 Storm 的 tuple 是动态类型化的。
为 tuple fields 增加静态类型会大幅增加 Storm 的 API 的复杂度。比如 Hadoop 就将它的 key 和 value 都静态化了，这就要求用户自己添加大量的注解。使用 Hadoop 的 API 非常繁琐，而相应的“类型安全”不值得。相对的，动态类型就非常易于使用。
进一步说，也不可能有什么合理的方法将 Storm 的 tuple 的类型静态化。假如一个 Bolt 订阅了多个 stream，从这些 stream 传入的 tuple 很可能都带有不同的类型。在 Bolt 的 execute 方法接收到一个 tuple 的时候，这个 tuple 可能来自任何一个 stream，也可能包含各种组合类型。也许你可以使用某种反射机制来为 bolt 订阅的每个 tuple stream 声明一个方法类处理 tuple，但是 Storm 可以提供一种更简单、更直接的动态类型机制来解决这个问题。
最后，Storm 使用动态类型定义的另一个原因就是为了用简洁直观的方式使用 Clojure、JRuby 这样的动态类型语言。
Custom serialization前面已经提到，Storm 使用 Kryo 来处理序列化。如果要实现自定义的序列化生成器，你需要用Kryo注册一个新的序列化生成器。强烈建议读者先仔细阅读 Kryo主页 来理解它是怎样处理自定义的序列化的。
可以通过topology的 topology.kryo.register 属性来添加自定义序列化生成器。该属性接收一个注册器列表，每个注册项都可以使用以下两种注册格式中的一种格式：
1.只有一个待注册的类的名称。在这种情况下，Storm 会使用 Kryo 的 FieldsSerializer 来序列化该类。这也许并不一定是该类的最优化方式 —— 可以查看 Kryo 的文档来了解更多细节内容。
2.一个包含待注册的类的名称和实现了com.esotericsoftware.kryo.Serializer接口的类组成的集合。
举例：
topology.kryo.register:
  - com.mycompany.CustomType1
  - com.mycompany.CustomType2: com.mycompany.serializer.CustomType2Serializer
  - com.mycompany.CustomType3
com.mycompany.CustomType1 和 com.mycompany.CustomType3 会使用 FieldsSerializer，而com.mycompany.CustomType2 则会使用 com.mycompany.serializer.CustomType2Serializer来实现序列化。
在topology的配置中，Storm 提供了用于注册序列化生成器的帮助类。Config 类有一个 registerSerialization 方法可以将序列化生成器注册到配置中。
Config 中有一个更高级的配置项做 Config.TOPOLOGY_SKIP_MISSING_KRYO_REGISTRATIONS。如果你将该项设置为 true，Storm 会忽略掉所有已注册但是在topology的类路径上没有相应的代码的序列化器。否则，Storm 会在无法查找到序列化器的时候抛出错误。如果你在集群中运行有多个topology并且每个topology都有不同的序列化器，但是你又想要在storm.yaml 中声明好所有的序列化器，在这种情况下这个配置项会有很大的帮助。
Java serialization如果 Storm 发现了一个没有注册序列化器的类型，它会使用 Java 序列化器来代替，如果这个对象无法被 Java 序列化器序列化，Storm 就会抛出异常。
注意，Java 自身的序列化机制非常耗费资源，而且不管在 CPU 的性能上还是在序列化对象的大小上都没有优势。强烈建议读者在生产环境中运行topology 的时候注册一个自定义的序列化器。保留 Java 的序列化机制主要为了便于设计新topology 的原型。
你可以通过将 Config.TOPOLOGY_FALL_BACK_ON_JAVA_SERIALIZATION 配置为 false 的方式来将序列化器回退到 Java 的序列化机制。
Component-specific serialization registrations
Storm 0.7.0 支持对特定组件的配置（详情请参阅Storm配置一文）。当然，如果某个组件定义了一个序列化器，这个序列化器也需要能够支持其他的 bolt —— 否则，后续的 bolt 将会无法接收来自该组件的消息！
在提交topology 的时候，topology 会选择一组序列化器用于在所有的组件间传递消息。这是通过将特定组件的序列化器注册信息与普通的序列化器信息融合在一起实现的。如果两个组件为同一个类定义了两个序列化器，Storm 会从中任意选择一个。
如果在两个组件的序列化器注册信息冲突的时候需要强制使用一个序列化器，可以在topology 级的配置中定义你想要的序列化器。对于序列化器的注册信息，拓扑中配置的值是优先于具体组件的配置的。

