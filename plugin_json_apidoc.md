# `Plugin.json` reference

`plugin.json` 包含4个主要元素： metadata, metrics, visualization, configuration 。

基本格式如下所示：

```json
{
    "name": "custom.python.demo_plugin",
    "version": "1.000",
    "type": "python",
    "requiredAgentVersion": "1.90",
    "entity": "PROCESS_GROUP_INSTANCE",
    "technologies" : [
        "PYTHON"
    ],
    "source": {
        "package": "demo_plugin",
        "className": "DemoPlugin",
        "install_requires": [
            "requests>=2.6.0"
        ],
        "activation": "Singleton"
    },
    "metricGroup": "My_Metrics.plugins",
    "metrics": [
        {
            "timeseries": {
                "key": "random",
                "unit": "Count",
                "displayname":".NET GC time"
            }
        },
        {
            "timeseries": {
                "key": "counter",
                "unit": "Count",
                "displayname":"Counter Value"
            },
            "alert_settings": [
                {
                            "alert_id": "custom_gc_alert_high",
                            "event_type": "PERFORMANCE_EVENT",
                            "event_name": "High GC time",
                            "description": "The {metricname} of {severity} is {alert_condition} the threshold of {threshold}",
                            "threshold": 35.0,
                            "alert_condition": "ABOVE",
                            "samples":5,
                            "violating_samples":3,
                            "dealerting_samples":5
                                    }
            ]
        }
    ],
    "ui": {
        "keymetrics": [],
        "keycharts": [],
        "charts": []
    },
    "configUI": {
        "displayName": "DemoPlugin",
        "properties": []
    },
    "properties": []
}
```



## Metadata

每个插件有以下属性：

