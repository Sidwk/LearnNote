# BMv2

BMv2（行为模型版本2）是参考p4软件交换机进行编写的。其由c++编写，将p4编译器从p4程序生成的JSON文件作为输入，并对其进行解释以实现该p4程序指定的数据包处理行为。

BMv2中还包含行为模型的多种变体的代码，例如`simple_switch`，`simple_switch_grpc`，`psa_switch`等。

**BMv2并不是为了成为工业级的软件交换机。**它旨在用作开发、测试和调试为它们编写的 P4 数据平面和控制平面软件的工具。因此，BMv2 的性能（在吞吐量和延迟方面）明显低于[Open vSwitch](https://www.openvswitch.org/)等生产级软件交换机。

![img](https://img1.sdnlab.com/wp-content/uploads/2018/10/p4tu1.jpg)

如图，我们写好xxx.p4代码，通过 **p4c** 这个 **p4 compiler** 将p4代码编译成为p4交换机可以理解的各种”机器代码”。如果目标交换机是 **bmv2** , 那么p4c将生成 .json文件。

通过安装ubuntu等一系列支持之后，构建代码，若在后面要是使用CLI，就需要安装nnpy Python包。

```shell
1. ./autogen.sh
2. ./configure
3. make
4. [sudo] make install  # if you need to install bmv2
```

进行安装测试之后，就可以开始运行代码了。

## 运行P4程序

**要在 bmv2 中运行您自己的 P4 程序，您首先需要将 P4 代码编译成 json 表示形式，供软件路由器使用。**此表示将告诉 bmv2 要初始化哪些表，如何配置解析器，...

如今有两个p4编译器可用于p4lang上的bmv2：

- [p4c](https://github.com/p4lang/p4c)包含一个 bmv2 后端，是推荐使用的编译器，因为它同时支持 P4_14 和 P4_16 程序。有关如何安装和使用 p4c 的信息。目前bmv2 p4c后端支持v1model架构，部分暂定支持PSA架构。为 v1model 编写的 P4_16 程序可以用 `simple_switch`二进制执行，而为 PSA 编写的程序可以用`psa_switch`二进制执行。

- [p4c-bm](https://github.com/p4lang/p4c-bm)是 bmv2 的遗留编译器（不再主动维护）并且只支持 P4_14 程序。

## 动态加载共享对象

一些目标（simple_switch 和 simple_switch_grpc）允许用户在运行时动态加载共享库。这是通过使用特定于目标的命令行选项来完成的`--load-modules`，该选项将以逗号分隔的共享对象列表作为参数。此功能目前仅在可用的系统上`dlopen`可用。确保动态加载程序可以看到共享对象（例如，通过`LD_LIBRARY_PATH` 在 Linux 上适当设置）。您可以在配置 bmv2 时使用`--enable-modules`/来控制此功能是否可用。`--disable-modules`默认情况下，此功能在可用时启用`dlopen`

## 与 Mininet 集成

我们将在单独的文档中提供更多信息。但是，可以使用simple_router目标立即测试 Mininet 集成。

在第一个终端中，键入以下内容：

```
- cd mininet
- sudo python 1sw_demo.py --behavioral-exe ../targets/simple_router/simple_router --json ../targets/simple_router/simple_router.json
```

然后在第二个终端：

```
- cd targets/simple_router
- ./runtime_CLI < commands.txt
```

现在交换机正在运行并且表已填充。您可以在 Mininet 中运行 `pingall`或在主机h1和h2之间使用 iperf 启动 TCP 流。

当使用simple_switch（而不是上面示例中的simple_router ）运行 P4 程序时，只需将适当的`simple_switch`二进制文件提供给 `1sw_demo.py`with` --behavioral-exe.`

## `simple_switch`

**这个目标是一个软件交换机**，运行在英特尔/AMD/等通用CPU上，可以执行大多数P4_14和P4_16程序，只对这些P4程序使用的功能有一些限制。

它实现了下面列出的P4功能。它们对应于P4_14语言规范，也对应于P4_16加上v1model架构，**其目的是为了匹配P4_14语言规范中定义的架构**。

- 计数器
- 仪表
- 注册器
- action状态
- action选择器
- 一些哈希函数和校验函数
- 伪随机数的生成
- 由数据平面产生的摘要信息，发送到控制平面
- 带有入口解析器、入口控制、带有数据包复制引擎的数据包缓冲器、出口控制和出口分离器的交换机架构。
  - 组播复制
  - 数据包的克隆/镜像
  - 重新提交数据包，从入口端返回到入口端。
  - 再循环数据包，从出口的末端回到入口的起点

**这个目标可以接受来自控制器的TCP连接，**这个连接上的控制信息的格式是由Thrift API定义的。装入该目标的P4程序在运行时可以改变，目标的可执行文件不需要重新编译，以便为该新的P4程序使用新的控制面消息，也就是说，控制面API是独立于程序的（PI）。

## `simple_switch_grpc`

该目标基于上述`simple_switch`目标。主要区别在于`simple_switch_grpc`它还可以接受来自控制器的 grpc连接，其中此连接上的控制消息格式由[P4Runtime API 规范](https://github.com/p4lang/p4runtime)定义。

`simple_switch_grpc`仍然可以使用 Thrift API 接受来自控制器的连接，但这主要用于调试目的。建议不要尝试让多个控制器控制同一目标设备，一个控制器使用 Thrift API，另一个控制器使用 P4Runtime API。很容易就会出问题。









