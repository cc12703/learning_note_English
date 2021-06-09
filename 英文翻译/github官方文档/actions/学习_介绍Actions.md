

# 学习_介绍Actions

* [原始文档](https://docs.github.com/en/actions/learn-github-actions/introduction-to-github-actions)


## 概述
* Actions用于帮助自动运行软件开发周期中的涉及到的任务
* Actions是事件驱动的，意味着在发生特定事件后你可以运行一系列命令
    * 例子：每次给一个仓库创建一个pull请求时，你可以自动运行一条命令来执行测试脚本

### 图示
![](https://gitee.com/cc12703/figurebed/raw/master/img/20210609192632.png)
* 一个**事件**自动触发一个包含了一个**工作**的**工作流**
* 工作使用**步骤**来控制**动作**的执行顺序
* 这些**动作**是一些自动运行软件测试的**命令**



## 组件

### 图示
![](https://gitee.com/cc12703/figurebed/raw/master/img/20210609193324.png)

### 工作流 - Worlflow
* 一个工作流就是加入到你仓库中的一个自动化过程
* 工作流由一个、多个被一个事件触发、调度的任务组成
* 工作流可以用于构建、测试、打包、发布、部署一个Github上的项目

### 事件 - Event
* 一个事件是一个触发工作流的特定的活动
* 例子：活动都来自于Github, 某人给一个仓库推送一个提交、一个问题或pull请求被创建
* 对于外部事件，你可以使用**仓库分发webhook**来触发工作流

### 任务 - Job
* 一个任务是一组在相同运行器上执行的步骤
* 默认情况下，一个拥有多个任务的工作流会并行执行所有任务
    * 但是也可以配置工作流串行地执行任务
* 例子：一个工作流包含两个串行的任务，一个构建代码、一个测试代码，测试任务要依赖于构建任务的执行状态。如果构建失败则测试不运行

### 步骤 - Step
* 一个步骤是任务中一个可以运行命令的作业
* 一个步骤可以是一个动作、或者一个shell命令
* 任务中的每个步骤都在相同的运行器上执行，任务中的动作都彼此共享数据

### 动作 - Action
* 动作是一些独立的命令，在创建任务时会合并进步骤中
* 动作是工作流中最小的可移植构建块
* 你可以创建自己的动作，或使用Github创建的动作
* 为了在工作流中使用动作，必须将其包含进行一个步骤中

### 运行器 - Runner
* 一个运行器是一个安装着**GitHub动作运行器应用**的服务器
* 你可以使用GitHub提供的运行器，或则自己搭建
* 一个运行器会监听所有有效的任务，一次运行一个任务，报告执行进度、日志，将结果上报GitHub
* GitHub提供的运行器基于Ubuntu,Windows,MacOS进行构建
    * 工作流中的每个任务都运行在一个崭新的虚拟环境中



## 例子

### 概述
* Actions使用yaml语法来定义事件、任务和步骤
* 这些yaml文件存储在代码仓库的`.github/workflows`目录中

### 步骤
1. 在仓库中，创建目录`.github/workflows`
1. 在目录中，创建文件`laern-github-actions.yml`
1. 在文件中，添加以下代码
    ```yml
    name: learn-github-actions
    on: [push]
    jobs:
        check-bats-version:
            runs-on: ubuntu-latest
            steps:
                - uses: actions/checkout@v2
                - uses: actions/setup-node@v1
                - run: npm install -g bats
                - run: bats -v
    ```
1. 提交这些变更，并将其push进行GitHub仓库


### 代码解释
* `name: learn-github-actions`
    * 可选，定义工作流的名字，显示在GitHub仓库的Actions页面中
* `on: [push]`
    * 指定自动触发工作流的事件，例子中使用了push事件
    * 每次有人将变动推送到仓库时，任务就会执行
    * 你可以设置工作流只运行在特定的分支、路径、tag上
* `jobs:`
    * 用于将工作流中的所有任务分组在一起
* `check-bats-versino:`
    * 定义一个任务的名字
* `runs-on: ubuntu-latest`
    * 配置任务运行在Ubuntu运行器中，意味着任务运行在GitHub支持的虚拟机上
* `steps:`
    * 将check-bats-version任务中的所有步骤分组在一起
    * 其包含的每个项都是一个分离的动作或shell命令
* `- uses: actions/checkout@v2`
    * users关键字告诉任务去获取名为actions/checkout动作的v2版本
    * 这个动作会从你的仓库中签出代码并下载到运行器中，允许对你的代码执行动作
    * 当任何时候你想运行针对仓库代码的工作流，或使用定义在仓库中的动作时，你必须要使用签出动作
* `- uses: actions/setup-node@v1`
    * 该动作在运行器中安装node软件包，可以允许你执行npm命令
* `- run: npm install -g bats`
    * run关键字告诉任务在运行器中执行一条命令
    * 例子中使用npm安装bats软件包
* `- run: bats -v`
    * 最后执行bats命令，打印出软件版本号

### 可视化
![](https://gitee.com/cc12703/figurebed/raw/master/img/20210610101158.png)
