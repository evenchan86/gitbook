# Pipeline As Code With Jenkins2.0

### Pipeline原理与流程

Pipeline为用户设计了三个最最基本的概念：

*  **Stage**：一个Pipeline可以划分为若干个Stage，每个Stage代表一组操作。注意，Stage是一个逻辑分组的概念，可以跨多个Node。
*  **Node**：一个Node就是一个Jenkins节点，或者是Master，或者是Agent，是执行Step的具体运行期环境。
*  **Step**：Step是最基本的操作单元，小到创建一个目录，大到构建一个Docker镜像，由各类Jenkins Plugin提供。

一个典型的Stage View如下图所示：![](//upload-images.jianshu.io/upload_images/9824247-46f22a19f0780f1b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)典型的Stage View

从图中可以十分方便地看到哪些Stage通过，哪些Stage失败，以及构建的时间。

Jenkins2.0的Pipeline搭建使用的是Groovy脚本，通过Groovy脚本实现工作流管理的步骤如下：

* 去Jenkins主界面建立Pipeline任务

![](//upload-images.jianshu.io/upload_images/9824247-ec813fc77ee5e081.png?imageMogr2/auto-orient/strip|imageView2/2/w/1124/format/webp)建立PipeLine任务

实际上更常用的是MultiBranch Pipeline，上面的图中截图没有包含，但与普通Pipeline基本类似。

* 使用Groovy脚本自定义工作流

![](//upload-images.jianshu.io/upload_images/9824247-b74e22e32c762fa5.png?imageMogr2/auto-orient/strip|imageView2/2/w/1133/format/webp)使用Groovy自定义工作流

上图的实例脚本如下：

```text
node { 
    stage('Checkout Code') { // for display purposes 
        // Get some code from a GitHub repository 
        git 'https://github.com/jglick/simple-maven-project-with-tests.git' 
    }
 
    stage('Build') { 
        // Run the maven build 
        if (isUnix()) { 
            sh "'${MAVEN_HOME}/bin/mvn' -Dmaven.test.failure.ignore clean package" 
        } else { 
            bat(/"${MAVEN_HOME}\bin\mvn" -Dmaven.test.failure.ignore clean package/) 
        } 
    } 
 
    stage('Unit test') { 
        junit '**/target/surefire-reports/TEST-UT.xml' 
        archive 'target/*.jar' 
    } 
} 
```

* 开始执行Pipeline

构建过程的stage View如下：![](//upload-images.jianshu.io/upload_images/9824247-24339d9c5f5ff1a7.png?imageMogr2/auto-orient/strip|imageView2/2/w/919/format/webp)stage View

很明显可以看出，这里显示的和Groovy脚本中格式化的代码是一致的，会实时显示各个工作流的执行进度和结果，直观易懂。鼠标移上去，能看到日志信息的缩略图，单击可以调到对应stage的console中。

总而言之，一切都是那么地优雅！

### Jenkins2.0 Pipeline关键DSL语法及示例

在这里总结一下Pipeline中的关键DSL语法，利用Groovy对其进行组合可以完成任何一项复杂的CI/CD流程，熟悉它们大有裨益。

* **archiveArtifacts**

归档文件，举例：

```text
archiveArtifacts 'target/*.jar'
```

* **bat**

执行windows平台下的批处理文件，如

```text
bat "call example.bat"
```

* **build**

触发构建一个jenkins job，如

```text
build 'TEST_JOB'
```

* **checkout**

从SCM系统中checkout repo，如：

```text
checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[credentialsId: '30e6c1e5-1035-4bdd-8a44-05ba8f885158', depthOption: 'infinity', ignoreExternalsOption: true, local: '.', remote: 'svn://xxxxxx']], workspaceUpdater: [$class: 'UpdateUpdater']]) 
```

* **deleteDir\(\)**

从workspace中删除当前目录

* **dir**

切换目录，如

```text
dir('/home/jenkins') { // 切换到/home/jenkins目录中做一些事情
    // some block
}
```

* **echo**

打印信息，如 echo 'hello world'

* **emailtext**

利用Jenkins发送邮件，内容、主题全都可以自定义，如

```text
emailext body: 'Subject_test', subject: 'Subject_test', to: 'hansonwang99@163.com.cn'
// 邮件的正文body，主题subject，收件人to等可以进行自定义
```

* **error**

抛出一个错误信号，可以自行在代码里抛出，如 error 'read\_error'

* **fileExists**

检查工作空间某个路径里是否存在某个file，举例：

```text
fileExists '/home/test.txt'  // 检查是否存在test.txt
```

* **input**

等待外界用户的交互输入，举例：

```text
input message: '', parameters: [string(defaultValue: '默认值', description: '版本号', name: 'version')] // 在某一步骤，等待用户输入version参数才能往下执行
```

* **isUnix**

用于判断当前任务是否运行于Unix-like节点上，举例：

```text
def flag = isUnix()
if( flag == false ) { // 可以据此进行判断
  echo "not run on a unix node !"
}
```

* **load**

调用一个外部groovy脚本，举例：

```text
load 'D:\\jenkins\\workspace\\test.groovy'
```

* **node**

分配节点给某个任务运行，举例：

```text
node('节点标签') { // 在对应标签的节点上运行某项任务
    Task()
}
```

* **parallel**

并行地执行任务，可以说是最实用高效的工具了，举例：

```text
parallel(   //并行地执行android unit tests和android e2e tests两个任务
    'android unit tests': {
        runCmdOnDockerImage(androidImageName, 'bash /app/ContainerShip/scripts/run-android-docker-unit-tests.sh', '--privileged --rm')
    },
    'android e2e tests': {
    runCmdOnDockerImage(androidImageName, 'bash /app/ContainerShip/scripts/run-ci-e2e-tests.sh --android --js', '--rm')
    }
)
```

*  **properties**：

设置Job的属性，举例：

```text
properties([parameters([string(defaultValue: '1.0.0', description: '版本号', name: 'VERSION')]), pipelineTriggers([])]) // 为job设置了一个VERSION参数
```

* **pwd** 显示当前目录
* **readFile**

从工作空间中读取文件，举例：

```text
def editionName = readFile '/home/Test/exam.txt'
```

* **retry**

重复body内代码N次，举例：

```text
retry(10) {
    // some block
}
```

* **sh**

执行shell脚本，如：sh "sh test.sh"

* **sleep**

延时，如延时2小时：sleep time: 2, unit: 'HOURS'

* **stage**

创建任务的stage，举例：

```text
stage('stage name') {
    // some block
}
```

* **stash**

存放文件为后续构建使用，举例：

```text
dir('target') {
    stash name: 'war', includes: 'x.war'
}
```

* **unstash**

将stash步骤中存放的文件在当前工作空间中重建，举例：

```text
def deploy(id) {
    unstash 'war'
    sh "cp x.war /tmp/${id}.war"
}
```

* **timeout**

时间限制，举例

```text
timeout(time: 4, unit: 'SECONDS') {
    // some block
}
```

* **timestamps**

用于在控制台加时间戳，举例：

```text
timestamps {
    // some block
}
```

* **touch**

创建文件，举例：

```text
touch file: 'TEST.txt', timestamp: 0
```

* **unzip**

解压文件，举例：

```text
unzip dir: '/home/workspace', glob: '', zipFile: 'TEST.zip'
```

* **validateDeclarativePipeline**

检查给定的文件是否包含一个有效的Declarative Pipeline，返回T或者F

```text
validateDeclarativePipeline '/home/wospace'
```

* **waitUntil**

等待，直到条件满足

```text
waitUntil {
    // some block
}
```

* **withCredentials**

使用凭据

```text
withCredentials([usernameColonPassword(credentialsId: 'mylogin', variable: 'USERPASS')]) {
    sh '''
      set +x
      curl -u $USERPASS https://private.server/ > output
    '''
}
```

* **withEnv**

设置环境变量，注意近本次运行有效!

```text
withEnv(['MYTOOL_HOME=/usr/local/mytool']) {
    sh '$MYTOOL_HOME/bin/start'
}
```

* **writeFile**

写文件到某个路径

```text
writeFile file: '/home/workspace', text: 'hello world'
```

* **writeJSON**

写JSON文件，用法基本同上

* **zip**

创建zip文件

```text
zip dir: '/home/workspace', glob: '', zipFile: 'TEST.zip'
```

* **ws**

自定义工作空间，在其中做一些工作，效果类似于Dir命令，举例：

```text
ws('/home/jenkins_workspace') {
    // some block
}
```

