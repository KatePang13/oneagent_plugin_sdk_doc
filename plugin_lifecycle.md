# Plugin life cycle

虽然 Agent 会自动控制所有插件的行为（例如，确定是否应运行特定插件，重新启动出错的插件以及收集所需的配置详细信息等），在某些情况下，更深入地了解Agent 的内部机制也是有好处的。比如，当一个新开发的插件没有启动，旧版本的插件在运行时，遇到这类的一场情况该如何处理呢。

plugin的生命周期主要分成4个阶段：

- Loading
- Activation
- Running
- Closing



## Plugin Loading

插件加载过程在 Agent 启动时进行。这个过程会评估插件是否与 Agent 和其他插件的当前版本兼容。

当加载完成时，插件要等到满足特定条件后才会被激活。（插件激活将在下一节介绍）

Agent 会加载 3个位置的插件：

- `plugin_deployment` 目录，在 Agent 安装的根目录下 

- Agent 从 Server 下载的插件
- 与 Agent  安装程序一起分发的插件

每个插件需要被校验，校验与其他已加载插件的兼容性。插件间需要检查是否存在依赖库版本冲突的情况（比如，plugin_a 需要依赖版本 == 2.9.1, 而 plugin_b需要依赖版本 <= 2.8.3 ）。造成冲突的插件将不会被加载，加载顺序由位置决定(即前面提到的三个位置)；如果在同一个位置，则由插件名称的字节序决定。

