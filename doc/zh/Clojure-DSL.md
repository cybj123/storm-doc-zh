---
title: Clojure DSL
layout: documentation
documentation: true
---
Storm配有Clojure DSL，用于定义spouts(喷口)，bolts(螺栓)和topologies(拓扑)。 Clojure DSL可以访问Java API暴露的所有内容，因此如果您是Clojure用户，您可以直接编写Storm拓扑，根本不需要使用Java。 Clojure DSL 的源码在 [org.apache.storm.clojure]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/clojure.clj)命名空间中定义。

本页概述了Clojure DSL的所有功能，包括：

1. Defining topologies(定义拓扑)
2. `defbolt`
3. `defspout`
4. Running topologies in local mode or on a cluster(在本地模式或集群上运行拓扑)
5. Testing topologies(测试拓扑)

### Defining topologies(定义拓扑)

请使用`topology`函来定义topology(拓扑)。`topology`有两个参数：“spout specs(规格)”的映射和“bolt specs(规格)” 的映射。每个spouts specs(规格)和bolt specs(规格) 通过指定输入和并行度 来将组件的代码连接到topology中。


我们来看一下[storm-starter ]({{page.git-blob-base}}/examples/storm-starter/src/clj/org/apache/storm/starter/clj/word_count.clj):项目中的topology定义示例：

```clojure
(topology
 {"1" (spout-spec sentence-spout)
  "2" (spout-spec (sentence-spout-parameterized
                   ["the cat jumped over the door"
                    "greetings from a faraway land"])
                   :p 2)}
 {"3" (bolt-spec {"1" :shuffle "2" :shuffle}
                 split-sentence
                 :p 5)
  "4" (bolt-spec {"3" ["word"]}
                 word-count
                 :p 6)})
```

spout和bolt specs(规格)的映射是从组件ID到相应规格的映射。组件ID必须在映射上是唯一的。就像在Java中定义topologies(拓扑)一样，在声明topologies(拓扑)中的bolts 输入时使用的组件ID。

#### spout-spec（spout-规格）

`spout-spec`作为spout实现（实现 [IRichSpout](javadocs/org/apache/storm/topology/IRichSpout.html))的对象）的可选关键字参数的参数 。当前存在的唯一选项是：`:p`选项，它指定了spout的并行性。如果您省略`:p`，则spout将作为单个任务执行。

#### bolt-spec（bolt-规格）

`bolt-spec`作为bolt实现（实现IRichBolt的对象）的可选关键字参数的参数

输入声明是从stream ids到stream groupings的映射。stream id 可以有以下两种形式中的一种：

1. `[==component id== ==stream id==]`: Subscribes to a specific stream on a component(订阅组件上的特定流)
2. `==component id==`: Subscribes to the default stream on a component(订阅组件上的默认流)

stream grouping可以是以下之一：

1. `:shuffle`: 用shuffle grouping进行订阅
2.  Vector of field names, like ["id" "name"](多字段名, 像 `["id" "name"]`): 使用fields grouping订阅指定的字段
3. `:global`:  使用global grouping进行订阅
4. `:all`: 使用all grouping进行订阅
5. `:direct`: 使用direct grouping进行订阅

See [Concepts](Concepts.html) for more info on stream groupings. Here's an example input declaration showcasing the various ways to declare inputs:
有关stream groupings的更多信息，请参阅[概念](Concepts.html)。下面是一个输入声明的示例，展示各种声明输入的方法：

```clojure
{["2" "1"] :shuffle
 "3" ["field1" "field2"]
 ["4" "2"] :global}
```

此输入声明共计三个流。它通过随机分组来订阅组件“2”上的流“1”，在字段“field1”和“field2”上以fields grouping的方式订阅组件“3”上的默认流，使用全局分组在组件“4”上订阅流“2”。

像`spout-spec`一样，bolt-spec唯一当前支持的关键字参数是：p，它指定了bolt的并行性。

