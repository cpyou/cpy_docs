# 实验介绍

本实验我们将会介绍 Jenkins 里的 Pipeline 的语法和使用，以及如何开发一个简单的 Pipeline 并在 Jenkins 中使用。

#### 知识点

- Pipeline 语法
- Jenkinsfile
- 共享库
- 共享库配置

# Pipeline 介绍

Pipeline 就是一套运行在 Jenkins 上的工作流框架，将原来独立运行于单个或者多个节点的任务连接起来，实现单个任务难以完成的复杂流程编排和可视化的工作。 Jenkins Pipeline 有几个核心概念：

- Node：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行环境，比如我们之前动态运行的 Jenkins Slave 就是一个 Node 节点
- Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如：Build、Test、Deploy，Stage 是一个逻辑分组的概念，可以跨多个 Node
- Step：步骤，Step 是最基本的操作单元，可以是打印一句话，也可以是构建一个 Docker 镜像，由各类 Jenkins 插件提供，比如命令：`sh 'make'`，就相当于我们平时 shell 终端中执行 make 命令一样。

Pipeline 的使用：

- Pipeline 脚本是由 Groovy 语言实现的
- Pipeline 支持两种语法：Declarative(声明式)和 Scripted Pipeline(脚本式)语法
- Pipeline 也有两种创建方法：可以直接在 Jenkins 的 Web UI 界面中输入脚本；也可以通过创建一个 Jenkinsfile 脚本文件放入项目源码库中
- 一般我们都推荐在 Jenkins 中直接从源代码控制(SCMD)中直接载入 Jenkinsfile Pipeline 这种方法

# Pipeline 语法简介

上面介绍过 Pipeline 的几个核心概念：Node、Stage、Step，这里简单介绍一下它们的实际语法。

# Node

节点是一个机器，可以是 Jenkins 的 master 节点也可以是 slave 节点。

```groovy
pipeline {
    agent any
    stages{
    //
    }
}
```

这表示可以使用任意 agent 来运行 Pipeline。

agent 除了 any 之外还有 none 和 label，其中 none 表示没有指定 agent，label 表示使用标签来选择 agent，如下：

```groovy
pipeline {
    agent {
        label 'jenkins-jnlp'
    }
    stages{
    //
    }
}
```

# Stage

stage 定义了在整个流水线的执行任务的概念性的不同的阶段。例如： GetCode、Build、Test、Deploy、CodeScan 每个阶段，如下：

```groovy
pipeline{
    agent any
    stages{
        stage("GetCode"){
        //代码区
        }
        stage("Build"){
        // 代码区
        }
    }
}
```

# Step

step 是每个阶段中要执行的每个步骤，例如：

```groovy
pipeline{
    agent any
    stages{
        stage("GetCode"){
            steps{
                sh "ls "    //step
            }
        }
    }
}
```

# 创建一条简单的 Pipeline

选择 **新建任务**，输入任务名称:`dev-simple-pipline`，选择 **流水线**(pipline)，如下：



然后拉到 **流水线** 配置处，输入以下内容：

```groovy
pipeline{
    agent any
    stages{
        stage("GetCode"){
            steps{
                sh "echo 'Get Code' "    //step
            }
        }

        stage("build"){
            steps{
                sh "echo 'Build Code' "    //step
            }
        }

        stage("push"){
            steps{
                sh "echo 'Build Image' "    //step
            }
        }

        stage("deploy"){
            steps{
                sh "echo 'Deploy app' "    //step
            }
        }
    }
}
```

结果如下：

然后点击 **保存**，然后在项目里选择 **立即构建**，查看构建效果，如下：

在 **阶段视图** 里可以看到具体的步骤以及每个步骤所花费的时间。

在 **构建历史** 处可以选择历史 ID，查看构建日志。



如果要在我们之前配置的 Jenkins Slave 中运行，也就是使用动态 Slave，只需要把脚本修改成如下：

```groovy
pipeline{
    agent {
        label 'jenkins-jnlp'
    }
    stages{
        stage("GetCode"){
            steps{
                sh "echo 'Get Code' "    //step
            }
        }

        stage("build"){
            steps{
                sh "echo 'Build Code' "    //step
            }
        }

        stage("push"){
            steps{
                sh "echo 'Build Image' "    //step
            }
        }

        stage("deploy"){
            steps{
                sh "echo 'Deploy app' "    //step
            }
        }
    }
}
```

# 共享库简介

上面只是一条很简单的流水线示例，然而在企业中，却没有这么简单。

企业中，往往有非常多的项目，而且有的项目非常相似，在编排 Pipeline 的时候只需要修改少部分内容，这时候我们可以使用到 **共享库**，通过共享流水线的方式来减少冗余。

**共享库** 的目录结构如下：

```bash
(root)
+- src                     # Groovy source files
|   +- org
|       +- foo
|           +- Bar.groovy  # for org.foo.Bar class
+- vars
|   +- foo.groovy          # for global 'foo' variable
|   +- foo.txt             # help for 'foo' variable
+- resources               # resource files (external libraries only)
|   +- org
|       +- foo
|           +- bar.json    # static helper data for org.foo.Bar
```

其中：

