## Configuration

本节你将学习如何提供带配置的插件，为此我们将拓展之前开发的第一个插件。

这个新的插件要监控的应用与此前类似，只是这次的页面访问需要鉴权（基于HTTP基本鉴权策略）。这个应用包含在我们的SDK中，你可以直接运行

```
plugin_sdk start_demo_app_auth
```

这个应用默认监听8090端口，向使用 ruxit:ruxit 认证的用户提供stats信息。你可以使用 `plugin_sdk start_demo_app_auth -help`  来查看该应用的其他选项。 

我们先来看看这个插件的源码 和 `plugin.json`   ，之后我们将修改它们。

源码：

```python
import requests
import requests.exceptions
from ruxit.api.base_plugin import BasePlugin
from ruxit.api.snapshot import pgi_name


class DemoPlugin(BasePlugin):
    def query(self, **kwargs):
        user = self.config["user"]
        password = self.config["password"]
        pgi = self.find_single_process_group(pgi_name('plugin_sdk.demo_app_auth'))
        pgi_id = pgi.group_instance_id
        stats_url = "http://localhost:8090"

        stats = requests.get(stats_url, auth=(user, password)).json()

        self.results_builder.absolute(key='random', value=stats['random'], entity_id=pgi_id)
        self.results_builder.relative(key='counter', value=stats['counter'], entity_id=pgi_id)
```

`plugin.json`:

```json
{
  "version": "1.9",
  "name": "custom.python.demo_plugin_conf",
  "type": "python",
  "entity": "PROCESS_GROUP_INSTANCE",
  "metricGroup": "demo_metrics.conf",
  "technologies": ["PYTHON"],
  "source": {
    "package": "demo_plugin_conf",
    "className": "DemoPlugin",
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
    "displayName": "OneAgent Demo Config Extension"
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

然后我们在 `plugin.json` 为这个插件添加所需的配置：

```json
  "configUI": {
    "displayName": "OneAgent Demo Config Extension"
  },
  "properties": [
    {
      "key": "user",
      "type": "String"
    },
    {
```

每个属性需要指定 key 和 类型。

另外，你可以使用 `defaultValue` 来指定缺省值。为了可以选择在Web UI中提供配置，还需要指定希望插件显示在网页上的名称，这个也是在 `plugin.json` 上完成。看起来像这样：

```
      }
    }
  ],
```

注意， `displayName` 和 `plugin.name` 是可以不一样的。

关于 `properties` 和 `configUI`  更多内容，请参考  [plugin.json reference](https://dynatrace.github.io/plugin-sdk/api/plugin_json_apidoc.html)  ，这里会介绍更详细的字段名称，字段顺序等。

当你上传插件到 Dynatrace Cluster Node 后，就可以进行所需的配置。界面上   **Settings > Monitored technologies > Custom plugins**  ，然后选择  `OneAgent Demo Auth Extension`  插件。

展开插件配置表单，输入认证信息，然后单击保存：

![../_images/demo_02_configuration_form.png](https://dynatrace.github.io/plugin-sdk/_images/demo_02_configuration_form.png)

如果你输入正确的认证信息，将会看到插件正常工作。

![../_images/demo_02_configuration_ok.png](https://dynatrace.github.io/plugin-sdk/_images/demo_02_configuration_ok.png)

仅知道如何声明所需属性并在页面上配置是不够的，你必须在代码中处理这些属性。当然这也很简单，就像这样：

```python
password = self.config["password"]
pgi = self.find_single_process_group(pgi_name('plugin_sdk.demo_app_auth'))
pgi_id = pgi.group_instance_id
stats_url = "http://localhost:8090"
```

后面还需要

```python
self.results_builder.absolute(key='random', value=stats['random'], entity_id=pgi_id)
```

当 OneAgent 接收到 Server 传来的 插件配置时，它将其传递给config关键字，以python字典形式提供查询，其key对应于JSON文件中存在的key。从字典中提取配置信息并将其传递到请求库非常简单。

**注意：**在服务端输入配置后，您的插件才会运行。在服务端输入配置之前，将不会创建DemoPlugin对象，也不会调用查询方法。

因此，您不必担心字典中不存在的属性。不过提供的属性值可能没生效。要了解OneAgent如何处理这种情况以及如何创建有效的错误处理，请查看我们的 [错误处理指南](https://dynatrace.github.io/plugin-sdk/extending_plugin/error_handling.html)。