#### shell-bolt-spec (shell-bolt-规格)

`shell-bolt-spec` is used for defining bolts that are implemented in a non-JVM language. It takes as arguments the input declaration, the command line program to run, the name of the file implementing the bolt, an output specification, and then the same keyword arguments that `bolt-spec` accepts.
`shell-bolt-spec`用于定义以非JVM语言实现的bolts。它作为输入声明参数，在命令行程序中运行，用文件的名称实现bolt，输出规范 以及接受的相同关键字参数作为参数的 `bolt-spec` 。

这有一个shell-bolt-spec的例子：

```clojure
(shell-bolt-spec {"1" :shuffle "2" ["id"]}
                 "python"
                 "mybolt.py"
                 ["outfield1" "outfield2"]
                 :p 25)
```

输出声明的语法在下面的defbolt部分中有更详细的描述。有关Storm的工作原理的详细信息，请参阅[使用Storm的非JVM语言](Using-non-JVM-languages-with-Storm.html) 。

### defbolt

`defbolt` 用于在Clojure中定义bolts。这里对bolts有一个限制，那就是他必须是可序列化的，这就是为什么你不能仅仅具体化`IRichBolt`来实现一个bolts（closures不可序列化）。 `defbolt` 在这个限制的基础上为定义bolts提供了一种更好的语法，而不仅仅是实现一个Java接口的。

在最充分的表现形势下，`defbolt`支持参数化bolts，并在bolts执行期间保持关闭状态。它还提供了用于定义不需要额外功能的bolts的快捷方式。 `defbolt`的签名如下所示：

(defbolt _name_ _output-declaration_ *_option-map_ & _impl_)

省略option map(选项映射)相当于具有{：prepare false}的option map(选项映射)。

#### Simple bolts (简单 bolts)

我们从最简单的defbolt形式开始吧。这是一个将包含句子的元组分割成每个单词的元组的示例bolt：

```clojure
(defbolt split-sentence ["word"] [tuple collector]
  (let [words (.split (.getString tuple 0) " ")]
    (doseq [w words]
      (emit-bolt! collector [w] :anchor tuple))
    (ack! collector tuple)
    ))
```

Since the option map is omitted, this is a non-prepared bolt. The DSL simply expects an implementation for the `execute` method of `IRichBolt`. The implementation takes two parameters, the tuple and the `OutputCollector`, and is followed by the body of the `execute` function. The DSL automatically type-hints the parameters for you so you don't need to worry about reflection if you use Java interop.
(由于感觉有不准确的地方，先留着方便优化。)
由于省略了option map(选项映射)，这是一个non-prepared bolt。 DSL只是期望执行一个IRichBolt的`execute`方法。该实现需要两个参数，即tuple(元组)和`OutputCollector`，后面是`execute`函数的正文。 DSL会为你自动提示参数，所以如果您使用Java交互，不需要担心反射问题。


This implementation binds `split-sentence` to an actual `IRichBolt` object that you can use in topologies, like so:
此实现将`split-sentence`绑定到一个可用于topologies实现的`IRichBolt`对象，如下所示：
```clojure
(bolt-spec {"1" :shuffle}
           split-sentence
           :p 5)
```


#### Parameterized bolts  (参数化 bolts)

有时候你想用其他参数来参数化你的bolts。例如，假设你想有一个可以接收到每个输入字符串后缀的bolts，并且希望在运行时设置该后缀。你可以在defbolt中通过在option map(选项映射)中包含：`:params`选项来执行此操作，如下所示：

```clojure
(defbolt suffix-appender ["word"] {:params [suffix]}
  [tuple collector]
  (emit-bolt! collector [(str (.getString tuple 0) suffix)] :anchor tuple)
  )
```

与前面的示例不同，`suffix-appender`将绑定到一个返回`IRichBolt`而不是直接作为`IRichBolt`对象的函数。这是通过在其option map(选项映射)中指定`:params`引起的。因此，在topology中使用`suffix-appender`，您可以执行以下操作：

