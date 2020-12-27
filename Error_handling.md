# Error handling

编写 自定义 Agent 插件 和 其他的变成任务并没有太多的不同。这意味着 错误是很可能出现的。有些错误是可预测的，而另一些则不太可预测。插件开发者有2种处理错误的选择：

- 让 Agent 处理错误
- 拦截可预测的错误情况，对其进行分类，并提供自定义错误消息。

第一种方法不需要额外的工作。当某些原因插件代码抛出异常时，错误信息会被上报到Server，而插件会在短时间内被恢复。但是这里有一个失败次数限制，超过后插件将无法恢复。详情请参考 [plugin lifecycle guide](plugin_lifecycle.md)。

第二种方式需要一些额外的处理，但能让你更好地控制提供给插件用户的错误信息。假设有一种情况，你用来链接数据库服务器的库以错误码 80010 响应，这表示的是无效凭证。你是希望用户看到 80010，还是接收到更可读的消息，比如“服务器拒绝该凭证”。

为了演示第二种方法，我们将扩展我们先前在[Plugin Configuration](Configuration.md)中创建的插件。这个插件从服务器接收一些配置，并使用它通过我们的演示程序对自己进行身份验证。在本文中，我们将假定上一教程中的所有操作均按预期进行。现在，让我们讨论如何处理各种问题。

提醒一下，我们正在监视一个demo，该应用程序可以使用以下命令启动：

```
plugin_sdk start_demo_app_auth
```

这个demo监听8090端口并接受 `ruxit:ruxit` 认证。可以使用命令查看该demo的详细使用

```
plugin_sdk start_demo_app_auth -h
```



修改后的插件源码如下所示：

```python
import requests
import requests.exceptions
import json
from ruxit.api.base_plugin import BasePlugin
from ruxit.api.exceptions import AuthException, ConfigException
from ruxit.api.snapshot import pgi_name


class DemoPlugin(BasePlugin):
    def query(self, **kwargs):
        user = self.config["user"]
        password = self.config["password"]
        pgi = self.find_single_process_group(pgi_name('plugin_sdk.demo_app_auth'))
        pgi_id = pgi.group_instance_id
        stats_url = "http://localhost:8090"

        try:
            response = requests.get(stats_url, auth=(user, password))
            if response.status_code == 401:
                raise AuthException(response)
            stats = response.json()
        except requests.exceptions.ConnectTimeout as ex:
            raise ConfigException('Timeout on connecting with "%s"' % stats_url) from ex
        except requests.exceptions.RequestException as ex:
            raise ConfigException('Unable to connect to "%s"' % stats_url) from ex
        except json.JSONDecodeError as ex:
            raise ConfigException('Server response from %s is not json' % stats_url) from ex

        self.results_builder.absolute(key='random', value=stats['random'], entity_id=pgi_id)
        self.results_builder.relative(key='counter', value=stats['counter'], entity_id=pgi_id)
```

相应的plugin.json 不需要做修改

```json
{
  "version": "1.8",
  "name": "custom.python.demo_plugin_auth",
  "type": "python",
  "entity": "PROCESS_GROUP_INSTANCE",
  "metricGroup": "demo_metrics.auth",
  "technologies": ["PYTHON"],
  "source": {
    "package": "demo_plugin_auth",
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
	"displayname": "Random value"
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
    "displayName": "OneAgent Demo Auth Extension"
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

你可以看到，向 Agent 上报 特定类型错误的方法是产生一个特定类型的异常

```python
                raise AuthException(response)
            stats = response.json()
```

我们处理的第一类错误是无访问权限。 HTTP应用程序通常使用错误码401来响应。我们检查该错误并 raise `AuthException`。服务器上的结果可能如下所示：

![../_images/demo_02_configuration_auth_error.png](https://dynatrace.github.io/plugin-sdk/_images/demo_02_configuration_auth_error.png)

除了无权限问题，其他错误也可能出现。如果你想要拦截它们并向用户提供错误信息，可以像这样做：

```python
            raise ConfigException('Timeout on connecting with "%s"' % stats_url) from ex
        except requests.exceptions.RequestException as ex:
            raise ConfigException('Unable to connect to "%s"' % stats_url) from ex
        except json.JSONDecodeError as ex:
            raise ConfigException('Server response from %s is not json' % stats_url) from ex
```

这里有几种特定错误被捕获并转换成 [`ConfigException`](https://dynatrace.github.io/plugin-sdk/_apidoc/ruxit.api.html#ruxit.api.exceptions.ConfigException):

- 连接超时（服务器可能正忙）
- 无法连接（访问到错误的端口）
- 接收到预期之外的响应（可能连接到其他的服务器了或者接收到的是文本响应而不是JSON）。

一旦出现这些情况，web UI 上都会展示相应的错误描述：

![../_images/demo_02_configuration_config_error.png](https://dynatrace.github.io/plugin-sdk/_images/demo_02_configuration_config_error.png)

请注意，与实际情况一样，可能会发生许多不同的错误类型。服务器可能会相应HTTP status 500；服务器可能会使用有效的JSON进行响应，但不是你想要请求的数据（随机数和计数器）；或者值可能填充的是无法转换成数字的数据。我们需要考虑 哪些问题需要被注意，哪些问题需要额外地处理。