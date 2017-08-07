---
title: Using non JVM languages with Storm
layout: documentation
---
使用没有jvm的语言编辑storm
两个部分：创建topologies 以及 使用其他语言来实现 spouts 和bolts
用另一种语言创建topologies 是比较容易的，因为topologies 用的是thrift 的结构（链接到storm.thrift）
用另一种语言实现 spouts 和 bolts 被称为“multilang components ”或“shelling ”
以下是协议的规范：Multilang协议
thrift 结构允许你将多个组件明确定义为程序和脚本（例如，使用python编写你的bolt 的文件）
在Java中，您可以通过重写ShellBolt或ShellSpout来创建multilang组件
请注意，输出字段声明发生在thrift 结构中，所以在java中创建multilang 组件需要按照以下方式 ：
在java中声明字段，通过在shellbolt的构造函数中指定它来处理另一种语言的代码
multilang使用stdin / stdout上的json消息与子进程进行通信
storm 带有Ruby、Python和实现协议的奇特适配器。下面展示一个python的示例

python 支持emitting, anchoring, acking, and logging
“storm shell ”命令使得构建jar和上传到nimbus变得更加容易
创建jar并且上传它
使用主机/端口nimbus和jarfile id来调用你的程序


关于在非JVM语言中实现DSL的注意事项

正确的打开方式地方是src / storm.thrift。由于storm topologies 是Thrift结构，Nimbus是Thrift守护进程，您可以使用任何语言创建和提交topologies 。

当您为spouts 和bolts 创建Thrift结构体时，将在ComponentObject结构体中指定spout 或bolt 的代码：

union ComponentObject {
  1：binary serialized_java;
  2：ShellComponent shell;
  3：JavaObject java_object;
}
对于非JVM DSL，您需要使用“2”和“3”。 ShellComponent允许您指定运行该组件的脚本（例如，您的python代码）。而JavaObject允许您为组件指定本地java的spout 和bolt （Storm将使用反射来创建该spout 或bolt ）。

有一个“storm shell ”命令有助于提交topology 。它的用法是这样的：

storm shell resources/ python topology.py arg1 arg2

storm shell 会 resources/ 打成一个jar ,并上传这个jar到Nimbus ，并像下面这样调用你的topology.py脚本：

python topology.py arg1 arg2 {nimbus-host} {nimbus-port} {uploaded-jar-location}

之后你可以使用Thrift API连接到Nimbus，并提交topology ，将{uploaded-jar-location}传递到submitTopology方法。为了方便参考我在下面展示了submitTopology类的定义。 ：

void submitTopology（1：string name，2：string uploadedJarLocation，3：string jsonConf，4：StormTopology拓扑）
    throws（1：AlreadyAliveException e，2：InvalidTopologyException ite）;

