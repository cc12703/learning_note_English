

# 学习_查找和定制Actions

* [原始文档](https://docs.github.com/en/actions/learn-github-actions/finding-and-customizing-actions)


## 概述
* 工作流中使用的动作定义在以下几个地方
    * 公开的仓库
    * 你的工作流引用动作的相同仓库
    * 一个发布在DockerHub上的Docker镜像
* GitHub市场是一个寻找由GitHub社区创建的动作的地方，[进入市场](https://github.com/marketplace?type=actions)


## 在编辑器中浏览市场中的Actions
* 你可以在你自己仓库的工作流编辑器中直接搜索和浏览Actions
* 在侧栏中你可以搜索特定的Actions，查看Actions的特点，浏览特点分类
* 你也可以查看到该Actions有多少点赞数

### 操作步骤
1. 在仓库中，浏览要编辑的工作流文件
1. 在右上角的文件视图区域，点击以下按键打开工作流编辑器
    ![](https://gitee.com/cc12703/figurebed/raw/master/img/20210610103048.png)
1. 在编辑器的右边，使用GitHub市场侧栏可以浏览Actions
    ![](https://gitee.com/cc12703/figurebed/raw/master/img/20210610103230.png)


## 为定制Actions使用发布管理
* 社区Action的创建者可以选择使用标签, 分支，SHA值来管理Action的发布
* 类似于增加任何依赖，你可以基于让你舒适地自动更新Action的方式来给你使用的Action标记上版本信息
* 你可以在工作流文件中指定Action的版本，查看Action的文档，可以知道该Action选择的发布管理方案并确定要使用的方式


### 使用标签
* 对于自主决定何时切换主、次版本号而言，标签是非常有用的
* 但是标签是短暂的，会被维护者移除或删除
* 示例
    ```yml
    steps:
        - uses: actions/javascript-action@v1.0.1
    ```


### 使用SHA值
* 如果需要更可靠的版本信息，你可以使用SHA值
* SHA值是不可变的，比标签和分支都更可靠
* 该方法的问题是无法自动更新Action（包括重要的BUG修复、安全更新）
* 你必须要使用一个完整的SHA值，而不是一个缩写值
* 示例
    ```yml
    steps:
        - uses: actions/javascript-action@172239021f7ba04fe7327647b213799853a9eb89
    ```

### 使用分支
* 通过指定Action的目标分支，你可以运行当前分支上的版本
* 当对该分支的更新包突发变化时，该方法会产生问题
* 示例
    ```yml
    steps:
        - uses: actions/javascript-action@main
    ```


## Action的输入和输出
* 一个动作会接收输入并产生输出
    * 例子，一个动作也许会要求你输入一个文件路径、一个标签的名字、其他数据
* 通过查看仓库根目录下的action.yml、action.yaml文件可以找到动作的输入信息和输出信息

### 示例
```yml
name: 'Example'
description: 'Receives file and generates output'
inputs:
  file-path:  # id of input
    description: "Path to test script"
    required: true
    default: 'test-file.js'
outputs:
  results-file: # id of output
    description: "Path to results file"
```
* inputs关键字定义了需要的输入信息
    * 例子中为file-path
* outpus关键字定义了输出信息
    * 例子中为results-file


## 引用同仓库中的Action
* 如果一个动作定义在和使用它的工作流在一个仓库中，你可以使用语法`{owner}/{repo}@{ref}`或`./path/to/dir`来引用它

### 示例
仓库目录结构
```
|-- hello-world (repository)
|   |__ .github
|       └── workflows
|           └── my-first-workflow.yml
|       └── actions
|           |__ hello-world-action
|               └── action.yml
```

工作流文件
```yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # 该步骤从仓库中签出代码
      - uses: actions/checkout@v2
      # 该步骤引用仓库中的动作
      - uses: ./.github/actions/hello-world-action
```

## 引用DockerHub中的容器
* 如果一个动作定义在DockerHub中的一个已发布的Docker镜像中，你可以使用语法`docker://{image}:{tag}`来引用它
* 为了保护你的代码和数据，我们强烈推荐你校验Docker镜像的完整性
* 示例
    ```yml
    jobs:
    my_first_job:
        steps:
        - name: My first step
            uses: docker://alpine:3.8
    ```