有关可能的插件不兼容的详细信息以及解决方法，请参阅  [Limitations](https://dynatrace.github.io/plugin-sdk/troubleshooting/limitations.html) 。

后两个位置只供内部使用，这些插件不应该手动修改。而 plugin_deployment 目录下的插件可以手动修改。



## Plugin Activation

大多数情况下，在检测到被监控进程时，就会启动相应的插件。

插件加载后，Agent 会决定是否要激活它。激活插件会触发获取插件配置的尝试，该配置存储在服务器上。 Agent 还会确认插件已启用。确认可用时，插件将被运行。

插件激活最重要的因素是 进程快照 [process snapshot](Process_snapshot.md) 。这个数据结构包含了操作系统中公认的那些重要进程的信息。如果探测到 一个 process snapshot 与 一个 plugin.json 有匹配的话 ，这个插件就变成 活跃状态 。大多数情况下，是通过 探测到指定类型的进程来得到的。因此，如果进程快照中的数据消失，则该插件将变成 不活跃状态。

当前提供3种类型的激活：

- 检测到单个触发过程时，运行单个插件实例
- 检测到被监控的进程组是，运行与进程组中进程个数相同的插件实例
- 持久运行插件

说了这么多，现在我们来看看具体的激活原理：

### 为所有被监视的进程激活一个插件实例

在大多数情况下(就像我们之前学习的第一个自定义插件)，最好的方式是 当一个指定类型的进程被探测到时，就创建一个插件实例。这种方式需要如下的 `plugin.json` 声明：

```json
{
  "entity": "PROCESS_GROUP_INSTANCE",
  "technologies": [ "PYTHON" ],
  "source": {
    "activation": "Singleton"
  },
}
```

根据 plugin.json中的上诉声明，快照中任意 python 进程组的实例都会激活这个插件。就像下面的这个 snapshot:

```json
 ProcessSnapshot(
     host_id=11711730974707348096,
     entries=[
         ProcessSnapshotEntry(
             group_id=9849894537414073908,
             node_id=0,
             group_instance_id=11914897446187082808,
             group_name='plugin_sdk.demo_app',
             processes=[
                 ProcessInfo(
                     pid=1541,
                     process_name='python3.5',
                     properties={
                         'CmdLine': '-m plugin_sdk.demo_app',
                         'WorkDir': '/home/demo',
                         'ListeningPorts': '8090'
                     })
                 ],
             properties={"Technologies": "PYTHON"}
         ),
         ProcessSnapshotEntry(
             group_id=483552688914919364,
             node_id=0,
             group_instance_id=11834758190185815364,
             group_name='puppet',
             processes=[
                 ProcessInfo(
                     pid=1257,
                     process_name='puppet',
                     properties={'CmdLine': '/usr/bin/puppet agent', 'WorkDir': '/'})
             ],
             properties={'Technologies': 'RUBY'}
         )
     containers=[]
 )
```

这个快照包含了2个进程组，一个是 python类型，跑的是 `plugin_sdk.demo_app`， 另一个是 ruby类型，跑的是

`puppet` 。

对于这个插件，不管创建多少个python进程，都只会创建一个插件实例。这说明如果一个进程类型是通用的（比如 Python, Java, Ruby）, 你需要深入到 快照中包含的进程，检查是否是你要监控的。

### 为给定类型的每个进程组激活一个插件

在某些情况下，最好的选择是为每个检测到的进程组实例，就为它单独创建一个插件实例。

- 这种方式，你不需要考虑深入到进程快照内来确保你想监控的进程是在运行的。
- 另一方面，如果你的插件需要配置，而同时有多个进程组实例在运行，你不能使用 Server来提供配置(因为这样，每个插件的配置都会是一样的)。

MMSQL 插件是这种方式的一个合适的场景，这个插件不需要额外的配置，并且其plugin.json文件仅作了简单声明：

```json
{
  "entity": "PROCESS_GROUP_INSTANCE",
  "technologies": [ "MSSQL" ]
}
```

相应的进程快照如下所示：

```json
 ProcessSnapshot(host_id=16649240629743570171, entries=[
     ProcessSnapshotEntry(
         group_id=4337044249244370985,
         node_id=0,
         group_instance_id=9687064182437432279,
         group_name='MSSQL10_50.NAMED_ID',
         processes=[
             ProcessInfo(
                 pid=26988,
                 process_name='sqlservr.exe',
                 properties={'CmdLine': '-sNAMED_INSTANCE_01', 'WorkDir': 'C:\\Windows\\system32\\'})
             ],
             properties={'Technologies': 'MSSQL', 'mssql_instance_name': 'NAMED_INSTANCE_01'}),
     ProcessSnapshotEntry(
         group_id=12107707763631947228,
         node_id=0,
         group_instance_id=10160066155805379574,
         group_name='MSSQL10.SQLEXPRESS',
         processes=[
             ProcessInfo(
                 pid=36632,
                 process_name='sqlservr.exe',
                 properties={'CmdLine': '-sSQLEXPRESS', 'WorkDir': 'C:\\Windows\\system32\\'})
             ],
         properties={'Technologies': 'MSSQL', 'mssql_instance_name': 'SQLEXPRESS'})
     containers=[]
 )
```

这种情况下将创建2个插件实例，因为存在2个进程是属于Python类型的（这里指的是MSSQL实例）。

### 仅当进程名与模式匹配时才激活

有些情况下，针对每个进程类型的激活粒度太大时，你可以指定额外的规则来决定何时应该运行插件。这种方式，你的 `plugin.json` 文件应该包含这样的声明:

```json
 {
   "entity": "PROCESS_GROUP_INSTANCE",
   "technologies": [ "PYTHON" ],
   "source": {
     "activation_name_pattern": "^plugin_sdk.demo_app$"
   }
 }
```

这种情况下，只有检测到进程名匹配这种样式时，才会激活这个插件。匹配是通过python的正则表达函数 [re.search function](https://docs.python.org/3.6/library/re.html#re.search) 来实现的。这两种激活方式之间的区别在于，第一种激活方式是为每个符合匹配条件的进程创建一个插件实例；第二种激活模式最多创建1个插件实例。

### 保持插件一直处于活跃状态

最后一种激活方式是保持插件持续活跃，需要如下的配置来声明：

```json
{
  "entity": "HOST"
}
```

这种情况下，OneAgent一旦收到进程快照，就会激活该插件。但是，出于以下几个原因，不建议使用此激活类型：

- 你的插件应该去检查是否有数据需要收集。
- 你在plugin.json文件中指定的每个度量还需要声明与之关联的“entity”类型。否则，测量值将与主机相关联，并且在服务器上将不可见。 
- 你在Python代码中收集的每个度量都需要具有从流程快照中提取的实体ID。

### 激活 Tips

如果你的插件没有激活

- 检查agent日志，确定你的插件是否激活。Refer to the Auditing logs section of [troubleshooting guide](Troubleshooting.md).
- 确认被监控进程有在运行
- 确保要监视的进程是相关的（确认该进程已在UI中的相应“主机”页面上列出）。
- 使用Plugin SDK example中的  `demo_oneagent_plugin_snapshot` 插件来获取 有关所有可用于激活的发现的进程，实体ID和进程名称的信息。
  - 将插件部署到运行相应组件的机器上，就可以在插件代理日志文件中获得流程快照信息。
  - 插件激活信息还将作为 `Extension technology`显示在每个进程的 `Properties` 部分的UI上。  
  - `Python plugin activation technologies`可以用作 plugin.json 中的 `technologies` 属性，而`Python Activation_name_pattern` 可以用作plugin.json的 `source` 中的 `activation_name_pattern`



## Running plugins

一旦 active 插件 接收到所需的配置时，就可以进行工作了。大多数简单场景中，插件的 `query` 方法 每分钟执行一次；不过有一些规则需要遵循：

**Agent 会试图限制 已创建插件实例的数量**。 也就是说，只要插件正常工作（其方法不会引发任何异常），并且其配置没有更改，则所有调用都将在同一实例对象上进行。如果您想在插件中保持某些状态，可以通过重写 `initialize()` 方法轻松实现。

**当插件的配置发生更改时，先前的插件实例将被丢弃，并在新的插件实例上调用查询方法**（例如，您修改了用于连接的凭据）。在这种情况发生之前，如果您用插件覆盖了`close()`方法，则可以选择清除它。

**当插件抛出异常时，插件也会被替换**

- 如果异常来自 `ruxit.api.exceptions` ，会将其归类为 可恢复的错误。插件将快速重启几次，失败后会每小时尝试一次。
- 如果异常无法识别，并且插件在有限的重试中无法正确执行，则不再调度执行该插件。
- 成功执行插件会重置与其关联的所有崩溃计数。
- 目前，崩溃限制设置为20。
- 当一个插件实例中的`query`方法被执行时，要到这次执行完成后，才会再次被调度。如果插件收集数据的时间超过一分钟（例如2分钟），则仅会每2分钟执行一次。如果插件无限期挂起，Agent 将不会安排下一轮数据收集（尽管其他插件不会受到影响）。



**请注意，创建插件的新实例时**：

- 不会影响与您的插件关联的 `ResultsBuilder`。在停用插件之前，此过程将保持不变（例如，不再检测到受监视的进程）
- 不会重新加载插件中使用的Python模块。要重载需要重启 Agent



## Plugin Closing

插件的生命周期结束时将关闭。生命周期的最后，调用了 `close()` 方法，因此，如果插件前面获得了一些资源，它可以在此时释放它们。调用此方法的最常见原因有：

- 由于插件实例异常 或者 配置变更，  插件实例 被 新的实例 取代
- 插件被 deactivate (检测不到被监控的进程)
- Agent 关闭时，将关闭所有的插件