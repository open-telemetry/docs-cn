### IntelliJ 配置

**这是一个不错的开头!** 当你在命令行进行项目的构建的时候，请确保 Intellij 使用了相同的 Java 配置。这样可以确保 gradle 任务可以合理利用缓存并减少构建的时间。

推荐的 plugin 和配置如下：

- Editor > Code Style > Java/Groovy > Imports
  - Class count to use import with '\*': `9999` (一个足够大到几乎不会超过的数字)
  - Names count to use static import with '\*': `9999`
  - Import Layout:
    ![import layout](https://user-images.githubusercontent.com/734411/43430811-28442636-94ae-11e8-86f1-f270ddcba023.png)
- [Google Java Format](https://plugins.jetbrains.com/plugin/8527-google-java-format)
- [Save Actions](https://plugins.jetbrains.com/plugin/7642-save-actions)
  ![Recommended Settings](save-actions.png)

