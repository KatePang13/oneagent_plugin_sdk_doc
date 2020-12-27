# Process snapshot

`embedded_api`  模块只在 Agent 内嵌的Python解释器中可用。它会暴露一些数据结构，这些数据在开发插件的时候很有用，特别是 process snapshot 。

请注意，除了这里罗列的还有其他方法可用于process snapshot。

## class `embeded_api.ProcessSnapshot`

包含关于进程当前状态的信息，这是由Agent感知得来的。这个类主要的意图是提供这些信息，以便在上报插件采集数据是，可以使用其中的 entities(各项数据) 和其相关的 IDs(各种粒度的ID标识)。

**host_id**

主机ID  - `int` 。

**entries**

主机上探测到的进程组。一个可迭代的 `ProcessSnapshotEntry` 实例列表

**entries**

主机上探测到的容器信息。一个可迭代的 `ContainerInfo`  实例列表



## class `embedded_api.ProcessSnapshotEntry`

包含一个单进程组的信息。里面是这个 进程组 的 entity，这些 entity 应该在上报相应采集数据时使用。

**group_id**

这个 snapshot entry 的进程组ID。进程组时一个 进程集合，这些进程 在多个主机上执行相同的功能。比如，你有一个服务器集群，每个服务器运行相同的进程以支持多个主机。

**node_id**

这个 snapshot entry 的 节点ID。用于区分在同一主机上运行的多个进程组实例。它是根据一组特定于过程类型的参数计算的，尽管通常值为0。

**group_instance_id**

进程组实例ID 是在 指定主机上执行相同任务的一组进程的ID。进程组实例ID是一个唯一标识，应该在上报相关数据时使用。

**process_type**

进程组类型 （`int`）。 请参考 [Plugin.json reference](https://dynatrace.github.io/plugin-sdk/api/plugin_json_apidoc.html) ，查看所有可用的进程类型。

**group_name**

进程组名， 以特定于每种 **process_type** 的方式进行分配。

**processes**

属于该进程组的所有进程。一个可迭代的 `ProcessInfo` 实例列表

**properties**

进程的其他信息，以字典的形式提供。可能的键包括：

- `mssql_instance_name`    MSSQL实例名，仅分配给MSSQL类型的进程组。



## *class* `embedded_api.ProcessInfo`

**pid**

进程 PID

**process_name**

进程名

**properties**

进程额外信息，以字典形式提供。

可能的值：

- `WorkDir`   进程工作目录
- `CmdLine`   进程命令行
- `PortBindings`  TCP 端口绑定数据。 使用  [`snapshot.parse_port_bindings()`](https://dynatrace.github.io/plugin-sdk/_apidoc/ruxit.api.html#ruxit.api.snapshot.parse_port_bindings)  来接收解析到数据。这个解析数据时一个2元组的列表，第一个字段是IP地址(`string`)，第二个字段是端口号(`int`) 。 `0.0.0.0` 地址会被转换成本机地址 `127.0.0.1` 
- `ListeningPorts`  进程正在监听的端口，以空格分隔的字符串形式提供。不建议使用，请使用 `PortBindings`。