| ield                                        | Type          | Mandatory                                                    | Description                                                  |
| :------------------------------------------ | :------------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| version                                     | String        | Yes                                                          | The plugin version in format "d.d+(.d+)?", must be updated whenever the plugin definition is updated. |
| name                                        | String        | Yes                                                          | A unique plugin name in Java package format. Custom plugins should follow custom."jmx"\|"python"\| pmi"."a".."z"\|"A".."Z"\|"0".."9"\|"_" \| "-" rule. For example custom.python.MyNewPlugin_Ver1 or custom.jmx.azNewA-Ver2 |
| experimentalMinVersion productiveMinVersion | String        | No                                                           | Those fields are used internally for build-in plugin releases and should not be used in custom plugins. |
| type                                        | String        | Yes                                                          | Possible values: "python" for python custom plugins data collection, "JMX" for collecting data from Java Mbeans. |
| requiredAgentVersion                        | String        | No                                                           | Used to mark plugins that require a minimum OneAgent version. This should be used when new APIs are introduced in OneAgent. |
| entity                                      | String        | Yes                                                          | Entity type used in plugin activation rules and default assignment of metrics. Use "PROCESS_GROUP_INSTANCE". |
| metricGroup                                 | String        | Yes                                                          | Metric group is used for grouping custom metrics into a hierarchical namespace where different sources, for example multiple plugins can contribute. Moreover, metric group becomes primary part of metric key. Hence once defined it could not be changed. Allowed characters are: "a".."z"\|"A".."Z"\|"0".."9"\|"_"\|"-"\|".". To add metrics to "Technology" group use prefix "tech." and technology group as defined in the: [technologies group table.](https://dynatrace.github.io/plugin-sdk/api/metric_groups.html) It is also possible to create own entry, for example: "My_Metrics.plugins" |
| processTypeNames(deprecated)                | String Array  | Need one                                                     | Type of process that plugin monitors, as defined in process type name of the [known process types table.](https://dynatrace.github.io/plugin-sdk/api/known_process.html) This is deprecated method, please use technologies instead. |
| processTypes((deprecated))                  | Integer Array | Type of process that plugin monitors, as defined in the ID column of the [known process types table.](https://dynatrace.github.io/plugin-sdk/api/known_process.html) This serves as an alternative to processTypeNames. This is deprecated method, please use technologies instead. |                                                              |
| technologies                                | String Array  | Type of technology that plugin monitors, as defined in the [known technologies table](https://dynatrace.github.io/plugin-sdk/api/known_technologies.html). |                                                              |

- `type`:   主要是 python/jmx

- `entity`:  用于 插件的激活规则 和 指标的默认分配粒度
- `metricGroup`：
  - 将 指标 聚合 成  层次结构的命名空间中，在命名空间中，可以使用不同的数据来源（比如聚合来自不同插件的数据）。
  -  此外，metricGroup 是 指标key的 重要组成部分。因此，一旦定义就无法更改。
  - 允许的字符为：“ a” ..“ z” |“ A” ..“ Z” |“ 0” ..“ 9” |“ _” |“-” |“。”。
  - 要将 指标 添加到 “Technology" group，请使用前缀 ”tech.“ ; Technology group 的定义在 [technologies group table](https://dynatrace.github.io/plugin-sdk/api/metric_groups.html)  。
  - 也可以创建自己的条目，例如：“ My_Metrics.plugins”

- `technologies` : 指定 插件要监控的 technology type ，定义在  [known process types table](https://dynatrace.github.io/plugin-sdk/api/known_process.html) 

如果同时设置了 `technologies`  和 `processTypeNames / processType`字段，则它们都必须匹配才能激活插件。可以在页面 Technology overview 上浏览 Technologys 。

![../_images/TechnologyOverview.png](https://dynatrace.github.io/plugin-sdk/_images/TechnologyOverview.png)

使用 MetricGroup 的首选方法是在 metric group 定义之前 添加 "tech" 前缀，比如 "tech.elasticsearch" 。 使用这种方法，其他用户才能在 technology 上下文中找到别人定义的指标名，达到分享复用的目的，例子如下所示:

![../_images/CustomChartsMetricGroupTechnology.png](https://dynatrace.github.io/plugin-sdk/_images/CustomChartsMetricGroupTechnology.png)

MetricGroup字段也可以用于创建全新的条目。请注意，您可以使用点来创建子组，例如此处的示例“ My_Metrics.plugins”：

![../_images/CustomChartsMetricGroupNewEntry.png](https://dynatrace.github.io/plugin-sdk/_images/CustomChartsMetricGroupNewEntry.png)

但是，之前没有metricGroup字段的已存在且正在运行的插件，就不需要设置metricGroup字段了。在这种情况下，metricGroup将自动设置为与插件名称相同，以免破坏数据的连续性。

### Source section

| Field                   | Type         | Mandatory Field | Description                                                  |
| :---------------------- | :----------- | :-------------- | :----------------------------------------------------------- |
| package                 | String       | Yes             | Package name which will be imported and executed by OneAgent. |
| className               | String       | Yes             | Name of the plugin's main Python class. Must inherit from [`BasePlugin`](https://dynatrace.github.io/plugin-sdk/_apidoc/ruxit.api.html#ruxit.api.base_plugin.BasePlugin). |
| install_requires        | String Array |                 | Relates to Python packaging, see: https://docs.python.org/3.6/distutils/setupscript.html. |
| packages                | String Array |                 | Relates to Python packaging, see: https://docs.python.org/3.6/distutils/setupscript.html. |
| package_data            | String Array |                 | Relates to Python packaging, see: https://docs.python.org/3.6/distutils/setupscript.html. |
| modules                 | String Array |                 | Relates to Python packaging, see py_modules in: https://docs.python.org/3.6/distutils/setupscript.html. |
| activation              | String       |                 | An additional parameter that controls the plugin activation process, possible values: 'Singleton' or 'SnapshotEntry'. Singleton means that only one plugin class instance will be created, no matter how many process groups of a given type are detected on the system. SnapshotEntry means that one plugin class instance will be created for each detected process group of a given type. If activation is not define 'SnapshotEntry' will be used. |
| activation_name_pattern | String       |                 | Python regular expression pattern for matching process group name: https://docs.python.org/3.6/library/re.html#regular-expression-syntax. |

- `activation` :   控制插件激活的附加参数，可选值为 `Singleton/SnapshotEntry`
  -  `Singleton`  无论检测到多少个 匹配 的进程组 ，只 创建 一个插件实例
  - `SnapshotEntry`   检测到的每一个匹配的进程组，都为其创建一个插件实例。
  - 默认是 `SnapshotEntry`   

- `activation_name_pattern` : 如果定义了activation_name_pattern，则该模式必须与进程组名称匹配才能激活插件。进程组名称在 Technology overview 页面上可以看到：

![../_images/TechnologyPGNames.PNG](https://dynatrace.github.io/plugin-sdk/_images/TechnologyPGNames.PNG)



## Metrics

这部分定义了插件 要采集哪些指标。每个 metric 使用如下格式的JSON来定义：

```json
{
        "metrics": [
                {
                        "entity": "PROCESS_GROUP_INSTANCE",
                        "timeseries": {
                                "key": "idx_tup_fetch",
                                "unit": "PerSecond",
                                "aggregation": "avg",
                                "dimensions": ["database"],
                                "displayname":"Fetch"
                        },
                        "source": {
                                "query": "db_stats['objects']"
                        }
                }
        ]
}
```

该部分声明了指标的元数据：

| Field      | Type   | Description                                                  |
| :--------- | :----- | :----------------------------------------------------------- |
| entity     | String | Entity type metrics should be associated with. Inherited from plugin metadata if not provided here. |
| timeseries | Object | Specification for metric timeseries. See chart below.        |
| source     |        | Can be used to specify any valid JSON structure, to be processed by the Plugin. See [MSSQL](https://dynatrace.github.io/plugin-sdk/plugins/ruxit_mssql.html) or [MongoDB](https://dynatrace.github.io/plugin-sdk/plugins/ruxit_mongodb.html) plugins for sample usage. |

- entity  指标分配的实体类型（粒度），如果没有提供则从 plugin中继承entity字段
- timeseries  指标格式
- source   用于指定任意合法的JSON节后，被 plugin 处理。可以参考  [MSSQL](https://dynatrace.github.io/plugin-sdk/plugins/ruxit_mssql.html) or [MongoDB](https://dynatrace.github.io/plugin-sdk/plugins/ruxit_mongodb.html)  插件，看看具体的应用

使用以下字段定义指标(metric timeseries)格式：

| Field       | Type         | Description                                                  |
| :---------- | :----------- | :----------------------------------------------------------- |
| key         | String       | Metric name. Must be unique within this plugin. Only letters, numbers and "-" , "_" chars are allowed in metrics keys. |
| unit        | Unit         | Metric unit. Must be one of the available units described below. |
| aggregation | String       | Time series data point aggregation (MIN/MAX/AVG/SUM). Default: AVG. |
| dimensions  | String Array | Dimensions are used to provide 1 metric per plugin ObjectName key property value. For example, version, service, or database. Dimension "rx_pid" at index 0 means the system process ID (PID). Only letters, numbers and "-" , "_" chars are allowed in metrics dimensions. |
| displayname | String       | Metric display name represent metric in Dynatrace. This field is obligatory. Must be different than metric key. |

- `key` 指标名，
  - 必须插件内唯一，
  - 只允许 字母，数字和 "-" , ”_“ 等字符构成
- `unit`  指标单位
  - NanoSecond, MicroSecond, MilliSecond, Second, Byte, KiloByte, MegaByte, BytePerSecond, BytePerMinute, KiloBytePerSecond, KiloBytePerMinute, MegaBytePerSecond, MegaBytePerMinute, Ratio, Percent, Promille, Count, PerSecond, PerMinute
- `aggregation`   时间序列数据点聚合 
  -  (MIN/MAX/AVG/SUM)
  - 默认 是 AVG
- `dimensions`    用于为每个插件的`ObjectName`键属性值提供1个指标。例如，版本，服务，数据库等。
  - `dimensions[0]` 的 rx_pid 表示 系统进程 ID
  - 仅允许使用字母，数字和“-”，“ _”字符    
- `displayname` 在 Server 页面上展示的指标名
  - 此字段是必填项。
  - 必须与 metric.key 不同 。



## Metric alerts

这部分定义了 由Server生成的指标告警。可以为插件生成的每个 `timeseries ` 定义多个告警设置。

这部分声明 metirc alerts 的元数据：

| Field              | Type    | Description                                                  |
| :----------------- | :------ | :----------------------------------------------------------- |
| alert_id           | String  | Unique alert id. Only letters, numbers and "-" , "_" chars are allowed in alert_id. |
| event_type         | String  | String Allowed types: PERFORMANCE_EVENT, ERROR_EVENT, AVAILABILITY_EVENT. |
| description        | String  | Description defines alert message, following code snippets could be used: {threshold} the value of the custom threshold that was violated {severity} the violating value {entityname} the display name of the entity where the metric violated {violating_samples} the number of violating samples that led to that event {dimensions} a string containg the violating dimensions of the metric {alert_condition} a string showing if above or below threshold is alerting |
| event_name         | String  | Event name displayed on UI pages.                            |
| threshold          | Float   | The value of the threshold.                                  |
| alert_condition    | String  | ABOVE or BELOW.                                              |
| samples            | Integer | Size of the “window” in which violating_samples are counted. |
| violating_samples  | Integer | The number of violating samples that rise an alert.          |
| dealerting_samples | Integer | The number of not violating samples that deactivate the alert. |

一个告警例子如下所示：

![../_images/alert.png](https://dynatrace.github.io/plugin-sdk/_images/alert.png)

```json
{
        "timeseries":
        {
                "key":"TimeInGC",
                "unit":"Percent",
                "displayname":".NET GC time",
                "dimensions":["rx_pid"]
        },
        "alert_settings": [
                {
                        "alert_id": "custom_gc_alert_high",
                        "event_type": "PERFORMANCE_EVENT",
                        "event_name": "High GC time",
                        "description": "The {metricname} of {severity} is {alert_condition} the threshold of {threshold}",
                        "threshold": 35.0,
                        "alert_condition": "ABOVE",
                        "samples":5,
                        "violating_samples":3,
                        "dealerting_samples":5
                }
        ],
}
```

这个例子中，当5个样本中的3个样本 > 35% 时，告警产生；然后当 5个样本都 <35%时，告警结束。



## Visualization

这部分定义了指标在每个进程页上是如何展示和绘制的，它包含一个可选的 `charts` 段 和 一个可选的 `keycharts` 段 。每个段都是同样的格式，如下所示：

```json
{
        "ui" :
        {
                "keymetrics" : [
                        {
                        "key" : "requestCount",
                        "aggregation" : "avg",
                        "mergeaggregation" : "sum",
                        "displayname" : "Requests"
                        }
                ],
                "keycharts" : [ ],
                "charts": [ ]
        }
}
```

 `keymetrics` 段 是完全可选的，允许您最多定义两个指标，作为  Process infographic(进程信息图) 的 一部分。它具有以下属性：

| Field            | Type   | Mandatory Field | Description                                                  |
| :--------------- | :----- | :-------------- | :----------------------------------------------------------- |
| key              | String | Yes             | The key for the time series to put into the graphic.         |
| aggregation      | String |                 | Time series data point aggregation (min/max/avg/sum/count). Default: AVG. |
| mergeaggregation | String |                 | If the metric contains multiple dimensions, this defines how to aggregate the dimension values into a single one. Default: AVG. |
| displayname      | String |                 | The name to display in the graphic. Overwite metric displayname. Default: metric displayname. |
| unit             | Unit   |                 | Displayed unit. Must be one of the available units. Default: metric unit. |

- `aggregation` 时间点数据聚合方式，默认是 AVG
- `mergeaggregation`  如果一个指标包含多个维度，则这定义了如何将各个维度的值汇总为一个维度。默认是 AVG
- `displayname` 没指定则默认是   metric.displayname
- unit,   默认是 metric.unit

以下是Web UI中的关键指标示例：

![../_images/viz_03_keymetric_01.png](https://dynatrace.github.io/plugin-sdk/_images/viz_03_keymetric_01.png)

每个 `chart` 段 和 `keychart` 段 ，都是一样的格式，如下所示:

```json
{
    "group": "Section Name",
    "title": "Chart Name",
    "series": [
     {
            "key": "MetricName",
            "aggregation": "avg",
            "displayname": "Display name for metric",
            "seriestype": "area"
     },
     {
            "key": "Other Metric Name",
            "aggregation": "avg",
            "displayname": "Display name for metric",
            "color": "rgba(42, 182, 244, 0.6)",
            "seriestype": "area"
     }
    ]
}
```

这个 `chart` 段 描述了 如何在进程页的细节子页面(点击 Further detail)中绘制 每个指标。

这两个部分都允许定义一系列图表chart。每个chart 具有以下必需属性：

| Field       | Type   | Mandatory Field | Description                                                  |
| ----------- | ------ | --------------- | ------------------------------------------------------------ |
| group       | String | Yes             | The section name that the chart should be put into.          |
| title       | String | Yes             | The name of the chart.                                       |
| description | String |                 | Chart description.                                           |
| series      | Array  | Yes             | An array of time series and charting definitions. One chart can contain multiple metrics. |

一个 `series ` 有以下属性：

| Field            | Type    | Mandatory Field | Description                                                  |
| :--------------- | :------ | :-------------- | :----------------------------------------------------------- |
| key              | String  | Yes             | The key for the time series to chart.                        |
| displayname      | String  |                 | Display name to show for the metric. Default: metric key.    |
| aggregation      | String  |                 | How multiple minute values should be aggregated in charts when viewing a longer time frame. Possible values: SUM, AVG, MIN, MAX. Default: AVG. |
| mergeaggregation | String  |                 | Key charts do not show multiple dimensions. If the metric contains multiple dimensions, this defines how to aggregate the dimension values into a single dimension. Default: AVG. |
| color            | String  |                 | HTML notation of a color (RGB or RGBA). Default: #00a6fb.    |
| seriestype       | String  |                 | Chart type. Possible values are: line, area, and bar. Default: area. |
| rightaxis        | Boolean |                 | If true, the metric will be placed on the right instead of the left axis. Note that web UI does support dual axis charts. Default: false. |
| stacked          | Boolean |                 | If true, then multiple metrics will be stacked upon each other. This only works for area and bar charts. Default: false. |
| unit             | Unit    |                 | Displayed unit. Must be one of the available units. Default: metric unit. |

`Keycharts ` 在每个 进程页都是可见的，如下所示：

![../_images/viz_01_keychart_01.png](https://dynatrace.github.io/plugin-sdk/_images/viz_01_keychart_01.png)

其他 `charts` ，在指定 进程页中 点击 **Further details** 之后展示：

![../_images/viz_02_chart_01.png](https://dynatrace.github.io/plugin-sdk/_images/viz_02_chart_01.png)



## Plugin configuration

`plugin configuration` 部分 控制 web UI 上 Setting > Monitored technologies 的 插件展示 和 其配置，比如 user name, password, connection string 等。 `configUI` 部分 定义在UI上展示了配置字段。`properties` 部分 定义 下发给 插件的 配置属性。

下面的例子展示了如何定义一个 `plugin configuration` :

```json
{
        "configUI" :{
                "displayName": "HAProxy",
                "properties" : [
                        { "key" : "url", "displayName": "URL", "displayOrder": 3, "displayHint": "http://localhost:8080/haproxy-statistics" },
                        { "key" : "auth_user", "displayName": "User", "displayOrder": 1 },
                        { "key" : "auth_password", "displayName": "Password", "displayOrder": 2 }
                ]
         }
}
```

`ConfigUI` 部分 有以下 属性：

| Field                   | Type   | Mandatory Field | Description                                                  |
| :---------------------- | :----- | :-------------- | :----------------------------------------------------------- |
| displayName             | String |                 | Human readable plugin name. This name is displayed in web UI at **Settings > Monitored technologies > Custom plugins** once the plugin is uploaded. |
| properties.key          | String | Yes             | Config property key, needs to match key from configUI properties section. |
| properties.displayName  | String | Yes             | Human readable property name.                                |
| properties.displayHint  | String |                 | Hint displayed in the tool-tip.                              |
| properties.displayOrder | String |                 | Determines display order on plugin configuration tile.       |

如果需要 插件配置, 则与 配置UI 相对应的 属性 properties 必须定义。这些 properties 负责携带 在 UI 上填写的 配置值。配置下发和配置处理等详情请参阅 [plugin configuration](Configuration.md) 。

JSON 定义的每个属性，格式都如下所示：

```json
{
        "properties" : [
                 {
                          "key" : "url",
                          "type" :  "String",
                          "defaultValue" : "https://localhost/haproxy_stats_ssl"
                 },
                 {
                          "key" : "auth_user",
                          "type" :  "String",
                 },
                 {
                          "key" : "auth_password",
                          "type" :  "Password",
                          "defaultValue" : "password"
                 }
          ]
}
```

 `properties`  包含以下属性：

| Field        | Type             | Mandatory Field | Description                                                  |
| ------------ | ---------------- | --------------- | ------------------------------------------------------------ |
| key          | String           | Yes             | Property key. Must be unique within this plugin and must match the key from configUI properties. |
| type         | String           | Yes             | Possible values: 'STRING', 'BOOLEAN', 'INTEGER', 'FLOAT', 'PASSWORD'. For 'PASSWORD' stars will be displayed while typing. |
| defaultValue | Defined in'type' |                 | Default value.                                               |