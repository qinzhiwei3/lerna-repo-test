# lerna

最近在看[vue-cli]的源码部分，注意到这一个仓库下维护了多个package，很好奇他是如何在一个repo中管理这些package的。

我们组现在也在使用组件库的方式维护项目间共用的业务代码。有两个组件库，存在依赖的关系，目前联调是通过`npm link`的方式，性能并不好，时常出现卡顿的问题。加上前一段时间组内分享vue3也提到了lerna，于是便决定仔细的调研一下这个工具，为接下里的组件库优化助力。

[lerna](https://github.com/lerna/lerna)的文档还是很详细的,因为全是英文的，考虑到阅读问题，这里我先是自己跑了几个demo，然后做了中文翻译。后续我会出一篇专门的**lerna实战篇**

## lerna 是干什么的？

Lerna 是一个工具，它优化了使用 git 和 npm 管理多包存储库的工作流。

## 背景

1.将一个大的 package 分割成一些小的 packcage 便于分享，调试

2.在多个 git 仓库中更改容易变得混乱且难以跟踪

3.在多个 git 仓库中维护测试繁琐

## MultiRepo vs MonoRepo

![multirepo vs monorepo](./images/WX20200909-163105@2x.png)

---

## 两种工作模式

### Fixed/Locked mode (default)

vue,babel 都是用这种，在 publish 的时候,所有的包版本都会更新，并且包的版本都是一致的，版本号维护在 lerna.jon 的 version 中

### Independent mode

`lerna init --independent`

独立模式，每个 package 都可以有自己的版本号。版本号维护在各自 package.json 的 version 中。每次发布前都会提示已经更改的包，以及建议的版本号或者自定义版本号。这种方式相对第一种来说，更灵活

## 初始化项目

```javascript
npm install -g lerna // 这里是全局安装，也可以安装为项目开发依赖，使用全局方便后期使用命令行
mkdir lerna-repo
cd lerna-repo
lerna init // 初始化一个lerna项目结构，如果希望各个包使用单独版本号可以加 -i | --independent
```

![lerna init](./images/WX20200909-163838@2x.png)

## 标准的 lerna 目录结构

- 每个单独的包下都有一个 package.json 文件
- 如果包名是带 scope 的，例如@test/lerna，package.json 中，必须配置"publishConfig": {"access": "public"}

```
my-lerna-repo/
  package.json
  lerna.json
  LICENSE
  packages/
    package-1/
      package.json
    package-2/
      package.json
```

## 启用 yarn Workspaces （强烈建议）

Workspaces can only be enabled in private projects.

默认是 npm, 每个子 package 下都有自己的 node_modules，通过这样设置后，会把所有的依赖提升到顶层的 node_modules 中，并且在 node_modules 中链接本地的 package，便于调试

**注意**：必须是 private 项目才可以开启 workspaces

```json
// package.json

"private": true,
  "workspaces": [
    "packages/*"
  ],

// lerna.json

"useWorkspaces": true,
"npmClient": "yarn",
```

**hoist:** 提取公共的依赖到根目录的`node_moduels`，可以自定义指定。其余依赖安装的`package/node_modeles`中，可执行文件必须安装在`package/node_modeles`。

**workspaces:** 所有依赖全部在跟目录的`node_moduels`，除了可执行文件

![hoist vs workspaces](./images/WX20200909-170703@2x.png)

## 常用命令

### [lerna init](https://github.com/lerna/lerna/blob/master/commands/init#readme)

初始化 lerna 项目

- -i, --independent 独立版本模式

### [lerna create <name> [loc]](https://github.com/lerna/lerna/blob/master/commands/create#readme)

创建一个 packcage

- `--access` 当使用`scope package`时(@qinzhiwei/lerna)，需要设置此选项 [可选值: "public", "restricted"][默认值: public]
- `--bin` 创建可执行文件 `--bin <executableName>`
- `--description` 描述 [字符串]
- `--dependencies` 依赖，用逗号分隔 [数组]
- `--es-module` 初始化一个转化的Es Module [布尔]
- `--homepage` 源码地址 [字符串]
- `--keywords` 关键字数 [数组]
- `--license` 协议 [字符串][默认值: isc]
- `--private` 是否私有仓库 [布尔]
- `--registry` 源 [字符串]
- `--tag` 发布的标签 [字符串]
- `-y, --yes` 跳过所有的提示，使用默认配置 [布尔]

### [lerna add](https://github.com/lerna/lerna/tree/master/commands/add#readme)

为匹配的 package 添加本地或者远程依赖，一次只能添加一个依赖

```sh
$ lerna add <package>[@version] [--dev] [--exact] [--peer]
```

运行该命令时做的事情:

1. 为匹配到的 package 添加依赖
2. 更改每个 package 下的 package.json 中的依赖项属性

#### Command Options

以下几个选项的含义和`npm install`时一致

- `--dev`
- `--exact`
- `--peer` 同级依赖，使用该package需要在项目中同时安装的依赖
- `--registry <url>`
- `--no-bootstrap` 跳过 `lerna bootstrap`，只在更改对应的 package 的 package.json 中的属性

[`所有的过滤选项都支持`](#过滤选项)

## Examples

```sh
# Adds the module-1 package to the packages in the 'prefix-' prefixed folders
lerna add module-1 packages/prefix-*

# Install module-1 to module-2
lerna add module-1 --scope=module-2

# Install module-1 to module-2 in devDependencies
lerna add module-1 --scope=module-2 --dev

# Install module-1 to module-2 in peerDependencies
lerna add module-1 --scope=module-2 --peer

# Install module-1 in all modules except module-1
lerna add module-1

# Install babel-core in all modules
lerna add babel-core
```

### [lerna bootstrap](https://github.com/lerna/lerna/blob/master/commands/bootstrap#readme)

将本地 package 链接在一起并安装依赖

**执行该命令式做了一下四件事：**

> 1.为每个 package 安装依赖 

> 2.链接相互依赖的库到具体的目录，例如：如果 lerna1 依赖 lerna2，且版本刚好为本地版本，那么会在 node_modules 中链接本地项目，如果版本不满足，需按正常依赖安装 

> 3.在 bootstraped packages 中 执行 `npm run prepublish` 

> 4.在 bootstraped packages 中 执行 `npm run prepare`

![lerna bootstrap](./images/WX20200909-171333@2x.png)

![lerna bootstrap](./images/WX20200909-171656@2x.png)

#### Command Options

- `--hoist` 匹配 [glob] 依赖 提升到根目录 [默认值: '**'], 包含可执行二进制文件的依赖项还是必须安装在当前 package 的 node_modules 下，以确保 npm 脚本的运行
- `--nohoist` 和上面刚好相反 [字符串]
- `--ignore-prepublish` 在 bootstraped packages 中不再运行 prepublish 生命周期中的脚本 [布尔]
- `--ignore-scripts` 在 bootstraped packages 中不再运行任何生命周期中的脚本 [布尔]
- `--npm-client` 使用的 npm 客户端(npm, yarn, pnpm, ...) [字符串]
- `--registry` 源 [字符串]
- `--strict` 在 bootstrap 的过程中不允许发出警告，避免花销更长的时间或者导致其他问题 [布尔]
- `--use-workspaces` 启用 yarn 的 workspaces 模式 [布尔]
- `--force-local` 无论版本范围是否匹配，强制本地同级链接 [布尔] 
- `--contents` 子目录用作任何链接的源。必须适用于所有包 [字符串][默认值: .] 

### [lerna link](https://github.com/lerna/lerna/tree/master/commands/link#readme)

将本地相互依赖的 package 相互连接。例如 lerna1 依赖 lerna2，且版本号刚好为本地的 lerna2，那么会在 lerna1 下 node_modules 中建立软连指向 lerna2

#### Command Options

- --force-local 无论本地 package 是否满足版本需求，都链接本地的

```json
// 指定软链到package的特定目录
"publishConfig": {
    "directory": "dist" // bootstrap的时候软链package下的dist目录 package-1/dist => node_modules/package-1
  }
```

### [lerna list](https://github.com/lerna/lerna/tree/master/commands/list#readme)

#### list 子命令

- `lerna ls`: 等同于 `lerna list`本身，输出项目下所有的 package
- `lerna ll`: 输出项目下所有 package 名称、当前版本、所在位置
- `lerna la`: 输出项目下所有 package 名称、当前版本、所在位置，包括 private package

#### Command Options

- [`--json`](#--json)
- [`--ndjson`](#--ndjson)
- [`-a`, `--all`](#--all)
- [`-l`, `--long`](#--long)
- [`-p`, `--parseable`](#--parseable)
- [`--toposort`](#--toposort)
- [`--graph`](#--graph)

[`所有的过滤选项都支持`](#过滤选项)

##### `--json`

以 json 形式展示

```sh
$ lerna ls --json
[
  {
    "name": "package-1",
    "version": "1.0.0",
    "private": false,
    "location": "/path/to/packages/pkg-1"
  },
  {
    "name": "package-2",
    "version": "1.0.0",
    "private": false,
    "location": "/path/to/packages/pkg-2"
  }
]
```

#### `--ndjson`

以[newline-delimited JSON](http://ndjson.org/)展示信息

```sh
$ lerna ls --ndjson
{"name":"package-1","version":"1.0.0","private":false,"location":"/path/to/packages/pkg-1"}
{"name":"package-2","version":"1.0.0","private":false,"location":"/path/to/packages/pkg-2"}
```

#### `--all`

Alias: `-a`

显示默认隐藏的 private package

```sh
$ lerna ls --all
package-1
package-2
package-3 (private)
```

#### `--long`

Alias: `-l`

显示包的版本、位置、名称

```sh
$ lerna ls --long
package-1 v1.0.1 packages/pkg-1
package-2 v1.0.2 packages/pkg-2

$ lerna ls -la
package-1 v1.0.1 packages/pkg-1
package-2 v1.0.2 packages/pkg-2
package-3 v1.0.3 packages/pkg-3 (private)
```

#### `--parseable`

Alias: `-p`

显示包的绝对路径

In `--long` output, each line is a `:`-separated list: `<fullpath>:<name>:<version>[:flags..]`

```sh
$ lerna ls --parseable
/path/to/packages/pkg-1
/path/to/packages/pkg-2

$ lerna ls -pl
/path/to/packages/pkg-1:package-1:1.0.1
/path/to/packages/pkg-2:package-2:1.0.2

$ lerna ls -pla
/path/to/packages/pkg-1:package-1:1.0.1
/path/to/packages/pkg-2:package-2:1.0.2
/path/to/packages/pkg-3:package-3:1.0.3:PRIVATE
```

#### `--toposort`

按照拓扑顺序(dependencies before dependents)对包进行排序，而不是按目录对包进行词法排序。

```sh
$ json dependencies <packages/pkg-1/package.json
{
  "pkg-2": "file:../pkg-2"
}

$ lerna ls --toposort
package-2
package-1
```

#### `--graph`

将依赖关系图显示为 JSON 格式的邻接表 [adjacency list](https://en.wikipedia.org/wiki/Adjacency_list).

```sh
$ lerna ls --graph
{
  "pkg-1": [
    "pkg-2"
  ],
  "pkg-2": []
}

$ lerna ls --graph --all
{
  "pkg-1": [
    "pkg-2"
  ],
  "pkg-2": [
    "pkg-3"
  ],
  "pkg-3": [
    "pkg-2"
  ]
}
```

### [lerna changed](https://github.com/lerna/lerna/tree/master/commands/changed#readme)

列出自上次发布（打 tag）以来本地发生变化的 package

**注意:** `lerna publish`和`lerna version`的`lerna.json`配置同样影响`lerna changed`。 例如 `command.publish.ignoreChanges`.

#### Command Options

`lerna changed` 支持 [`lerna ls`](https://github.com/lerna/lerna/tree/master/commands/list#options)的所有标记：

- [`--json`](https://github.com/lerna/lerna/tree/master/commands/list#--json)
- [`--ndjson`](https://github.com/lerna/lerna/tree/master/commands/list#--ndjson)
- [`-a`, `--all`](https://github.com/lerna/lerna/tree/master/commands/list#--all)
- [`-l`, `--long`](https://github.com/lerna/lerna/tree/master/commands/list#--long)
- [`-p`, `--parseable`](https://github.com/lerna/lerna/tree/master/commands/list#--parseable)
- [`--toposort`](https://github.com/lerna/lerna/tree/master/commands/list#--toposort)
- [`--graph`](https://github.com/lerna/lerna/tree/master/commands/list#--graph)

lerna 不支持[过滤选项](https://www.npmjs.com/package/@lerna/filter-options), 因为`lerna version` or `lerna publish`不支持过滤选项.

`lerna changed` 支持 [`lerna version`](https://github.com/lerna/lerna/tree/master/commands/version#options) (the others are irrelevant)的过滤选项：

- [`--conventional-graduate`](https://github.com/lerna/lerna/tree/master/commands/version#--conventional-graduate).
- [`--force-publish`](https://github.com/lerna/lerna/tree/master/commands/version#--force-publish).
- [`--ignore-changes`](https://github.com/lerna/lerna/tree/master/commands/version#--ignore-changes).
- [`--include-merged-tags`](https://github.com/lerna/lerna/tree/master/commands/version#--include-merged-tags).



### [lerna import](https://github.com/lerna/lerna/tree/master/commands/import#readme)

`lerna import <path-to-external-repository>`

将现有的 package 导入到 lerna 项目中。可以保留之前的原始提交作者，日期和消息将保留。

**注意**：如果要在一个新的 lerna 中引入，必须至少有个 commit

#### Command Options

- `--flatten` 处理合并冲突
- `--dest` 指定引入包的目录
- `--preserve-commit` 保持引入项目原有的提交者信息

### [lerna clean](https://github.com/lerna/lerna/tree/master/commands/clean#readme)

`lerna clean`

移除所有 packages 下的 node_modules，并不会移除根目录下的
[`所有的过滤选项都支持`](#过滤选项)

### [lerna diff](https://github.com/lerna/lerna/tree/master/commands/diff#readme)

查看自上次发布（打 tag）以来某个 package 或者所有 package 的变化

```sh
$ lerna diff [package]

$ lerna diff
# diff a specific package
$ lerna diff package-name
```

> Similar to `lerna changed`. This command runs `git diff`.

### [lerna exec](https://github.com/lerna/lerna/tree/master/commands/exec#readme)

在每个 package 中执行任意命令，用波折号(`--`)分割命令语句

#### 使用方式

```sh
$ lerna exec -- <command> [..args] # runs the command in all packages
$ lerna exec -- rm -rf ./node_modules
$ lerna exec -- protractor conf.js
```

可以通过`LERNA_PACKAGE_NAME`变量获取当前 package 名称：

```sh
$ lerna exec -- npm view \$LERNA_PACKAGE_NAME
```

也可以通过`LERNA_ROOT_PATH`获取根目录绝对路径：

```sh
$ lerna exec -- node \$LERNA_ROOT_PATH/scripts/some-script.js
```

#### Command Options

[`所有的过滤选项都支持`](#过滤选项)

```sh
$ lerna exec --scope my-component -- ls -la
```

- --concurrenty

> 使用给定的数量进行并发执行(除非指定了 `--parallel`)。
> 输出是经过管道过滤，存在不确定性。
> 如果你希望命令一个接着一个执行，可以使用如下方式：

```sh
$ lerna exec --concurrency 1 -- ls -la
```

- `--stream`

从子进程立即输出，前缀是包的名称。该方式允许交叉输出：

```sh
$ lerna exec --stream -- babel src -d lib
```

![lerna exec --stream -- babel src -d lib](./images/WX20200827-182918@2x.png)

- `--parallel`

和`--stream`很像。但是完全忽略了并发性和排序，立即在所有匹配的包中运行给定的命令或脚本。适合长时间运行的进程。例如处于监听状态的`babel src -d lib -w`

```sh
$ lerna exec --parallel -- babel src -d lib -w
```

> **注意:** 建议使用命令式控制包的范围。
> 因为过多的进程可能会损害`shell`的稳定。例如最大文件描述符限制

- `--no-bail`

```sh
# Run a command, ignoring non-zero (error) exit codes
$ lerna exec --no-bail <command>
```

默认情况下，如果一但出现命令报错就会退费进程。使用该命令会禁止此行为，跳过改报错行为，继续执行其他命令

- `--no-prefix`

在输出中不显示 package 的名称

- `--profile`

生成一个 json 文件，可以在 chrome 浏览器（`devtools://devtools/bundled/devtools_app.html`）查看性能分析。通过配置`--concurrenty`可以开启固定数量的子进程数量

![lerna exec --stream -- babel src -d lib](./images/WX20200828-175558@2x.png)

```sh
$ lerna exec --profile -- <command>
```

> **注意:** 仅在启用拓扑排序时分析。不能和 `--parallel` and `--no-sort`一同使用。

- `--profile-location <location>`

设置分析文件存放位置

```sh
$ lerna exec --profile --profile-location=logs/profile/ -- <command>
```

### [lerna run](https://github.com/lerna/lerna/tree/master/commands/run#readme)

在每个 package 中运行 npm 脚本

#### 使用方法

```sh
$ lerna run <script> -- [..args] # runs npm run my-script in all packages that have it
$ lerna run test
$ lerna run build

# watch all packages and transpile on change, streaming prefixed output
$ lerna run --parallel watch
```

#### Command Options

- `--npm-client <client>`

设置`npm`客户端，默认是`npm`

```sh
$ lerna run build --npm-client=yarn
```

也可以在`lerna.json`配置:

```json
{
  "command": {
    "run": {
      "npmClient": "yarn"
    }
  }
}
```

- 其余同`lerna exec`

### [lerna version](https://github.com/lerna/lerna/tree/master/commands/version#readme)

> 生成新的唯一版本号
> bumm version：在使用类似 github 程序时，升级版本号到一个新的唯一值

#### 使用方法

```sh
lerna version 1.0.1 # 显示指定
lerna version patch # 语义关键字
lerna version       # 从提示中选择
```

当执行时，该命令做了一下事情:

1.识别从上次打标记发布以来发生变更的 package 2.版本提示 3.修改 package 的元数据反映新的版本，在根目录和每个 package 中适当运行[lifecycle scripts](#lifecycle-scripts) 4.在 git 上提交改变并对该次提交打标记(`git commit` & `git tag`) 5.提交到远程仓库(`git push`)

![lerna version](./images/WX20200904-153703@2x.png)

#### Positionals

##### semver `bump`

```sh
lerna version [major | minor | patch | premajor | preminor | prepatch | prerelease]
# uses the next semantic version(s) value and this skips `Select a new version for...` prompt
```

When this positional parameter is passed, `lerna version` will skip the version selection prompt and [increment](https://github.com/npm/node-semver#functions) the version by that keyword.
You must still use the `--yes` flag to avoid all prompts.

#### Prerelease

如果某些 package 是预发布版本(e.g. `2.0.0-beta.3`)，当你运行`lerna version`配合语义化版本时(`major`, `minor`, `patch`)，它将发布之前的预发布版本和自上次发布以来改变过的 packcage。

对于使用常规提交的项目，可以使用如下标记管理预发布版本：

- **[`--conventional-prerelease`](#--conventional-prerelease):** 发布当前变更为预发布版本（即便采用的是固定模式，也会单独升级该 package）
- **[`--conventional-graduate`](#--conventional-graduate):** 升级预发布版本为稳定版（即便采用的是固定模式，也会单独升级该 package）

当一个 package 为**预发版本**时，不使用上述标记，使用`lerna version --conventional-commits`，也会按照预发版本升级继续升级当前 package。

![lerna la](./images/WX20200907-150143@2x.png)
![lerna version --conventional-commits](./images/WX20200907-150352@2x.png)

#### Command Options

- [`--allow-branch`](#--allow-branch-glob)
- [`--amend`](#--amend)
- [`--changelog-preset`](#--changelog-preset)
- [`--conventional-commits`](#--conventional-commits)
- [`--conventional-graduate`](#--conventional-graduate)
- [`--conventional-prerelease`](#--conventional-prerelease)
- [`--create-release`](#--create-release-type)
- [`--exact`](#--exact)
- [`--force-publish`](#--force-publish)
- [`--git-remote`](#--git-remote-name)
- [`--ignore-changes`](#--ignore-changes)
- [`--ignore-scripts`](#--ignore-scripts)
- [`--include-merged-tags`](#--include-merged-tags)
- [`--message`](#--message-msg)
- [`--no-changelog`](#--no-changelog)
- [`--no-commit-hooks`](#--no-commit-hooks)
- [`--no-git-tag-version`](#--no-git-tag-version)
- [`--no-granular-pathspec`](#--no-granular-pathspec)
- [`--no-private`](#--no-private)
- [`--no-push`](#--no-push)
- [`--preid`](#--preid)
- [`--sign-git-commit`](#--sign-git-commit)
- [`--sign-git-tag`](#--sign-git-tag)
- [`--force-git-tag`](#--force-git-tag)
- [`--tag-version-prefix`](#--tag-version-prefix)
- [`--yes`](#--yes)

##### `--allow-branch <glob>`

A whitelist of globs that match git branches where `lerna version` is enabled.
It is easiest (and recommended) to configure in `lerna.json`, but it is possible to pass as a CLI option as well.

设置可以调用`lerna version`命令的分支白名单，也可以在`lerna.json`中设置

```json
{
  "command": {
    "version": {
      "allowBranch": ["master", "beta/*", "feature/*"]
    }
  }
}
```

##### `--amend`

```sh
lerna version --amend
# commit message is retained, and `git push` is skipped.
```

默认情况下如果暂存区有未提交的内容，`lerna version`会失败，需要提前保存本地内容。使用该标记可以较少 commit 的次数，将当前变更内容随着本次版本变化一次 commit。并且不会`git push`

##### `--changelog-preset`

```sh
lerna version --conventional-commits --changelog-preset angular-bitbucket
```

默认情况下，changelog 预设设置为[`angular`](https://github.com/conventional-changelog/conventional-changelog/tree/master/packages/conventional-changelog-angular#angular-convention)。在某些情况下，您可能需要使用另一个预置或自定义。

##### `--conventional-commits`

```sh
lerna version --conventional-commits
```

当使用这个标志运行时，lerna 版本将使用[传统的提交规范/Conventional Commits Specification](https://conventionalcommits.org/)来[确定版本]((https://github.com/conventional-changelog/conventional-changelog/tree/master/packages/conventional-recommended-bump)并生成[CHANGELOG.md](https://github.com/conventional-changelog/conventional-changelog/tree/master/packages/conventional-changelog-cli)。

传入 [`--no-changelog`](#--no-changelog) 将阻止生成或者更新`CHANGELOG.md`.

##### `--conventional-graduate`

```sh
lerna version --conventional-commits --conventional-graduate=package-2,package-4

# force all prerelease packages to be graduated
lerna version --conventional-commits --conventional-graduate
```

但使用该标记时，`lerna vesion`将升级指定的 package（用逗号分隔）或者使用`*`指定全部 package。和`--force-publish`很像，无论当前的 HEAD 是否发布，该命令都会起作用，任何没有预发布的 package 将会被忽略。如果未指定的包(如果指定了包)或未预先发布的包发生了更改，那么这些包将按照它们通常使用的`--conventional-commits`进行版本控制。

"升级"一个包意味着将一个预发布的包升级为发布版本，例如`package-1@1.0.0-alpha.0 => package-1@1.0.0`。

> 注意: 当指定包时，指定包的依赖项将被释放，但不会被“升级”。必须和`--conventional-commits`一起使用

##### `--conventional-prerelease`

```sh
lerna version --conventional-commits --conventional-prerelease=package-2,package-4

# force all changed packages to be prereleased
lerna version --conventional-commits --conventional-prerelease
```

当使用该标记时，`lerna version`将会以预发布的版本发布指定的 package（用逗号分隔）或者使用`*`指定全部 package。

##### `--create-release <type>`

```sh
lerna version --conventional-commits --create-release github
lerna version --conventional-commits --create-release gitlab
```

当使用此标志时，`lerna version`会基于改变的 package 创建一个官方正式的 GitHub 或 GitLab 版本记录。需要传递`--conventional-commits`去创建 changlog。

GithuB 认证，以下环境变量需要被定义。

- `GH_TOKEN` (required) - Your GitHub authentication token (under Settings > Developer settings > Personal access tokens).
- `GHE_API_URL` - When using GitHub Enterprise, an absolute URL to the API.
- `GHE_VERSION` - When using GitHub Enterprise, the currently installed GHE version. [Supports the following versions](https://github.com/octokit/plugin-enterprise-rest.js).

GitLab 认证，以下环境变量需要被定义。

- `GL_TOKEN` (required) - Your GitLab authentication token (under User Settings > Access Tokens).
- `GL_API_URL` - An absolute URL to the API, including the version. (Default: https://gitlab.com/api/v4)

> 注意: 不允许和[`--no-changelog`](#--no-changelog)一起使用

这个选项也可以在`lerna.json`中配置：

```json
{
  "changelogPreset": "angular"
}
```

If the preset exports a builder function (e.g. `conventional-changelog-conventionalcommits`), you can specify the [preset configuration](https://github.com/conventional-changelog/conventional-changelog-config-spec) too:

```json
{
  "changelogPreset": {
    "name": "conventionalcommits",
    "issueUrlFormat": "{{host}}/{{owner}}/{{repository}}/issues/{{id}}"
  }
}
```

![lerna version --conventional-commits --create-release github](./images/WX20200907-163903@2x.png)

##### `--exact`

```sh
lerna version --exact
```

##### `--force-publish`

```sh
lerna version --force-publish=package-2,package-4

# force all packages to be versioned
lerna version --force-publish
```

强制更新版本

> 这个操作将跳过`lerna changed`检查，即便 package 没有做任何变更也会更新版本

##### `--git-remote <name>`

```sh
lerna version --git-remote upstream
```

将本地`commit`push 到指定的远程残酷，默认是`origin`

##### `--ignore-changes`

变更检测时忽略的文件

```sh
lerna version --ignore-changes '**/*.md' '**/__tests__/**'
```

建议在根目录`lerna.json`中配置：

```json
{
  "ignoreChanges": ["**/__fixtures__/**", "**/__tests__/**", "**/*.md"]
}
```

`--no-ignore-changes` 禁止任何现有的忽略配置：

##### `--ignore-scripts`

禁止[lifecycle scripts](#lifecycle-scripts)

##### `--include-merged-tags`

```sh
lerna version --include-merged-tags
```

##### `--message <msg>`

`-m`别名，等价于`git commit -m `

```sh
lerna version -m "chore(release): publish %s"
# commit message = "chore(release): publish v1.0.0"

lerna version -m "chore(release): publish %v"
# commit message = "chore(release): publish 1.0.0"

# When versioning packages independently, no placeholders are replaced
lerna version -m "chore(release): publish"
# commit message = "chore(release): publish
#
# - package-1@3.0.1
# - package-2@1.5.4"
```

也可以在`lerna.json`配置：

```json
{
  "command": {
    "version": {
      "message": "chore(release): publish %s"
    }
  }
}
```

##### `--no-changelog`

```sh
lerna version --conventional-commits --no-changelog
```

不生成`CHANGELOG.md`。

> 注意：不可以和[`--create-release`](#--create-release-type)一起使用

##### `--no-commit-hooks`

默认情况下，`lerna version`会运行`git commit hooks`。使用该标记，阻止`git commit hooks`运行。

##### `--no-git-tag-version`

默认情况下，`lerna version` 会提交变更到`package.json`文件，并打标签。使用该标记会阻止该默认行为。

##### `--no-granular-pathspec`

默认情况下，在创建版本的过程中，会执行`git add -- packages/*/package.json`操作。

也可以更改默认行为，提交除了`package.json`以外的信息，前提是必须做好敏感数据的保护。

```json
// leran.json
{
  "version": "independent",
  "granularPathspec": false
}
```

##### `--no-private`

排除`private:true`的 package

##### `--no-push`

By default, `lerna version` will push the committed and tagged changes to the configured [git remote](#--git-remote-name).
Pass `--no-push` to disable this behavior.

##### `--preid`

```sh
lerna version prerelease
# uses the next semantic prerelease version, e.g.
# 1.0.0 => 1.0.1-alpha.0

lerna version prepatch --preid next
# uses the next semantic prerelease version with a specific prerelease identifier, e.g.
# 1.0.0 => 1.0.1-next.0
```

版本语义化

##### `--sign-git-commit`

`npm version` [option](https://docs.npmjs.com/misc/config#sign-git-commit)

##### `--sign-git-tag`

`npm version` [option](https://docs.npmjs.com/misc/config#sign-git-tag)

##### `--force-git-tag`

取代已存在的`tag`

##### `--tag-version-prefix`

自定义版本前缀。默认为`v`

```bash
# locally
lerna version --tag-version-prefix=''
# on ci
lerna publish from-git --tag-version-prefix=''
```

##### `--yes`

```sh
lerna version --yes
# skips `Are you sure you want to publish these packages?`
```

跳过所有提示

#### 生成更新日志`CHANGELOG.md`

如果你在使用多包存储一段时间后，开始使用[`--conventional-commits`](#--conventional-commits)标签，你也可以使用[`conventional-changelog-cli`](https://github.com/conventional-changelog/conventional-changelog/tree/master/packages/conventional-changelog-cli#readme) 和 [`lerna exec`](https://github.com/lerna/lerna/tree/master/commands/exec#readme)为之前的版本创建 changelog：

```bash
# Lerna does not actually use conventional-changelog-cli, so you need to install it temporarily
npm i -D conventional-changelog-cli
# Documentation: `npx conventional-changelog --help`

# fixed versioning (default)
# run in root, then leaves
npx conventional-changelog --preset angular --release-count 0 --outfile ./CHANGELOG.md --verbose
npx lerna exec --concurrency 1 --stream -- 'conventional-changelog --preset angular --release-count 0 --commit-path $PWD --pkg $PWD/package.json --outfile $PWD/CHANGELOG.md --verbose'

# independent versioning
# (no root changelog)
npx lerna exec --concurrency 1 --stream -- 'conventional-changelog --preset angular --release-count 0 --commit-path $PWD --pkg $PWD/package.json --outfile $PWD/CHANGELOG.md --verbose --lerna-package $LERNA_PACKAGE_NAME'
```

If you use a custom [`--changelog-preset`](#--changelog-preset), you should change `--preset` value accordingly in the example above.

#### Lifecycle Scripts

```js
// preversion:  Run BEFORE bumping the package version.
// version:     Run AFTER bumping the package version, but BEFORE commit.
// postversion: Run AFTER bumping the package version, and AFTER commit.
```

Lerna will run [npm lifecycle scripts](https://docs.npmjs.com/misc/scripts#description) during `lerna version` in the following order:

1. Detect changed packages, choose version bump(s)
2. Run `preversion` lifecycle in root
3. For each changed package, in topological order (all dependencies before dependents):
   1. Run `preversion` lifecycle
   2. Update version in package.json
   3. Run `version` lifecycle
4. Run `version` lifecycle in root
5. Add changed files to index, if [enabled](#--no-git-tag-version)
6. Create commit and tag(s), if [enabled](#--no-git-tag-version)
7. For each changed package, in _lexical_ order (alphabetical according to directory structure):
   1. Run `postversion` lifecycle
8. Run `postversion` lifecycle in root
9. Push commit and tag(s) to remote, if [enabled](#--no-push)
10. Create release, if [enabled](#--create-release-type)

### [lerna publish](https://github.com/lerna/lerna/tree/master/commands/publish#readme)

```sh
lerna publish              # 发布自上次发版依赖更新的packages
lerna publish from-git     # 显示的发布在当前提交中打了tag的packages
lerna publish from-package # 显示的发布当前版本在注册表中（registry）不存在的packages（之前没有发布到npm上）
```

运行时，该命令执行以下操作之一：

- 发布自上次发版依赖更新的 packages（背后调用`lerna version`判断）
  - 这是 2.x 版本遗留的表现
- 显示的发布在当前提交中打了 tag 的 packages
- 显示的发布在最新的提交中当前版本在注册表中（registry）不存在的 packages（之前没有发布到 npm 上）
- 发布在之前提交中未版本化的进行过金丝雀部署的 packages([`canary release`](https://www.dazhuanlan.com/2019/09/27/5d8d956b87584/))

> Lerna 无法发布私有的 packcage(`"private":true`)

在所有发布操作期间，适当的生命周期脚本（[`lifecycle scripts`](https://github.com/lerna/lerna/tree/master/commands/publish#lifecycle-scripts)）在根目录和每个包中被调用(除非被`--ignore-scripts`禁用)。

#### Positionals

- bump `from-git`

除了`lerna version`支持的 semver 关键字之外，`lerna publish`还支持`from-git`关键字。这将识别`lerna version`标记的包，并将它们发布到 npm。这在 CI 场景中非常有用，在这种场景中，您希望手动增加版本，但要通过自动化过程一致地发布包内容本身

- bump `from-package`

与`from-git`关键字相似，除了要发布的软件包列表是通过检查每个`package.json`并确定注册表中是否没有任何软件包版本来确定的。 注册表中不存在的任何版本都将被发布。 当先前的`lerna publish`未能将所有程序包发布到注册表时，此功能很有用。

#### Command Options

`lerna publish`除了支持一下选项外，还支持`lerna version`的所有选项：

- [`--canary`](#--canary)
- [`--contents <dir>`](#--contents-dir)
- [`--dist-tag <tag>`](#--dist-tag-tag)
- [`--git-head <sha>`](#--git-head-sha)
- [`--graph-type <all|dependencies>`](#--graph-type-alldependencies)
- [`--ignore-scripts`](#--ignore-scripts)
- [`--ignore-prepublish`](#--ignore-prepublish)
- [`--legacy-auth`](#--legacy-auth)
- [`--no-git-reset`](#--no-git-reset)
- [`--no-granular-pathspec`](#--no-granular-pathspec)
- [`--no-verify-access`](#--no-verify-access)
- [`--otp`](#--otp)
- [`--preid`](#--preid)
- [`--pre-dist-tag <tag>`](#--pre-dist-tag-tag)
- [`--registry <url>`](#--registry-url)
- [`--tag-version-prefix`](#--tag-version-prefix)
- [`--temp-tag`](#--temp-tag)
- [`--yes`](#--yes)

##### `--canary`

```sh
lerna publish --canary
# 1.0.0 => 1.0.1-alpha.0+${SHA} of packages changed since the previous commit
# a subsequent canary publish will yield 1.0.1-alpha.1+${SHA}, etc

lerna publish --canary --preid beta
# 1.0.0 => 1.0.1-beta.0+${SHA}

# The following are equivalent:
lerna publish --canary minor
lerna publish --canary preminor
# 1.0.0 => 1.1.0-alpha.0+${SHA}
```

针对最近一次提交发生改变的 package，做更精细的版本控制。类似于金丝雀部署，构建生产环境的容错测试。如果是统一的版本控制，其他 package 版本号不做升级，只针对变更的 package 做精准调试。

![lerna publish --canary](./images/1598864025009.jpg)

##### `--contents <dir>`

子目录发布。子目录中必须包含 package.json。

```sh
lerna publish --contents dist
# publish the "dist" subfolder of every Lerna-managed leaf package
```

##### `--dist-tag <tag>`

```sh
lerna publish --dist-tag custom-tag
```

自定义 npm[发布标签](https://docs.npmjs.com/cli/dist-tag)。默认是`latest`

该选项可以用来定义[`prerelease`](http://carrot.is/coding/npm_prerelease) 或者 `beta` 版本

> **注意**: `npm install my-package` 默认安装的是`latest`版本.
> 安装其他版本 `npm install my-package@prerelease`.

![lerna publish --canary](./images/WX20200831-182407@2x.png)

##### `--git-head <sha>`

只可以和`from-package`配合使用，根据指定的`git <sha>`发布

[也可以使用环境变量指定](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html)

```sh
lerna publish from-package --git-head ${CODEBUILD_RESOLVED_SOURCE_VERSION}
```

##### `--graph-type <all|dependencies>`

`npm`上构建`package dependencies`所采用的方式，默认是`dependencies`，只列出`dependencies`。`all`会列出`dependencies` 和 `devDependencies`

```sh
lerna publish --graph-type all
```

也可以通过`lerna.json`配置:

```json
{
  "command": {
    "publish": {
      "graphType": "all"
    }
  }
}
```

![lerna publish --graph-type all](./images/WX20200831-185905@2x.png)

##### `--ignore-scripts`

关闭`npm脚本生命周期事件`的触发

##### `--ignore-prepublish`

近关闭`npm脚本生命周期 prepublish事件`的触发

##### `--legacy-auth`

发布前的身份验证

```sh
lerna publish --legacy-auth aGk6bW9t
```

##### `--no-git-reset`

默认情况下，`lerna publish`会把暂存区内容全部提交。即`lerna publish`发布时更改了本地 package 中的 version，也一并提交到 git。

![lerna publish](./images/WX20200903-183002@2x.png)

未避免上述情况发生，可以使用`--no-git-reset`。这对作为管道配置`--canary`使用时非常有用。例如，已经改变的`package.json`的版本号可能会在下一步操作所用到（例如 Docker builds）。

```sh
lerna publish --no-git-reset
```

##### `--no-granular-pathspec`

By default, `lerna publish` will attempt (if enabled) to `git checkout` _only_ the leaf package manifests that are temporarily modified during the publishing process. This yields the equivalent of `git checkout -- packages/*/package.json`, but tailored to _exactly_ what changed.

If you **know** you need different behavior, you'll understand: Pass `--no-granular-pathspec` to make the git command _literally_ `git checkout -- .`. By opting into this [pathspec](https://git-scm.com/docs/gitglossary#Documentation/gitglossary.txt-aiddefpathspecapathspec), you must have all intentionally unversioned content properly ignored.

This option makes the most sense configured in lerna.json, as you really don't want to mess it up:

```json
{
  "version": "independent",
  "granularPathspec": false
}
```

The root-level configuration is intentional, as this also covers the [identically-named option in `lerna version`](https://github.com/lerna/lerna/tree/master/commands/version#--no-granular-pathspec).

##### `--no-verify-access`

默认情况下`lerna`会验证已登录用户对即将发布的 package 的权限。使用此标记将会阻止该默认行为。

如果你正在使用第三方的不支持`npm access ls-packages`的 npm 库，需要使用该标记。或者在`lerna.json`中设置`command.publish.verifyAccess`为`false`。

> 谨慎使用

##### `--otp`

当发布需要双重认证的 package 时，需要指定[一次性密码](https://docs.npmjs.com/about-two-factor-authentication)

```sh
lerna publish --otp 123456
```

当开启[npm 双重认证](https://docs.npmjs.com/about-two-factor-authentication)后，可以通过配置对 account 和 npm 操作的进行二次验证。需要 npm 版本大于`5.5.0`

[验证工具](https://authy.com/guides/npm/)

![Two-factor authentication](./images/WX20200902-170413@2x.png)

> 密码的有效时长为 30s，过期后需要重新输入验证

##### `--preid`

Unlike the `lerna version` option of the same name, this option only applies to [`--canary`](#--canary) version calculation.

和 [`--canary`](#--canary) 配合使用，指定语义化版本

```sh
lerna publish --canary
# uses the next semantic prerelease version, e.g.
# 1.0.0 => 1.0.1-alpha.0

lerna publish --canary --preid next
# uses the next semantic prerelease version with a specific prerelease identifier, e.g.
# 1.0.0 => 1.0.1-next.0
```

当使用该标记时，`lerna publish --canary` 将增量改变 `premajor`, `preminor`, `prepatch`, 或者 `prerelease` 的语义化版本。
[语义化版本(prerelease identifier)](http://semver.org/#spec-item-9)

##### `--pre-dist-tag <tag>`

```sh
lerna publish --pre-dist-tag next
```

效果和[`--dist-tag`](#--dist-tag-tag)一样。只适用于发布的预发布版本。

##### `--registry <url>`

##### `--tag-version-prefix`

更改标签前缀

如果分割`lerna version`和`lerna publish`，需要都设置一遍：

```bash
# locally
lerna version --tag-version-prefix=''

# on ci
lerna publish from-git --tag-version-prefix=''
```

也可以在`lerna.json`中配置该属性，效果等同于上面两条命令：

```json
{
  "tagVersionPrefix": "",
  "packages": ["packages/*"],
  "version": "independent"
}
```

##### `--temp-tag`

当传递时，这个标志将改变默认的发布过程，首先将所有更改过的包发布到一个临时的 dis tag (' lerna-temp ')中，然后将新版本移动到['--dist-tag '](#——dist-tag-tag)(默认为' latest ')配置的 dist-tag 中。

这通常是没有必要的，因为 Lerna 在默认情况下会按照拓扑顺序(所有依赖先于依赖)发布包

##### `--yes`

```sh
lerna publish --canary --yes
# skips `Are you sure you want to publish the above changes?`
```

跳过所有的确认提示

在[Continuous integration (CI)](https://en.wikipedia.org/wiki/Continuous_integration)很有用，自动回答发布时的确认提示

#### 每个 package 的配置

每个 package 可以通过更改[`publishConfig`](https://docs.npmjs.com/files/package.json#publishconfig)，来改变发布时的一些行为。

##### `publishConfig.access`

当发布一个`scope`的 package(e.g., `@mycompany/rocks`)时，必须设置[`access`](https://docs.npmjs.com/misc/config#access)：

```json
  "publishConfig": {
    "access": "public"
  }
```

- 如果在没有使用 scope 的 package 中使用该属性，将失败
- 如果你希望保持一个 scope 的 package 为私有(i.e., `"restricted"`)，那么就不需要设置

  注意，这与在包中设置`"private":true`不一样;如果设置了`private`字段，那么在任何情况下都不会发布该包。

##### `publishConfig.registry`

```json
  "publishConfig": {
    "registry": "http://my-awesome-registry.com/"
  }
```

- 也可以通过`--registry`或者在 lerna.json 中设置`command.publish.registry`进行全局控制

##### `publishConfig.tag`

自定义该包发布时的标签[`tag`](https://docs.npmjs.com/misc/config#tag):

```json
  "publishConfig": {
    "tag": "flippin-sweet"
  }
```

- [`--dist-tag`](#--dist-tag-tag)将覆盖每个 package 中的值
- 在使用[--canary]时该值将被忽略

##### `publishConfig.directory`

非标准字段，自定义发布的文件

```json
  "publishConfig": {
    "directory": "dist"
  }
```

#### npm 脚本生命周期

```js
// prepublish:      Run BEFORE the package is packed and published.
// prepare:         Run BEFORE the package is packed and published, AFTER prepublish, BEFORE prepublishOnly.
// prepublishOnly:  Run BEFORE the package is packed and published, ONLY on npm publish.
// prepack:     Run BEFORE a tarball is packed.
// postpack:    Run AFTER the tarball has been generated and moved to its final destination.
// publish:     Run AFTER the package is published.
// postpublish: Run AFTER the package is published.
```

`lerna publish`执行时，按如下顺序调用[npm 脚本生命周期](https://docs.npmjs.com/misc/scripts#description)：

1. 如果采用隐式版本管理，则运行所有 [version lifecycle scripts](https://github.com/lerna/lerna/tree/master/commands/version#lifecycle-scripts)。
2. Run `prepublish` lifecycle in root, if [enabled](#--ignore-prepublish)
3. Run `prepare` lifecycle in root
4. Run `prepublishOnly` lifecycle in root
5. Run `prepack` lifecycle in root
6. For each changed package, in topological order (all dependencies before dependents):
   1. Run `prepublish` lifecycle, if [enabled](#--ignore-prepublish)
   2. Run `prepare` lifecycle
   3. Run `prepublishOnly` lifecycle
   4. Run `prepack` lifecycle
   5. Create package tarball in temp directory via [JS API](https://github.com/lerna/lerna/tree/master/utils/pack-directory#readme)
   6. Run `postpack` lifecycle
7. Run `postpack` lifecycle in root
8. For each changed package, in topological order (all dependencies before dependents):
   1. Publish package to configured [registry](#--registry-url) via [JS API](https://github.com/lerna/lerna/tree/master/utils/npm-publish#readme)
   2. Run `publish` lifecycle
   3. Run `postpublish` lifecycle
9. Run `publish` lifecycle in root
   - To avoid recursive calls, don't use this root lifecycle to run `lerna publish`
10. Run `postpublish` lifecycle in root
11. Update temporary dist-tag to latest, if [enabled](#--temp-tag)

### 过滤选项

- `--scope` 为匹配到的 package 安装依赖 [字符串]
- `--ignore` 和上面正相反 [字符串]
- `--no-private` 排除 private 的 packcage
- `--since` 包含从指定的[ref]依赖改变的 packages，如果没有[ref]，默认是最近的 tag
- `--exclude-dependents` 当使用—since 运行命令时，排除所有传递依赖项，覆盖默认的“changed”算法 [布尔]
- `--include-dependents` 启动命令式包含所有传递的依赖项，无视 --scope, --ignore, or --since [布尔]
- `--include-dependencies` 启动命令式包含所有传递的依赖项，无视 --scope, --ignore, or --since [布尔]
- `--include-merged-tags` 在使用—since 运行命令时，包含来自合并分支的标记 [布尔]

### 全局选项

- `--loglevel` 打印日志的级别 [字符串] [默认值: info][silly, versbose, info, success, warn, error, slient]
- `--concurrency` 并行任务时启动的进程数目 [数字] [默认值: 4]
- `--reject-cycles` 如果 package 之间相互依赖，则失败 [布尔]
- `--no-progress` 关闭进程进度条 [布尔]
- `--no-sort` 不遵循拓扑排序 [布尔]
- `--max-buffer` 设置子命令执行的 buffer(以字节为单位) [数字]
- `-h, --help ` 显示帮助信息
- `-v, --version` 显示版本信息

## Concept

### lerna.json

```json
{
  "version": "1.1.3", // 版本
  "npmClient": "npm", // npm客户端
  "command": {
    "publish": {
      "ignoreChanges": ["ignored-file", "*.md"], // 发布检测时忽略的文件
      "message": "chore(release): publish", // 发布时 tag标记的版本信息
      "registry": "https://npm.pkg.github.com" // 源
    },
    "bootstrap": {
      "ignore": "component-*", // bootstrap时忽略的文件
      "npmClientArgs": ["--no-package-lock"], // 命令行参数
    },
    "version": {
      "allowBranch": [ // 允许运行lerna version的分支
        "master",
        "feature/*"
      ],
      "message": "chore(release): publish %s" // 创建版本时 tag标记的版本信息
    }
  },
  "ignoreChanges": [
    "**/__fixtures__/**",
    "**/__tests__/**",
    "**/*.md"
  ],
  "packages": ["packages/*"],// package位置
}
```

## Wizard

这里推荐一个新手引导工具[lerna-wizard](https://github.com/szarouski/lerna-wizard)

![lerna-wizard](./images/WX20200909-152126@2x.png)