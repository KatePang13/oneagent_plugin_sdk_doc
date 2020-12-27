# Alerting

Server 的一个关键特性是，具备在用户受影响之前探测和分析性能问题的能力。这是依托于我们的智能告警系统的，它可以帮助你即使面对复杂的环境，也能定位性能问题的根因。为了发挥这个功能的又是，你的插件应该支持定制告警，当被监控的进程出现意外行为的时候产生相应的告警。

对于插件上报的任何时序数据，都可以定义多种告警类型条件。为此， 必须在插件的metrics定义中添加`alert_setting` 分节。请注意，这里不需要做任何的代码修改。

```json
{
        "metrics": [
                {
                        "metricKey": "counter",
                        "alertSettings": [
                                {
                                        "alertId": "counter_alert_high",
                                        "eventType": "PGI_CUSTOM_PERFORMANCE",
                                        "eventName": "Enormous counter rate",
                                        "threshold": 10.0,
                                        "alertCondition": "ABOVE",
                                        "samples": 5,
                                        "violatingSamples": 3,
                                        "deAlertingSamples": 5
                                },
                                {
                                        "alertId": "counter_alert_low",
                                        "eventType": "PGI_CUSTOM_ERROR",
                                        "eventName": "Low counter rate",
                                        "threshold": 1.0,
                                        "alertCondition": "BELOW",
                                        "samples": 5,
                                        "violatingSamples": 5,
                                        "deAlertingSamples": 3
                                }
                        ]
                }
        ]
}
```

关于 告警 配置，详情请参考 [plugin.json reference](https://dynatrace.github.io/plugin-sdk/api/plugin_json_apidoc.html) 

提供告警定义之后，Server 会自动处理告警的激活和停用。展示告警时，你可以在 Problem 页面 上查看 相关问题和受影响的组件信息。 

![../_images/demo_04_high_alert.png](https://dynatrace.github.io/plugin-sdk/_images/demo_04_high_alert.png)