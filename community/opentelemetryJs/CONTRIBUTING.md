# 贡献者指南

我们很乐意得到您的帮助！

## 开发快速开始

可以跟着下列的步骤，来快速开始项目。想要查看更多细节的内容，可以在 [开发](#development) 下查看

```sh
git clone https://github.com/open-telemetry/opentelemetry-js.git
cd opentelemetry-js
npm install
npm run compile
npm test
```

## 报告一个 bug 或者提出一个新功能

报告 bug 是一个很重要的贡献。报告中请确保包括：

- expected and actual behavior.
- Node version that application is running.
- OpenTelemetry version that application is using.
- if possible - repro application and steps to reproduce.

- 预期的结果和实际的结果。
- 启动应用时，Node 的版本。
- 启动应用时，使用的 OpenTelemetry 的版本。
- 如果可以，请列出如何重现该 bug 的步骤。

## 如何贡献

### 在你开始贡献之前

请阅读项目的 [贡献指南](https://github.com/open-telemetry/community/blob/master/CONTRIBUTING.md) 中的 Opentelemetry 项目实践。

#### 常规提交

常规提交规范是基于提交消息的轻量级约定。它提供了一个简单的规则集来创建一个明确的提交历史;这使得在其上编写自动化工具变得更容易。该约定与 SemVer 相吻合，通过描述提交消息中所做的功能、修复和破坏性更改。你可以查看一些案例 [here](https://www.conventionalcommits.org/en/v1.0.0-beta.4/#examples)。
使用 [commitlint](https://github.com/conventional-changelog/commitlint) 和 [husky](https://github.com/typicode/husky) 来防止不好的提交消息。
例如，你想要提交你的消息 `git commit -s -am "my bad commit"`。
你会收到下面的错误：

```text
✖   type must be one of [ci, feat, fix, docs, style, refactor, perf, test, revert, chore] [type-enum]
```

这个提交会通过规范验证：`git commit -s -am "chore(opentelemetry-core): update deps"`

### 分支

为了保证库的规整和易管理，你应该在分支上做开发。可以点击仓库上方的 `Fork` 按钮，来生成分支，然后用命令克隆分支到本地：`git clone git@github.com:USERNAME/opentelemetry-js.git`。

你可以添加本仓库到你本地的远程分支作为 “upstream”，以保证更新到最新的版本，可以通过下面的命令添加远程分支：

```sh
git remote add upstream https://github.com/open-telemetry/opentelemetry-js.git

#verify that the upstream exists
git remote -v
```

更新你的分支，拉去上游厂库的分支和提交，然后合并到你的主分支中：

```sh
git fetch upstream
git checkout main
git merge upstream/main
```

保存你所有的工作记录到你本地中，你可能会从主分支中遇到代码冲突。

请阅读 [GitHub workflow](https://github.com/open-telemetry/community/blob/master/CONTRIBUTING.md#github-workflow) 的文章作为贡献指南。

## 开发

### 工具使用

- [NPM](https://npmjs.com)
- [TypeScript](https://www.typescriptlang.org/)
- [lerna](https://github.com/lerna/lerna) 管理依赖，编译，和包之间的格式检查。大多数命令应该使用 npm 脚本运行。
- [MochaJS](https://mochajs.org/) 单元测试
- [gts](https://github.com/google/gts)
- [eslint](https://eslint.org/)

Most of the commands needed for development are accessed as [npm scripts](https://docs.npmjs.com/cli/v6/using-npm/scripts). It is recommended that you use the provided npm scripts instead of using `lerna run` in most cases.

对于开发，大多数命令应该使用 [npm scripts](https://docs.npmjs.com/cli/v6/using-npm/scripts)。在大多数情况下，建议您使用提供的 npm 脚本，而不是使用' lerna run '。

### 安装依赖

This will install all dependencies for the root project and all modules managed by `lerna`. By default, a `postinstall` script will run `lerna bootstrap` automatically after an install. This can be avoided using the `--ignore-scripts` option if desired.

通过 `lerna` 为主项目安装所有的依赖和所有的模块。默认，在安装之后 `postinstall` 脚本将会自动运行 `lerna bootstrap`。可以使用 `--ignore-scripts` 参数来避免运行。

```sh
npm install
```

### 编译模块

所有模块的管理通过集成 typescript 项目 [Project References](https://www.typescriptlang.org/docs/handbook/project-references.html)。这意味着一个模块中的重大更改将自动反映在其依赖模块的编译中。

除非您知道自己在做什么，否则不要使用 lerna 编译所有模块，因为这会导致为项目中的每个模块生成一个新的 typescript 进程。

```sh
# Build all modules
npm run compile

# Remove compiled output
npm run clean
```

这些命令也可以针对特定的包而不是整个项目运行，这可以在开发时加快编译速度。

```sh
# Build a single module and all of its dependencies
cd packages/opentelemetry-module-name
npm run compile
```

最后，可以使用 `watch` npm 脚本在文件更改时监听文件变更持续运行构建。

```sh
# Build all modules
npm run watch

# Build a single module and all of its dependencies
cd packages/opentelemetry-module-name
npm run watch
```

### 运行测试

与编译类似，测试可以从根目录运行所有测试，也可以从单个模块运行该模块的测试。


```sh
# Test all modules
npm test

# Test a single module
cd packages/opentelemetry-module-name
npm test
```

### Linting

这个项目使用了 `gts` 和 `eslint` 的组合。 就像测试和编译一样，可以对所有包或仅对单个包进行 linting。


```sh
# Lint all modules
npm run lint

# Lint a single module
cd packages/opentelemetry-module-name
npm run lint
```

还有一个脚本可以自动修复许多 linting 错误。

```sh
# Lint all modules, fixing errors
npm run lint:fix

# Lint a single module, fixing errors
cd packages/opentelemetry-module-name
npm run lint:fix
```

### 添加新的依赖包

要添加新包，请将 `packages/template` 复制到新包目录并修改 `package.json` 文件以反映你所需的包设置。 如果包不支持浏览器，`karma.conf` 和 `tsconifg.esm.json` 文件可能会被删除。 如果包将支持 es5 目标，则应将 `tsconfig.json` 中对 `tsconfig.base.json` 的引用更改为 `tsconfig.es5.json`。 

添加包后，从项目的根目录运行 `npm install`。 这将自动更新 `tsconfig.json` 项目引用并在新包中安装所有依赖项。 对于支持浏览器的包，需要手动更新文件 `tsconfig.esm.json` 以包含对 ES 模块构建的引用。

### Guidelines for Pull Requests

- 通常，我们会尝试在一到两个工作日内回复评论。
- 通常期望维护者 ([@open-telemetry/javascript-maintainers](https://github.com/orgs/open-telemetry/teams/javascript-maintainers)) 应该审查并合并每个 PR。
  - 如果更改满足下面列出的要求，审批者也可以合并拉取请求。
- 大多数 PR 应该在一到两周内合并。
- If a PR is taking longer than 30 days, please ping the approvers ([@open-telemetry/javascript-approvers](https://github.com/orgs/open-telemetry/teams/javascript-approvers)) as it may have been lost
- 如果 PR 花费的时间超过 30 天，请告知审批人 ([@open-telemetry/javascript-approvers](https://github.com/orgs/open-telemetry/teams/javascript-approvers))，因为它可能已经丢失。
- 依赖升级和安全修复：这个 PR 很小和/或低风险，只能维护者审查合并。
- 如果你的补丁没有被审查或者你需要一个特定的人来审查它，你可以@username 或 @open-telemetry/javascript-approvers 一个审查者在拉取请求中要求审查。
- API 更改、重大更改或大更改将受到更严格的审查，并可能需要更多的人员来审核。 这些 PR 应该只由维护者合并。
- 对现有 插件（plugin） 和 导出器（exports） 的更改通常需要原始 plugin/export 作者的批准。

### 一般合并要求

- 所有要求均由维护者自行决定。
  - 维护者可以合并没有严格满足这些要求的拉取请求。
  - 即使维护者严格满足这些要求，他们也可以关闭、阻止或搁置拉取请求。
- 没有“要求更改”的评论。
- 没有未解决的对话。
- 三个批准，包括至少两个维护者的批准
  - 贡献者的一个 pr 应该只有两个维护者的审核
  - 小的（简单的错别字、URL、更新文档或语法修复）或高优先级的更改可能会更快地合并，或者由维护者决定合并更少的审阅者。 这通常用快递标签表示。
- 对于 plugins、exporters 和 propagators，原始代码模块作者的批准是最优的，但不是必需的。
- 新的或更改的功能通过单元测试进行测试。
- 记录了新的或更改的功能。
- 重大改变在 24 小时内应该一直打开，不能合并。以便所有时区的审核者能正常审核。

如果满足上述所有要求并且没有未解决的讨论，则拉取请求可以由维护者或批准者合并。

### 生成 API 文档

- `npm run docs` 生成 API 文档. 生成的文档在 `packages/opentelemetry-api/docs/out` 目录下

### 生成 CHANGELOG 文档

- 生成并且导出到你的 [Github access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token): `export GITHUB_AUTH=<your_token>`
- `npm run changelog` 生成 CHANGELOG 文档到你的终端中 (具体查看 [RELEASING.md](RELEASING.md) 来了解更多细节).

### 基准

当必须比较两种或多种方法时，请在 benchmark/index.js 模块中编写一个基准测试，以便我们可以跟踪最有效的算法。

- `npm run bench` 运行你的基准
