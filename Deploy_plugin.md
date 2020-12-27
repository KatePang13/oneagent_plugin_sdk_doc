# Deploy your plugin

当你确定插件可以部署时，可以使用 `oneagent_build_plugin` 来构建发行版本。

**注意事项：**

发布自定义插件时需要注意：

- 插件目录至少需要相应用户的读取权限 (linux 为 dtuser, Windows为本地系统账号)

- 如果不确定插件的加载方式，以及是否已将其复制到正确的位置，请参考  [Plugin life cycle](https://dynatrace.github.io/plugin-sdk/plugin_lifecycle/index.html) 中 "Plugin loading" 那一节内容。
- 确保 插件构建的系统环境 和 部署环境 是相同的。比如：
  - 如果您的插件要部署在32位Linux机器上，则必须使用32位Python 3.6  在32位Linux机器上构建
  - 如果您的插件要部署在64位Windows机器上，则必须使用64位Python 3.6 在64位Windows机器上构建
  - 依次类推。