```clojure
(bolt-spec {"1" :shuffle}
           (suffix-appender "-suffix")
           :p 10)
```

#### Prepared bolts (准备 bolts)


要做更复杂的bolts，如加入和流聚合的bolt，bolt需要存储状态。您可以通过在option map(选项映射)中创建一个通过包含`{:prepare true}`指定的prepared bolt 来实现此目的。例如，思考下这个实现单词计数的bolt：

```clojure
(defbolt word-count ["word" "count"] {:prepare true}
  [conf context collector]
  (let [counts (atom {})]
    (bolt
     (execute [tuple]
       (let [word (.getString tuple 0)]
         (swap! counts (partial merge-with +) {word 1})
         (emit-bolt! collector [word (@counts word)] :anchor tuple)
         (ack! collector tuple)
         )))))
```

prepared bolt的实现是通过一个函数 ，它将topology的配置“TopologyContext”和“OutputCollector”作为输入，并返回“IBolt”接口的一个实现。此设计允许您围绕`execute`和`cleanup`的实现时进行闭包。

在这个例子中，单词计数存储在一个名为`counts`的映射的闭包中。 `bolt`宏用于创建`IBolt`实现。 `bolt`宏是一种比简化实现界面更简洁的方法，它会自动提示所有的方法参数。该bolt实现了更新映射中的计数并发出新的单词计数的执行方法。

请注意， prepared bolts 中的`execute`方法只能作为元组的输入，因为`OutputCollector`已经在函数的闭包中（对于简单的bolts，collector是`execute`函数的第二个参数）。

Prepared bolts 可以像 simple bolts 一样进行参数化。

#### Output declarations (输出声明)

Clojure DSL具有用于bolt输出的简明语法。声明输出的最通用的方法就是从stream id到stream spec的映射。例如：

```clojure
{"1" ["field1" "field2"]
 "2" (direct-stream ["f1" "f2" "f3"])
 "3" ["f1"]}
```

stream id 是一个字符串，而stream spec(流规范)是个字段的向量或由`direct-stream`包装的字段的向量。 `direct stream`将流标记为direct stream（有关直接流的更多详细信息，请参阅[Concepts](Concepts.html) 和[Direct groupings](空的。。)）。


如果bolt只有一个输出流，您可以使用向量而不用输出声明的映射来定义bolt的默认流。例如：

```clojure
["word" "count"]
```
This declares the output of the bolt as the fields ["word" "count"] on the default stream id.
这段bolt输出的声明 为默认 stream id 上的字段[“word” “count”]。
#### Emitting, acking, and failing  (发射，确认和失败)

Rather than use the Java methods on `OutputCollector` directly, the DSL provides a nicer set of functions for using `OutputCollector`: `emit-bolt!`, `emit-direct-bolt!`, `ack!`, and `fail!`.
DSL可以使用`OutputCollector`：`emit-bolt！`，`emit-direct-bolt！`，`ack！`和`fail ！`，而不是直接在`OutputCollector`上使用Java方法.

1. `emit-bolt!`: takes as parameters the `OutputCollector`, the values to emit (a Clojure sequence), and keyword arguments for `:anchor` and `:stream`. `:anchor` can be a single tuple or a list of tuples, and `:stream` is the id of the stream to emit to. Omitting the keyword arguments emits an unanchored tuple to the default stream.
2. `emit-direct-bolt!`: takes as parameters the `OutputCollector`, the task id to send the tuple to, the values to emit, and keyword arguments for `:anchor` and `:stream`. This function can only emit to streams declared as direct streams.
2. `ack!`: takes as parameters the `OutputCollector` and the tuple to ack.
3. `fail!`: takes as parameters the `OutputCollector` and the tuple to fail.

See [Guaranteeing message processing](Guaranteeing-message-processing.html) for more info on acking and anchoring.