- src 目录类似标准的 Java 源目录结构。当执行流水线时，该目录被添加到类路径下。
- vars 目录存放了可从流水线访问的全局变量的脚本。 每个 `*.groovy` 文件的基名应该是一个 `Groovy (~ Java)` 标识符, 通常是 camelCased。 匹配 `*.txt`, 如果存在, 可以包含文档, 通过系统的配置标记格式化从处理 (所以可能是 HTML, Markdown 等，虽然 txt 扩展是必需的)。 这些目录中的 Groovy 源文件 在脚本化流水线中的 “CPS transformation” 一样。
- resources 目录允许从外部库中使用 libraryResource 步骤来加载有关的非 Groovy 文件。 目前，内部库不支持该特性。
- 根目录下的其他目录被保留下来以便于将来的增强。

# 配置共享库

#### 创建项目

在 `Gitlab` 的 `devops` 组里创建名叫 `jenkins-sharelibrary` 项目，

#### 添加文件

在仓库中添加 `src/org/devops/tools.groovy` 文件，输入以下内容：

```groovy
package org.devops

def printMsg(content){
    print(content)
}
```

#### 在 Jenkins 上配置共享库

在 Jenkins 上选择 **系统配置** -> **系统配置**，找到 `Global Pipeline Libraries`，选择 **新增**，

配置 **仓库名** 和 **分支**

仓库名:sharelibrary

分支:master

配置共享仓库 Gitlab 地址，如下：

项目仓库：http://192.168.3.125:30180/devops/jenkins-sharelibrary.git

其中，凭据是我们在上一个章节里配置的。

然后点击保存，即完成配置。

#### 在 Pipeline 中使用共享库

如果要使用共享库，首先要在 Pipeline 中引入，方法如下：

```groovy
// 配置共享库，其中'sharelibrary'是在Jenkins中配置的名字。
@Library('sharelibrary')
```

如果要使用共享库中的 `printMsg` 方式，则需要先引入，再使用，如下：

```groovy
// 配置共享库，其中'sharelibrary'是在Jenkins中配置的名字。
@Library('sharelibrary')

// 引入共享库中的方法
def tools = new org.devops.tools()
```

然后就可以使用 `tools.printMsg` 调用方法了。

我们将上面的 Pipeline 改造如下：

```groovy
// 配置共享库，其中'sharelibrary'是在Jenkins中配置的名字。
@Library('sharelibrary')

// 引入共享库中的方法
def tools = new org.devops.tools()

pipeline{
    agent {
        label 'jenkins-jnlp'
    }
    stages{
        stage("GetCode"){
            steps{
                script{
                    tools.printMsg('Get Code')
                }
            }
        }

        stage("build"){
            steps{
                script{
                    tools.printMsg('Build Code')
                }
            }
        }

        stage("push"){
            steps{
                script{
                    tools.printMsg('Build Image')
                }
            }
        }

        stage("deploy"){
            steps{
                script{
                    tools.printMsg('Deploy APP')
                }
            }
        }
    }
}
```

然后替换 `dev-simple-pipeline` 项目中 Pipeline 代码.

运行测试，看能否正常使用 **共享库**，如下表示正常：

```
Started by user admin
Loading library sharelibrary@master
Attempting to resolve master from remote references...
```

但是，现在的输出平平无奇，我们给输出加点颜色。

#### 安装 `AnsiColor` 插件

选择 **系统管理** -> **插件管理**，安装 `AnsiColor` 插件，

#### 修改 `jenkins-sharelibrary` 中 `tools.groovy` 代码

如下：

```groovy
//格式化输出
def printMsg(value,color){
    colors = ['red'   : "\033[40;31m >>>>>>>>>>>${value}<<<<<<<<<<< \033[0m",
              'blue'  : "\033[47;34m ${value} \033[0m",
              'green' : "\033[40;32m >>>>>>>>>>>${value}<<<<<<<<<<< \033[0m" ]
    ansiColor('xterm') {
        println(colors[color])
    }
}
```

然后将 Pipeline 代码调整如下：

```groovy
// 配置共享库，其中'sharelibrary'是在Jenkins中配置的名字。
@Library('sharelibrary')

// 引入共享库中的方法
def tools = new org.devops.tools()

pipeline{
    agent {
        label 'jenkins-jnlp'
    }
    stages{
        stage("GetCode"){
            steps{
                script{
                    tools.printMsg('Get Code','red')
                }
            }
        }

        stage("build"){
            steps{
                script{
                    tools.printMsg('Build Code','blue')
                }
            }
        }

        stage("push"){
            steps{
                script{
                    tools.printMsg('Build Image','green')
                }
            }
        }

        stage("deploy"){
            steps{
                script{
                    tools.printMsg('Deploy APP','green')
                }
            }
        }
    }
}
```

然后替换 `dev-simple-pipeline` 项目中的 Pipeline，运行流水线查看效果，如下表示正常：



# 实验总结

这次实验我们介绍了什么是 Pipeline，以及如何开发一个简单的 Pipeline 并再 Jenkins 中部署使用。在针对复杂企业环境中，大多数都会使用到共享库，所以本次实验也带大家从 0 到 1 搭建并使用共享库，后续的实验中，我们会不断的丰富共享库，以满足日常需求。
