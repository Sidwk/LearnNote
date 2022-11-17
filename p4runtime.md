# p4runtime

#### 简单介绍

在`p4runtime`实验中分几个部分，分别为

库的导入，交换机的创建，建立节点与交换机的连接，设置主节点，将p4代码和配置信息安装到交换机中，操作列表。

他的整个的结构是由`mycontroller.py`，`switch.py`，`helper.py`组成，是一个比较复杂的结构。

其中`mycontroller.py` 是整个控制平面的组成代码。`switch.py` 是下流表等操作的函数集，即关于交换机的函数基本都在这里，不光是有流表的，还有事像定义交换机的操作等。然后最后的是`helper.py` ，其主要着着重于定义关于流表的操作，主要还是集中于三个方面：

**1.解析p4代码里编写的关于`match-action` 然后把它变成交换机可以看懂的配置。

**2.创建可以使用的流表数据类型`table_entry` 。

**3.进行读取流表里的具体的数据，表项id，表项名字等内容。明白这三个文件的分工之后，下面就是组成控制平面的操作。

**整个p4的逻辑就是首先用p4编译器把p4代码编译成p4info文件，然后**`helper.py` **读取这些info文件里的配置，然后把这些信息导入交换机中，后面再根据配置有哪些表项，把这些表项导入交换机的表中，之后整个系统完成，开始运行。**

## main函数

```python
if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='P4Runtime Controller')
    parser.add_argument('--p4info', help='p4info proto in text format from p4c',
                        type=str, action="store", required=False,
                        default='./build/advanced_tunnel.p4.p4info.txt')
    parser.add_argument('--bmv2-json', help='BMv2 JSON file from p4c',
                        type=str, action="store", required=False,
                        default='./build/advanced_tunnel.json')
    args = parser.parse_args()

    if not os.path.exists(args.p4info):
        parser.print_help()
        print("\np4info file not found: %s\nHave you run 'make'?" % args.p4info)
        parser.exit(1)
    if not os.path.exists(args.bmv2_json):
        parser.print_help()
        print("\nBMv2 JSON file not found: %s\nHave you run 'make'?" % args.bmv2_json)
        parser.exit(1)
    main(args.p4info, args.bmv2_json)
```

main函数中前面的parser为添加环境和参数，主要加入了`p4info`和`BMv2 JSON`文件，然后检查是否添加进去，在之后输入参数，进入main函数之中。

## 库的导入

在`p4runtime`中主要使用了`grpc`的库和`p4runtime`自己书写得代码。

其中`grpc`库主要是用于交换机的创建工作。

而对于`p4runtime`的库来说，其存在于`util`文件夹的`p4runtime_lib`里，先对它加入环境之中，在进行导入。

```python
import grpc

sys.path.append(
    os.path.join(os.path.dirname(os.path.abspath(__file__)),
                 '../../utils/'))
import p4runtime_lib.bmv2
import p4runtime_lib.helper
from p4runtime_lib.error_utils import printGrpcError
from p4runtime_lib.switch import ShutdownAllSwitchConnections
```

导入的库中`p4runtime_lib.bmv2`用于帮助建立与交换机的连接工作。

 `p4runtime_lib.helper`会读取并解析编译生成的`xxx.p4.p4info.txt`代码，之后把p4代码和配置信息安装到虚拟机中。

`printGrpcError`集成在`p4runtime_lib.error_utils`中，作用为当出错时，可以使用的错误处理方法，在下面的主要函数`main`中就是全程写在了`try-except`的里面

## 建立与交换机的关系

在`def main`函数中可以看到整个框架都在`try-except`里面

```python
s1 = p4runtime_lib.bmv2.Bmv2SwitchConnection(
            name='s1',
            address='127.0.0.1:50051',
            device_id=0,
            proto_dump_file='logs/s1-p4runtime-requests.txt')
        s2 = p4runtime_lib.bmv2.Bmv2SwitchConnection(
            name='s2',
            address='127.0.0.1:50052',
            device_id=1,
            proto_dump_file='logs/s2-p4runtime-requests.txt')
```

