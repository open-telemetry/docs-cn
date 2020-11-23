# OpenTelemetry贡献者指南

欢迎来到OpenTelemetry! 若你想贡献代码给OpenTelemetry，那么该指南将是你唯一的参考。


# 开始之前的准备

## CLA

在开始之前，首先你需要签订[Contributor License
Agreement](https://identity.linuxfoundation.org/projects/cncf).

## 行为准则

请确保阅读过行为准则，我们遵循 [CNCF行为准则](https://github.com/cncf/foundation/blob/master/code-of-conduct.md).

## 社区角色

OpenTelemetry是社区项目，因此，我们需要完全依赖社区才能提供一个富有生产力、友好和协作的环境。

- 查看[社区成员](./membership.md)
  获取多种贡献角色对应的责任信息

# 首次贡献

你想要参与到在所有现代化项目中默认内置一个健壮、现代化的数据采集器的伟业中来吗？我们可以帮助你理解项目的组织结构，
然后引导你从一个最佳的位置开始。最终你将有能力找到适合的issues，然后编写代码、提交，最终合并。

请注意，考虑到我们的团队已经在处理大量的issues，因此我们无法提供完成issues所需的技术支持。若你对于开发流程有疑问，请加入我们的
社群进行交流:
- 英文：[Gitter Channel](https://gitter.im/open-telemetry/community) .
- 中文： QQ群773124426

同时你也可以在[Stack Overflow](https://stackoverflow.com/questions/tagged/opentelemetry)上提问.

## 不知道做啥？

帮助永远都是想要的！例如，文档(就如你当前正在阅读的)永远存在改进的空间；代码本身、函数/变量命名、
注释永远都有可以优化的地方；单元测试方面也一直有更多更好的需要。**所以你要意识到：如果你看到哪些东西你觉得需要
修改,就勇敢的走出这一步，干就完了**

你可以通过搜索标签**opentelemetry**来获取可以参与到的issue: 
[up-for-grabs.net](https://up-for-grabs.net/#/filters?tags=opentelemetry).

如果你对代码没有兴趣，也可以帮助改进文档、在社区回答问题等等。

### 找到第一个好的主题

在OpenTelemetry中有多个代码库。每个代码库都有对于初学者很友好的issues，它们可以帮你逐步的
参与到贡献的过程中来。

例如，[Java SDK](https://github.com/open-telemetry/opentelemetry-java/) 
有[help wanted](https://github.com/open-telemetry/opentelemetry-java/labels/help%20wanted)
和 [good first issue](https://github.com/open-telemetry/opentelemetry-java/labels/good%20first%20issue)
这些标签，它们无需对系统有很深入的了解就可以参与。

同时`good first issue`标签意味着现有的成员也会给予你额外的帮助


#### 关于Github上issue的指定问题

经常会有新的贡献者询问：能否把某某issue指定给我。不幸的是，因为Github的限制，我们只能把issues指定给
组织成员或者代码仓库的协作者。

但是没有关系，你只要在issue下留言说明自己想要参与进来，我们就可以把issue安排给你了，无需显示的指定。


### 了解SIG

#### SIG结构

你可能已经注意到，OpenTelemetry组织下的一些仓库是被特殊兴趣小组(Special Interest Groups or SIG)拥有的。
我们把社区划分为SIGs，是为了改进工作流程，这样可以更容易的管理社区项目。

一个SIG是一个开发的社区单元，我们欢迎任何人加入到某一个SIG中，然后开始解决issues，评判设计标准，
做代码review等。 SIGs拥有定期的[视频会议](https://github.com/open-telemetry/community#special-interest-groups),
这里欢迎任何人参与.

### SIG特定的贡献者准则

一些SIGs拥有自己的CONTRIBUTING.md文件(译者注：描述如何贡献的文件，例如你正在阅读的)，可能
会包含额外的信息或者准则，因此你需要额外去阅读它们。这些文件一般都在SIG特定的github代码仓库中。


### 提出issue

没有准备好贡献代码，但是你发现了某些需要投入工作的地方？虽然社区欢迎任何人来贡献代码，但是我们也欢迎
大家提出问题性的issue(例如发现了代码存在的问题，但是你无法亲修改提交)。**Issues应该在对应的OpenTelemetry
代码仓库下提交**.

确保使用代码仓库提供的特定issue模版来提供详细的信息，这样可以获得更快速的回答和解决。


### 贡献

作为一个潜在的贡献者，你可以在任何时间提交你的代码变动或者想法。在有想法的时候，请不要犹豫，立刻
行动起来。


### 沟通交流

最好联系你的[SIG](https://github.com/open-telemetry/community#Special-Interest-Groups)直接沟通，这样要比
提一个通用的问题，要快得多。

对于通用的问题，请查看[沟通的标准方式](https://github.com/open-telemetry/community#Communication).

对于中国开发者，可以加入QQ群773124426进行沟通交流。

## GitHub工作流(workflow)

想要check out代码，请使用Kubernetes的[the GitHub Workflow Guide](https://github.com/kubernetes/community/blob/master/contributors/guide/github-workflow.md).

OpenTelemetry使用了同样的工作流。其中有一个重点：所有的工作都应该在forks进行，这样可以减少
对应代码仓库的代码分支。

## 开启一个Pull Request

Pull requests通常缩写"PR". OpenTelemetry遵循标准的
[github pull request](https://help.github.com/articles/about-pull-requests/)流程.

新贡献者提价PR时，常见的问题：

- 在提交第一个PR前没有正确的[签名CLA](#CLA) 
- 没有遵循SIG或者特定代码仓库的贡献者准则(查阅对应仓库中的CONTRIBUTING.md )
- PR的测试用例没有通过或者跟你提交的代码变更无关
- 引入变更需要首先通过TSC的证明，例如引入一个新的术语

## 代码Review

这里有两种代码Review的角色：Review别人、被别人Review.

为了让你的PR被他人Review时更加方便，请考虑以下因素：

- 遵循项目和代码仓库的代码风格约定
- 编写[好的commit注释](https://chris.beams.io/posts/git-commit/)
- 把大的更新变为一系列拥有内在逻辑的小更新，每个小更新都拥有独立的意义，然后聚合在一起解决一个大的问题
- 为PR打上合适的SIGs和代码Reviewer的标签：你可以在PR的过程中按照bot机器人发送的规则来做

Reviewer,review别人代码的人，需要先仔细阅读
 [CNCF行为准则](https://github.com/cncf/foundation/blob/master/code-of-conduct.md).

当Review他人的PR时， [The Gentle Art of Patch
Review](http://sage.thesharps.us/2014/09/01/the-gentle-art-of-patch-review/)，
提建议时需要循序渐进、逐步深入，这样可以指导新的贡献者更好的去协作，而不是一开始就询问他们无尽的细节：

- 这次贡献背后的想法好不好？
- 贡献的架构设计正确吗?
- 贡献经过优化打磨了吗?

注意：如果你提交的pull request没有得到足够的关注，你可以显式地提醒该仓库仓库的维护者。


# 社区

不知道你注意到没，我们已经拥有了一个十分活跃、友好的大型开源社区。我们必须要依赖大家的踊跃参与，才能变得更好，
因此欢迎你们加入！[社区成员文档](./membership.md)
包含了成员关系和行为准则的描述。

## 沟通交流

- [通用交流](https://github.com/open-telemetry/community#Communication)
- 中国开发者, 可以加入QQ群773124426进行沟通交流。

## 活动

OpenTelemetry参与了 KubeCon + CloudNativeCon, 在中国、欧洲和北美，每年共举办三次。
关于活动的具体信息请查看[CNCF Events](https://www.cncf.io/events/).
