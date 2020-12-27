# Changing plugin definition

一旦 `plugin.json` 被上传到 Server, 就不是可以任意修改的了，尤其是关系到指标的定义，必须保证与历史数据的一致性。在本节，我们将讨论 plugin 定义的哪些部分是可以变更的，一起如果应对不该出现的变更。

注意：只要修改 `plugin.json`，都应该递增版本号`version` 。

特别需要说明的是，不能修改 `metrics` 中 `timeseries` 的任意元素。只要任意 ``timeseries` ` 字段变更了，都必须变更 插件名。否则 Server 将拒绝这次修改的上传。尽管您总是可以删除旧插件并创建一个新插件，但下表提供了一些替代方案。

| Change                   | Possible | Comment                                                      |
| :----------------------- | :------- | :----------------------------------------------------------- |
| Visualization            | Yes      | Entire *"ui"* section may be safely modified.                |
| Configuration            | Yes      | Entire *"configUI"* section may be modified.Note that backward plugins compatibility couldbe broken if an old configuration parameter is removed. |
| Alert definition         | Yes      | *"alert_settings"* section of *"metrics"* may besafely modified. |
| Adding new timeseries    | Yes      |                                                              |
| Removing timeseries      | No       | If new version of plugin does not send given timeseriesit will disappear from UI over time. |
| Changing timeseries name | No       | You have to create a new timeseries with different name      |
| Changing timeseries unit | No       | You have to create a new timeseries with different name      |

更多详情，请参考  [Plugin.json reference](https://dynatrace.github.io/plugin-sdk/api/plugin_json_apidoc.html)