首先创建两个交换机s1和s2，添加两个交换机的ip地址和编号还有其对应的配置信息。

而这一切信息都是通过`Bmv2SwitchConnection`类方法进行实现的，关于`Bmv2SwitchConnection`类方法中

```python
def buildDeviceConfig(bmv2_json_file_path=None):
    "Builds the device config for BMv2"
    device_config = p4config_pb2.P4DeviceConfig()
    device_config.reassign = True
    with open(bmv2_json_file_path) as f:
        device_config.device_data = f.read().encode('utf-8')
    return device_config

class Bmv2SwitchConnection(SwitchConnection):
    def buildDeviceConfig(self, **kwargs):
        return buildDeviceConfig(**kwargs)
```

可以看到其继承于`SwitchConnection`方法，并重写了其中的`buildDeviceConfig`方法。这个方法在于直接通过前面导入的JSON文件把配置输入进交换机之中。其主要工作都在其继承的`SwitchConnection`中定义的方法。后面打开`SwitchConnection`，我们可以看见参数通过`object`传进来。

```python
def __init__(self, name=None, address='127.0.0.1:50051', device_id=0,
                 proto_dump_file=None):
        self.name = name
        self.address = address
        self.device_id = device_id
        self.p4info = None
        self.channel = grpc.insecure_channel(self.address)
        if proto_dump_file is not None:
            interceptor = GrpcRequestLogger(proto_dump_file)
            self.channel = grpc.intercept_channel(self.channel, interceptor)
        self.client_stub = p4runtime_pb2_grpc.P4RuntimeStub(self.channel)
        self.requests_stream = IterableQueue()
        self.stream_msg_resp = self.client_stub.StreamChannel(iter(self.requests_stream))
        self.proto_dump_file = proto_dump_file
        connections.append(self)
```

在这里面前面的`self`用于创建参数，后面的使用`grpc`函数创建交换机。又在`grpc`创建的函数中`self.client_stub.StreamChannel(iter(self.requests_stream))`最为重要，其作用为初始化交换机和控制器的连接，探测交换机的存活(心跳检测)，发送/接收包。

到此为止，控制平面就与交换机成功的建立连接了。

**其实整个就是调用了已经写好的已经封装好的`Bmv2SwitchConnection`类，配置好相应的交换机，就可以直接进行生成。**

## 设置主节点

p4交换机支持多集群的控制器，同时可以给给控制器组设置角色，每个角色有一个主控制器。

我感觉这里对每个交换机调用`MasterArbitrationUpdate()`函数，是相当于设立了两个角色，然后把每个角色设置为了该角色的主控制器。即即便只有一个控制器也得手动设置主控制器。

```python
        # Send master arbitration update message to establish this controller as
        # master (required by P4Runtime before performing any other write operation)
        s1.MasterArbitrationUpdate()
        s2.MasterArbitrationUpdate()

def MasterArbitrationUpdate(self, dry_run=False, **kwargs):
        request = p4runtime_pb2.StreamMessageRequest()
        request.arbitration.device_id = self.device_id
        request.arbitration.election_id.high = 0
        request.arbitration.election_id.low = 1

        if dry_run:
            print("P4Runtime MasterArbitrationUpdate: ", request)
        else:
            self.requests_stream.put(request)
            for item in self.stream_msg_resp:
                return item # just one
```

## p4代码和配置信息安装到交换机中

首先在main函数的开头处，有定义`P4InfoHelper`类

```python
p4info_helper = p4runtime_lib.helper.P4InfoHelper(p4info_file_path)
```

其作用为读取并解析p4编译器编译生成的`xxx.p4.p4info.txt`文件，在该文件中，存放着关于每个交换机模型，表id，名称与字段等信息。为基础的交换机里的初始配置信息。

然后在鞋面的源码中通过`SetForwardingPipelineConfig`()函数输入进定义好的s1和s2之中。

