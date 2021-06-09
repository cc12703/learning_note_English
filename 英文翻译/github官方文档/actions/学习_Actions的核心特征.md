

# 学习_Actions的核心特征

* [原始文档](https://docs.github.com/en/actions/learn-github-actions/essential-features-of-github-actions)



## 概述
* Actions允许你定制你的工作流，来满足你应用和团队的特殊需求
* 本节将讨论一些核心特征：使用变量、运行脚本、在任务间共享数据和构件


## 使用变量
* Actions包括有一些默认的环境变量
* 如果你需要使用定制的环境变量，可以在工作流文件中进行设置

### 示例
```yml
jobs:
  example-job:
      steps:
        - name: Connect to PostgreSQL
          run: node client.js
          env:
            POSTGRES_HOST: postgres
            POSTGRES_PORT: 5432
```
* 示例中创建了两个定制变量POSTGRES_HOST和POSTGRES_PORT
* 这些变量在`node client.js`中有效


## 添加脚本
* 你可以使用动作来运行脚本和shell命令，它们会在指定的运行器总被执行

### 示例1
```yml
jobs:
  example-job:
    steps:
      - run: npm install -g bats
```

### 示例2
```yml
jobs:
  example-job:
    steps:
      - name: Run build script
        run: ./.github/scripts/build.sh
        shell: bash
```

## 在任务间共享数据
* 如果你想生成一个文件来供相同工作流中的其他任务使用、或者想保存文件以供后面使用，你可以在GitHub将它们保存为**构件**
* 构件是当你构件、测试代码时生成的一些文件
    * 例子：构件包括二进制文件、包文件、测试结果、截屏、日志文件
* 构件是在工作流运行时被关联的，它们可以被创建、在其他任务中被使用

### 示例1
```yml
jobs:
  example-job:
    name: Save output
    steps:
      - shell: bash
        run: |
          expr 1 + 1 > output.log
      - name: Upload output file
        uses: actions/upload-artifact@v2
        with:
          name: output-log-file
          path: output.log
```
* 例子创建了一个文件，并将其作为构件进行上传


### 示例2
```yml
jobs:
  example-job:
    steps:
      - name: Download a single artifact
        uses: actions/download-artifact@v2
        with:
          name: output-log-file
```
* 可以通过使用`actions/download-artifact`来在其他工作流中下载构件