### defspout

`defspout` is used for defining spouts in Clojure. Like bolts, spouts must be serializable so you can't just reify `IRichSpout` to do spout implementations in Clojure. `defspout` works around this restriction and provides a nicer syntax for defining spouts than just implementing a Java interface.

The signature for `defspout` looks like the following:

(defspout _name_ _output-declaration_ *_option-map_ & _impl_)

If you leave out the option map, it defaults to {:prepare true}. The output declaration for `defspout` has the same syntax as `defbolt`.

Here's an example `defspout` implementation from [storm-starter]({{page.git-blob-base}}/examples/storm-starter/src/clj/org/apache/storm/starter/clj/word_count.clj):

```clojure
(defspout sentence-spout ["sentence"]
  [conf context collector]
  (let [sentences ["a little brown dog"
                   "the man petted the dog"
                   "four score and seven years ago"
                   "an apple a day keeps the doctor away"]]
    (spout
     (nextTuple []
       (Thread/sleep 100)
       (emit-spout! collector [(rand-nth sentences)])         
       )
     (ack [id]
        ;; You only need to define this method for reliable spouts
        ;; (such as one that reads off of a queue like Kestrel)
        ;; This is an unreliable spout, so it does nothing here
        ))))
```

The implementation takes in as input the topology config, `TopologyContext`, and `SpoutOutputCollector`. The implementation returns an `ISpout` object. Here, the `nextTuple` function emits a random sentence from `sentences`. 

This spout isn't reliable, so the `ack` and `fail` methods will never be called. A reliable spout will add a message id when emitting tuples, and then `ack` or `fail` will be called when the tuple is completed or failed respectively. See [Guaranteeing message processing](Guaranteeing-message-processing.html) for more info on how reliability works within Storm.

`emit-spout!` takes in as parameters the `SpoutOutputCollector` and the new tuple to be emitted, and accepts as keyword arguments `:stream` and `:id`. `:stream` specifies the stream to emit to, and `:id` specifies a message id for the tuple (used in the `ack` and `fail` callbacks). Omitting these arguments emits an unanchored tuple to the default output stream.

There is also a `emit-direct-spout!` function that emits a tuple to a direct stream and takes an additional argument as the second parameter of the task id to send the tuple to.

Spouts can be parameterized just like bolts, in which case the symbol is bound to a function returning `IRichSpout` instead of the `IRichSpout` itself. You can also declare an unprepared spout which only defines the `nextTuple` method. Here is an example of an unprepared spout that emits random sentences parameterized at runtime:

```clojure
(defspout sentence-spout-parameterized ["word"] {:params [sentences] :prepare false}
  [collector]
  (Thread/sleep 500)
  (emit-spout! collector [(rand-nth sentences)]))
```

The following example illustrates how to use this spout in a `spout-spec`:

```clojure
(spout-spec (sentence-spout-parameterized
                   ["the cat jumped over the door"
                    "greetings from a faraway land"])
            :p 2)
```

### Running topologies in local mode or on a cluster

That's all there is to the Clojure DSL. To submit topologies in remote mode or local mode, just use the `StormSubmitter` or `LocalCluster` classes just like you would from Java.

To create topology configs, it's easiest to use the `org.apache.storm.config` namespace which defines constants for all of the possible configs. The constants are the same as the static constants in the `Config` class, except with dashes instead of underscores. For example, here's a topology config that sets the number of workers to 15 and configures the topology in debug mode:

```clojure
{TOPOLOGY-DEBUG true
 TOPOLOGY-WORKERS 15}
```

### Testing topologies

[This blog post](http://www.pixelmachine.org/2011/12/17/Testing-Storm-Topologies.html) and its [follow-up](http://www.pixelmachine.org/2011/12/21/Testing-Storm-Topologies-Part-2.html) give a good overview of Storm's powerful built-in facilities for testing topologies in Clojure.