```python
# Install the P4 program on the switches
        s1.SetForwardingPipelineConfig(p4info=p4info_helper.p4info,
                                       bmv2_json_file_path=bmv2_file_path)
        print("Installed P4 Program using SetForwardingPipelineConfig on s1")
        s2.SetForwardingPipelineConfig(p4info=p4info_helper.p4info,
                                       bmv2_json_file_path=bmv2_file_path)
        print("Installed P4 Program using SetForwardingPipelineConfig on s2")
```

然后查看关于`SetForwardingPipelineConfig()`方法的源码。

```python
def SetForwardingPipelineConfig(self, p4info, dry_run=False, **kwargs):
        device_config = self.buildDeviceConfig(**kwargs)
        request = p4runtime_pb2.SetForwardingPipelineConfigRequest()
        request.election_id.low = 1
        request.device_id = self.device_id
        config = request.config

        config.p4info.CopyFrom(p4info)
        config.p4_device_config = device_config.SerializeToString()

        request.action = p4runtime_pb2.SetForwardingPipelineConfigRequest.VERIFY_AND_COMMIT
        if dry_run:
            print("P4Runtime SetForwardingPipelineConfig:", request)
        else:
            self.client_stub.SetForwardingPipelineConfig(request)
```

没看懂，不过理解了它的用法也就是使用在编译的时候生成的配置文件导入对应的交换机之中，纸样完成的配置，而具体的`switch,py`里的函数，等到需要的时候再看看吧。

## 操作表项

#### 写流表

```python
# Write the rules that tunnel traffic from h1 to h2
        writeTunnelRules(p4info_helper, ingress_sw=s1, egress_sw=s2, tunnel_id=100,
                         dst_eth_addr="08:00:00:00:02:22", dst_ip_addr="10.0.2.2")

        # Write the rules that tunnel traffic from h2 to h1
        writeTunnelRules(p4info_helper, ingress_sw=s2, egress_sw=s1, tunnel_id=200,
                         dst_eth_addr="08:00:00:00:01:11", dst_ip_addr="10.0.1.1")

        # TODO Uncomment the following two lines to read table entries from s1 and s2
        readTableRules(p4info_helper, s1)
        readTableRules(p4info_helper, s2)
```

这部分是关于下流表的重点，其中`writeTunnelRules()`是将编写的表项的规则输入到交换机中，当传入参数`ingress`，`engress`等时，会根据参数把`writeTunnelRules()`里面的规则全部导入，有几个规则导入几个表项。

这个`p4runtime`实现代码里有三个规则，所以就导入了三个表项。我们知道在p4编程的`match-action`中，一个match有好几个action，在这里也是如此，但是需要对每个action都分开书写，分开的定义规则，也在交换机中生成不同的表项。

定义好需要下发的流表之后使用` WriteTableEntry()`函数将流表下发，`WriteTableEntry()`的定义在`switch.py`文件中为

```python
def WriteTableEntry(self, table_entry, dry_run=False):
        request = p4runtime_pb2.WriteRequest()
        request.device_id = self.device_id
        request.election_id.low = 1
        update = request.updates.add()
        if table_entry.is_default_action:
            update.type = p4runtime_pb2.Update.MODIFY
        else:
            update.type = p4runtime_pb2.Update.INSERT
        update.entity.table_entry.CopyFrom(table_entry)
        if dry_run:
            print("P4Runtime Write:", request)
        else:
            self.client_stub.Write(request)
```

这个就是`update`方法对表项进行更新和添加。这里的官方封装的用法是一个表项调用一次`WriteTableEntry(table_entry)`但是事实上底层的接口可以一次性下发多个表项。

#### 读流表

这一个没什么好说的，就是一个读取一个交换机内的所有的流表项，然后打印出来。具体实施上，也是调用了`switch.py`里面的`get_tables_name`，`get_match_field_name`等函数。只要看懂如何使用就可以了。



**PS：**现在又有了**新的问题**，在这里可以看到在数据平面也就是p4的代码里面我们定义表格是，也就是定义有哪些表，然后在控制平面也就是python里面写表项。但那是不是说控制平面不光是可以下发流表也可以更改表的吗？所以到底可不可以在控制平面就对交换机定义它的有哪些表。还有就是我们可不可以再整个系统运行的时候对表项进行就修改的操作？



