# Monitor multi applications

本文我们将修改在[上文](Error_handling.md)中开发的插件，赋予它监控多个应用的能力（或者我们称之为 进程组 process group）。这个插件将同时监控2个 demo。我们将在不同的端口上运行demo：

```shell
python -m plugin_sdk.demo_app  -p 8991
python -m plugin_sdk.demo_app_auth -p 8992
```

首先，我们先看一下完整的插件源码；然后，我们将介绍需要执行的修改。这是插件源代码：

```python
'''
Demo plugin which shows how to monitor multiple process groups with single plugin.
'''
import requests
import requests.exceptions
import json
import logging
from ruxit.api.base_plugin import BasePlugin
from ruxit.api.snapshot import parse_port_bindings
from ruxit.api.exceptions import AuthException, ConfigException
from ruxit.api.snapshot import pgi_name

class DemoPluginMulti(BasePlugin):
    '''
    Class implementing plugin which shows how to monitor multiple process groups with single plugin.
    The group is to be monitored if its name starts with predefined string.
    '''
    _APPL_PREFIX='plugin_sdk.demo_app'

    def query(self, **kwargs):
        '''
        Scan process snapshot for groups with name starting with given prefix, and
        for all matching this prefix find ports on which they are listening. Then use this port
        for building URL to query with http. The result is json with two variables which are
        put into results with entity id of given process group.
        '''
        user = self.config["user"]
        password = self.config["password"]
        # search process snapshot using criteria defined by lambda expression
        pgi_list = self.find_all_process_groups( lambda entry: entry.group_name.startswith(self._APPL_PREFIX))
        for pgi in pgi_list:
            pgi_id = pgi.group_instance_id
            self.logger.info( "Demo pgid=%x application=%s" % (pgi_id,pgi.group_name ))
            port = None
            for process in pgi.processes:
               port = process.properties.get("ListeningPorts", None)
               break
            if port is None:
                raise ValueError( "no port definition for process group with id=%d" % pgi_id )
            # build URL for quering
            url = "http://127.0.0.1:" + port
            self.logger.info( "using url %s for pgid=%x app=%s" % (url,pgi_id,pgi.group_name ))

            try:
                # query URL for results
                response = requests.get(url, auth=(user, password))
                if response.status_code == 401:
                    raise AuthException(response)
                stats = response.json()
            except requests.exceptions.ConnectTimeout as ex:
                raise ConfigException('Timeout on connecting with "%s"' % url) from ex
            except requests.exceptions.RequestException as ex:
                raise ConfigException('Unable to connect to "%s"' % url) from ex
            except json.JSONDecodeError as ex:
                raise ConfigException('Server response from %s is not json' % url) from ex

            # save received results
            self.results_builder.absolute(key='random', value=stats['random'], entity_id=pgi_id)
            self.results_builder.relative(key='counter', value=stats['counter'], entity_id=pgi_id)
```

相应的 plugin.json 基本一样，需要修改的就是version,version,source.package 

```json
{
  "version": "1.6",
  "requiredAgentVersion": "1.90",
  "name": "custom.python.demo_plugin_multi",
  "type": "python",
  "entity": "PROCESS_GROUP_INSTANCE",
  "metricGroup": "demo_metrics.multi",
  "technologies": ["PYTHON"],
  "source": {
    "package": "demo_plugin_multi",
    "className": "DemoPluginMulti",
    "install_requires": ["requests>=2.6.0"],
    "activation": "Singleton"
  },
  "metrics": [
    {
      "timeseries": {
        "key": "random",
        "unit": "Count",
        "dimensions": [],
	"displayname": "Random Value"

      }
    },
    {
      "timeseries": {
        "key": "counter",
        "unit": "Count",
        "dimensions": [],
	"displayname":"Counter Value"

      }
    }
  ],
  "configUI": {
    "displayName": "OneAgent Demo Multi Extension"
  },
  "properties": [
    {
      "key": "user",
      "type": "String"
    },
    {
      "key": "password",
      "type": "Password"
    }
  ]
}
```

请参阅 [plugin.json reference](https://dynatrace.github.io/plugin-sdk/api/plugin_json_apidoc.html) 以获取有关属性和configUI元素的更多信息，包括更详细的字段名称，字段顺序等。

这个插件里，我们将监控所有被 Agent 探测到的，进程名以 `plugin_sdk.demo_app` 开头的所有应用。当我们启动这些demo时，web UI 将展示相应的 进程信息：

![../_images/demo_03_applications.png](https://dynatrace.github.io/plugin-sdk/_images/demo_03_applications.png)

这意味着我们要监控  `plugin_sdk.demo_app` ， `plugin_sdk.demo_app_auth` 。以下是在当前进程快照中搜索的代码片段：

```python
        for pgi in pgi_list:
```

在我们的例子中，快照如下所示：

```json
[
ProcessSnapshotEntry(
  group_id=12668946693571282165, 
  node_id=0, 
  group_instance_id=18165771568562771184, 
  process_type=29, 
  group_name='plugin_sdk.demo_app_auth', 
  processes=[
    ProcessInfo(
      pid=10426, 
      process_name='python3.5', 
      properties={
        'CmdLine': '-m plugin_sdk.demo_app_auth -p 8992', 
        'WorkDir': '/home/jj/ruxit/trunk/core/product/agent', 
        'PortBindings': '127.0.0.1_8992', 
        'ListeningPorts': '8992'
      }
    )
  ], 
  properties={}
), 
ProcessSnapshotEntry(
  group_id=17388020258730275021, 
  node_id=0, 
  group_instance_id=5122137151859959739, 
  process_type=29, 
  group_name='plugin_sdk.demo_app', 
  processes=[
    ProcessInfo(
      pid=6586, 
      process_name='python3.5', 
      properties={
        'CmdLine': '-m plugin_sdk.demo_app -p 8991', 
        'WorkDir': '/home/jj/ruxit/trunk/core/product/agent', 
        'PortBindings': '127.0.0.1_8991', 
        'ListeningPorts': '8991'
      }
    )
  ], 
  properties={}
)
]
```

然后，根据找到的进程，我们从快照中读取 应用正在监听的端口号。应用打开的监听端口信息由 Agent 收集，并放在 进程快照 中 供 插件访问。我们使用它来创建一个URL，该URL将通过HTTP查询感兴趣的计数器：

```python
               port = process.properties.get("ListeningPorts", None)
               break
            if port is None:
                raise ValueError( "no port definition for process group with id=%d" % pgi_id )
            # build URL for quering
            url = "http://127.0.0.1:" + port
            self.logger.info( "using url %s for pgid=%x app=%s" % (url,pgi_id,pgi.group_name ))
```

想要更深入的学习，请查阅以下内容

- [More information on plugin.json files](https://dynatrace.github.io/plugin-sdk/api/plugin_json_apidoc.html).
- [Plugin lifecycle](https://dynatrace.github.io/plugin-sdk/plugin_lifecycle/index.html).
- [Technical documentation](https://dynatrace.github.io/plugin-sdk/apidoc.html).

