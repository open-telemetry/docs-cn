# 开发者指南

在贡献代码之前，请先打开源文件，读一下我们的[贡献者指南文档](./CONTRIBUTING.md)。我们非常欢迎你对文档和代码的提高。

代码库使用一个单库。我们使用 [Lerna](https://lerna.js.org/) 作为管理内部模块依赖的工具，他能更加容易的去开发协调多个模块。运行的命令是使用 `npm`，而不是 `lerna`。

## 开发工具

因为项目支持多 Node 版本，所以推荐使用一个版本管理工具，例如 [nvm](https://github.com/creationix/nvm) 

安装完 Node 后运行下面的命令：

```sh
npm install
```
会安装完所有必要的模块

## 测试

### 单元测试

可使用下面的命令，来执行所有的单元测试：

```sh
npm run test
```
可使用下面的命令，在开发的过程中监听变化持续运行单元测试：

```sh
npm run tdd
```

## Linting

我们使用 [gts](https://www.npmjs.com/package/gts) 来保证新代码符合我们的代码规范。

在提交代码之前，请确保没有 lint 报错。

使用下面命令来检查 lint 问题：

```sh
npm run lint
```

使用下面命令来修复 lint 问题：

```sh
npm run lint:fix
```

## 持续集成

We rely on CircleCI 2.0 for our tests. If you want to test how the CI behaves
locally, you can use the CircleCI Command Line Interface as described here:
<https://circleci.com/docs/2.0/local-jobs/>

我们依靠 CircleCI 2.0 来做持续集成。如果你想在本地测试 CI，你可以使用 CircleCI 命令行界面，可以阅读文档使用<https://circleci.com/docs/2.0/local-jobs/>

安装 `circleci` CLI 后，只需运行以下命令之一：

```sh
circleci build --job lint
circleci build --job node8
circleci build --job node10
circleci build --job node11
circleci build --job node12
circleci build --job node12-browsers
```

## 文档

我们使用 [typedoc](https://www.npmjs.com/package/typedoc) 来生成 api 文档。

使用下面的命令生成文档：

```sh
npm run docs
```

文档的输出路径在 `packages/opentelemetry-api/docs/out` 下。