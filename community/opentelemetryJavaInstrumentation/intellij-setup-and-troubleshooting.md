### IntelliJ 配置

**这是一个不错的开头!** 当你在命令行进行项目的构建的时候，请确保 Intellij 使用了相同的 Java 配置。这样可以确保 gradle 任务可以合理利用缓存并减少构建的时间。

推荐的 plugin 和配置如下：

## [google-java-format](https://plugins.jetbrains.com/plugin/8527-google-java-format)

安装:

![google format](https://user-images.githubusercontent.com/5099946/131758519-14d27c17-5fc2-4447-84b0-dbe7a7329022.png)

配置:

![enable google format](https://user-images.githubusercontent.com/5099946/131759832-36437aa0-a5f7-42c0-9425-8c5b45c16765.png)

## [Save Actions](https://plugins.jetbrains.com/plugin/7642-save-actions)

安装:

![save action](https://user-images.githubusercontent.com/5099946/131758940-7a1820db-3cf4-4e30-b346-c45c1ff4646e.png)

配置:

![Recommended Settings](save-actions.png)

## 故障排除

有的时候使用Intellij会感到困惑，可能是因为这个项目中的模块数量太多，也可能是其他原因。无论如何，这里有一些可能会有所帮助的事情：

### 使缓存无效 > "只需重启"

* 选择文件 > Invalidate Caches...
* 取消选择所有选项
* 单击 "Just restart" 链接

这似乎解决了更多问题，而不仅仅是关闭和重新打开 Intellij :shrug:.

### 删除你的`.idea`目录

- 关闭 Intellij
- 删除`.idea`本地仓库根目录下的目录
- 打开 Intellij
