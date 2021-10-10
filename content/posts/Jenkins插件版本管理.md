---
title: "Jenkins插件版本管理"
date: 2021-10-10T16:04:55+08:00
draft: false
---

基本方法

* 控制变量法
* 二分法
* 查找信息源头，Read The Fucking Source Code(时间成本高，需结合谷歌大法）

## 问题现象

Jenkins Kubernetes plugin不工作，不断创建新POD，但是不执行pipeline中的step

Jenkins version: 2.263.1-lts

Kubernetes plugin version: 1.28.4

## 如何快速修复问题

尝试升级版本

控制变量法: kubernetes集群版本不变，Jenkins版本不变，只升级kubernetes插件到最新版本（1.30.3）。
升级后，一切运行正常。`It just works`，但是看起来太简单了。升级可能影响其他插件工作

你是专业的，就不要干业余的人也能轻易做到的事  --《李诞脱口秀工作手册》

尝试取法其上

## 构建可以复现问题的最小环境

创建一个干净的Jenkins服务，使用[jenkins/jenkins:2.263.1-lts](https://hub.docker.com/layers/jenkins/jenkins/2.263.1-lts/images/sha256-1433deaac433ce20c534d8b87fcd0af3f25260f375f4ee6bdb41d70e1769d9ce?context=explore)作为基础镜像，并使用该镜像中自带的插件工
具安装[kubernetes:1.28.4](https://github.com/jenkinsci/kubernetes-plugin/releases/tag/kubernetes-1.28.4)

1. 准备`Dockerfile`

    ```Dockerfile
    $ cat Dockerfile
    FROM jenkins/jenkins:2.263.1-lts
    ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
    RUN jenkins-plugin-cli -p kubernetes:1.28.4 workflow-aggregator
    ```

2. 构建测试镜像

    ```bash
    docker build -t jenkins:test .
    ```

3. 启动测试Jenkins

    ```bash
    docker run -p 8080:8080 -p 5000:5000 jenkins:test
    ```

4. 配置kubernetes

5. 测试pipline job

    ```pipeline
    pipeline {
        agent {
            kubernetes {
                yaml '''
    apiVersion: v1
    kind: Pod
    spec:
    containers:
    - name: shell
        image: busybox
        command:
        - sleep
        args:
        - infinity
    '''
                defaultContainer 'shell'
            }
        }
        stages {
            stage('Main') {
                steps {
                    sh 'hostname'
                }
            }
        }
    }
    ```

## 复现问题

运行pipeline，Jenkins job的console output如下，强制终止该job，否则会一直创建新的pod，且
错误的pod不会被删除，如没有配置quota，最终kubernetes集群资源会被耗尽

```text
Started by user unknown or anonymous
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] Start of Pipeline
[Pipeline] podTemplate
[Pipeline] {
[Pipeline] node
Created Pod: kubernetes default/test-2-3cgsd-hbsnq-szzdm
Created Pod: kubernetes default/test-2-3cgsd-hbsnq-x5k2k
Still waiting to schedule task
‘Jenkins’ doesn’t have label ‘test_2-3cgsd’
Created Pod: kubernetes default/test-2-3cgsd-hbsnq-6mvbv
Created Pod: kubernetes default/test-2-3cgsd-hbsnq-njc0l
Created Pod: kubernetes default/test-2-3cgsd-hbsnq-4lgpq
Created Pod: kubernetes default/test-2-3cgsd-hbsnq-r9rkr
Created Pod: kubernetes default/test-2-3cgsd-hbsnq-grm03
Aborted by unknown
[Pipeline] // node
[Pipeline] }
[Pipeline] // podTemplate
[Pipeline] End of Pipeline
Finished: ABORTED
```

## 确定问题修复的版本

二分法

无问题版本kubernetes-plugin:1.30.3

有问题版本kubernetes-plugin:1.28.4

[列出版本](https://github.com/jenkinsci/kubernetes-plugin/releases)

1.30.3, 1.30.2, 1.30.2, 1.27.8, 1.30.0, 1.29.7, ... , 1.28.5, 1.28.4

经过一番测试，最终确定修复问题最小版本为1.30.0。被`Read The Fucking Source Code`支配的大脑
尝试对比1.30.0和1.29.7的版本源码差异，发现依赖的`kubernetes-client-api`由`4.13.2-1`变为`5.4.1`。
此变更可能是问题的根本原因，但条件还不够充分

## 查找问题原因

查看Jenkins日志。关键日志`Refusing headers from remote: Unknown client name: `，
据google大法，是由于jenkins agent中的pod运行失败导致Jenkins不识别该agent（此段仍待考证，但不
影响进一步分析），同时发现未解决的issue，[JENKINS-54683](https://issues.jenkins.io/browse/JENKINS-54683)

```log
2021-10-10 10:25:27.071+0000 [id=30]	INFO	o.c.j.p.k.KubernetesCloud#provision: Label "test_2-3cgsd" excess workload: 1, executors: 0, plannedCapacity: 0, launching: 0
2021-10-10 10:25:28.095+0000 [id=227]	WARNING	i.f.kubernetes.client.Config#tryServiceAccount: Error reading service account token from: [/var/run/secrets/kubernetes.io/serviceaccount/token]. Ignoring.
2021-10-10 10:25:28.100+0000 [id=228]	INFO	hudson.slaves.NodeProvisioner#lambda$update$6: test-2-3cgsd-hbsnq-4lgpq provisioning successfully completed. We have now 2 computer(s)
2021-10-10 10:25:28.119+0000 [id=227]	INFO	o.c.j.p.k.KubernetesLauncher#launch: Created Pod: kubernetes default/test-2-3cgsd-hbsnq-4lgpq
2021-10-10 10:25:28.120+0000 [id=227]	WARNING	o.c.j.p.k.KubernetesLauncher#launch: Error in provisioning; agent=KubernetesSlave name: test-2-3cgsd-hbsnq-4lgpq, template=PodTemplate{id='e96857aa-b41c-4fc3-b210-5232a5a68bd6', name='test_2-3cgsd-hbsnq', label='test_2-3cgsd', annotations=[PodAnnotation{key='buildUrl', value='http://192.168.31.115:8080/job/test/2/'}, PodAnnotation{key='runUrl', value='job/test/2/'}]}
java.lang.NoSuchMethodError: io.fabric8.kubernetes.client.dsl.PodResource.watch(Ljava/lang/Object;)Ljava/lang/Object;
    at org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher.launch(KubernetesLauncher.java:159)
    at hudson.slaves.SlaveComputer.lambda$_connect$0(SlaveComputer.java:294)
    at jenkins.util.ContextResettingExecutorService$2.call(ContextResettingExecutorService.java:46)
    at jenkins.security.ImpersonatingExecutorService$2.call(ImpersonatingExecutorService.java:71)
    at java.util.concurrent.FutureTask.run(FutureTask.java:266)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)
2021-10-10 10:25:28.120+0000 [id=227]	INFO	o.c.j.p.k.KubernetesSlave#_terminate: Terminating Kubernetes instance for agent test-2-3cgsd-hbsnq-4lgpq
2021-10-10 10:25:32.300+0000 [id=257]	INFO	h.TcpSlaveAgentListener$ConnectionHandler#run: Connection #15 failed: java.io.EOFException
2021-10-10 10:25:32.354+0000 [id=258]	INFO	h.TcpSlaveAgentListener$ConnectionHandler#run: Accepted JNLP4-connect connection #16 from /172.17.0.1:62916
2021-10-10 10:25:32.441+0000 [id=227]	INFO	o.j.r.p.i.ConnectionHeadersFilterLayer#onRecv: [JNLP4-connect connection from 172.17.0.1/172.17.0.1:62916] Refusing headers from remote: Unknown client name: test-2-3cgsd-hbsnq-4lgpq
2021-10-10 10:25:37.072+0000 [id=34]	INFO	o.c.j.p.k.KubernetesCloud#provision: Label "test_2-3cgsd" excess workload: 1, executors: 0, plannedCapacity: 0, launching: 0
```

进一步查看pod日志，发现类似错误信息`Local headers refused by remote: Unknown client name:`。

```log
$ kubectl logs test-2-3cgsd-hbsnq-4lgpq jnlp
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main createEngine
INFO: Setting up agent: test-2-3cgsd-hbsnq-4lgpq
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener <init>
INFO: Jenkins agent is running in headless mode.
Oct 10, 2021 10:25:32 AM hudson.remoting.Engine startEngine
INFO: Using Remoting version: 4.3
Oct 10, 2021 10:25:32 AM org.jenkinsci.remoting.engine.WorkDirManager initializeWorkDir
INFO: Using /home/jenkins/agent/remoting as a remoting work directory
Oct 10, 2021 10:25:32 AM org.jenkinsci.remoting.engine.WorkDirManager setupLogging
INFO: Both error and output logs will be printed to /home/jenkins/agent/remoting
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Locating server among [http://192.168.31.115:8080/]
Oct 10, 2021 10:25:32 AM org.jenkinsci.remoting.engine.JnlpAgentEndpointResolver resolve
INFO: Remoting server accepts the following protocols: [JNLP4-connect, Ping]
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Agent discovery successful
Agent address: 192.168.31.115
Agent port:    50000
Identity:      02:ba:f1:1a:79:a5:cf:47:cb:72:8e:d6:5a:46:50:57
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Handshaking
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Connecting to 192.168.31.115:50000
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Trying protocol: JNLP4-connect
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Remote identity confirmed: 02:ba:f1:1a:79:a5:cf:47:cb:72:8e:d6:5a:46:50:57
Oct 10, 2021 10:25:32 AM org.jenkinsci.remoting.protocol.impl.ConnectionHeadersFilterLayer onRecv
INFO: [JNLP4-connect connection to 192.168.31.115/192.168.31.115:50000] Local headers refused by remote: Unknown client name: test-2-3cgsd-hbsnq-4lgpq
Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Protocol JNLP4-connect encountered an unexpected exception
java.util.concurrent.ExecutionException: org.jenkinsci.remoting.protocol.impl.ConnectionRefusalException: Unknown client name: test-2-3cgsd-hbsnq-4lgpq
    at org.jenkinsci.remoting.util.SettableFuture.get(SettableFuture.java:223)
    at hudson.remoting.Engine.innerRun(Engine.java:743)
    at hudson.remoting.Engine.run(Engine.java:518)
Caused by: org.jenkinsci.remoting.protocol.impl.ConnectionRefusalException: Unknown client name: test-2-3cgsd-hbsnq-4lgpq
    at org.jenkinsci.remoting.protocol.impl.ConnectionHeadersFilterLayer.newAbortCause(ConnectionHeadersFilterLayer.java:378)
    at org.jenkinsci.remoting.protocol.impl.ConnectionHeadersFilterLayer.onRecvClosed(ConnectionHeadersFilterLayer.java:433)
    at org.jenkinsci.remoting.protocol.ProtocolStack$Ptr.onRecvClosed(ProtocolStack.java:816)
    at org.jenkinsci.remoting.protocol.FilterLayer.onRecvClosed(FilterLayer.java:287)
    at org.jenkinsci.remoting.protocol.impl.SSLEngineFilterLayer.onRecvClosed(SSLEngineFilterLayer.java:172)
    at org.jenkinsci.remoting.protocol.ProtocolStack$Ptr.onRecvClosed(ProtocolStack.java:816)
    at org.jenkinsci.remoting.protocol.NetworkLayer.onRecvClosed(NetworkLayer.java:154)
    at org.jenkinsci.remoting.protocol.impl.BIONetworkLayer.access$1500(BIONetworkLayer.java:48)
    at org.jenkinsci.remoting.protocol.impl.BIONetworkLayer$Reader.run(BIONetworkLayer.java:247)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at hudson.remoting.Engine$1.lambda$newThread$0(Engine.java:117)
    at java.lang.Thread.run(Thread.java:748)
    Suppressed: java.nio.channels.ClosedChannelException
        ... 7 more

Oct 10, 2021 10:25:32 AM hudson.remoting.jnlp.Main$CuiListener error
SEVERE: The server rejected the connection: None of the protocols were accepted
java.lang.Exception: The server rejected the connection: None of the protocols were accepted
    at hudson.remoting.Engine.onConnectionRejected(Engine.java:828)
    at hudson.remoting.Engine.innerRun(Engine.java:768)
    at hudson.remoting.Engine.run(Engine.java:518)
```

`kubernetes plugin`还有更详细的日志吗？尝试从[官网](https://github.com/jenkinsci/kubernetes-plugin)找答案。

> For more detail, configure a new Jenkins log recorder for org.csanchez.jenkins.plugins.kubernetes at ALL level.

尝试配置新的[log recorder](https://support.cloudbees.com/hc/en-us/articles/204880580-How-do-I-create-a-logger-in-Jenkins-for-troubleshooting-and-diagnostic-information-)后，再次运行该job，`Log records`如下

```log
Creating Pod: kubernetes default/test-3-5zkhw-l3pmd-jmjh7
Oct 10, 2021 10:58:46 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch
Created Pod: kubernetes default/test-3-5zkhw-l3pmd-jmjh7
Oct 10, 2021 10:58:46 AM WARNING org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher launch
Error in provisioning; agent=KubernetesSlave name: test-3-5zkhw-l3pmd-jmjh7, template=PodTemplate{id='0d4eb474-ffa3-475e-b0dd-1a0bfda5b44b', name='test_3-5zkhw-l3pmd', label='test_3-5zkhw', annotations=[PodAnnotation{key='buildUrl', value='http://192.168.31.115:8080/job/test/3/'}, PodAnnotation{key='runUrl', value='job/test/3/'}]}
java.lang.NoSuchMethodError: io.fabric8.kubernetes.client.dsl.PodResource.watch(Ljava/lang/Object;)Ljava/lang/Object;
	at org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher.launch(KubernetesLauncher.java:159)
	at hudson.slaves.SlaveComputer.lambda$_connect$0(SlaveComputer.java:294)
	at jenkins.util.ContextResettingExecutorService$2.call(ContextResettingExecutorService.java:46)
	at jenkins.security.ImpersonatingExecutorService$2.call(ImpersonatingExecutorService.java:71)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

Oct 10, 2021 10:58:46 AM FINER org.csanchez.jenkins.plugins.kubernetes.KubernetesLauncher
Removing Jenkins node: test-3-5zkhw-l3pmd-jmjh7
Oct 10, 2021 10:58:46 AM INFO org.csanchez.jenkins.plugins.kubernetes.KubernetesSlave _terminate
Terminating Kubernetes instance for agent test-3-5zkhw-l3pmd-jmjh7
Oct 10, 2021 10:58:46 AM FINEST org.csanchez.jenkins.plugins.kubernetes.KubernetesCloud
Building connection to Kubernetes kubernetes URL https://192.168.31.115:8443 namespace null
Oct 10, 2021 10:58:46 AM FINE org.csanchez.jenkins.plugins.kubernetes.KubernetesCloud
Connected to Kubernetes kubernetes URL https://192.168.31.115:8443/ namespace null
```

关键信息，`java.lang.NoSuchMethodError: io.fabric8.kubernetes.client.dsl.PodResource.watch(Ljava/lang/Object;)Ljava/lang/Object;`。以上信息显示是kubernetes client代码报错

jenkins plugin manager显示kubernetes client版本是5.4.1。

查看[kubernetes plugin依赖的plugin版本](https://github.com/jenkinsci/kubernetes-plugin/blob/kubernetes-1.28.4/pom.xml#L61)

```xml
    <dependency>
      <groupId>org.jenkins-ci.plugins</groupId>
      <artifactId>kubernetes-client-api</artifactId>
      <version>4.11.1</version>
    </dependency>
```

由此怀疑是`kubernetes:1.28.4`插件和`kubernetes-client-api:5.4.1`插件版本不兼容导致。

可是为什么`jenkins-plugin-cli`不按指定的依赖版本`4.11.1`安装，而是选择`5.4.1`呢？

尝试指定`kubernetes-client-api:4.11.1`，修改Dockerfile如下

```Dockerfile
$ cat Dockerfile
FROM jenkins/jenkins:2.263.1-lts
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
RUN jenkins-plugin-cli -p kubernetes:1.28.4 kubernetes-client-api:4.11.1 workflow-aggregator
```

重新构建镜像

```
$ docker build -t jenkins:test .
[+] Building 13.1s (5/5) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                         0.0s
 => => transferring dockerfile: 232B                                                                                                                         0.0s
 => [internal] load .dockerignore                                                                                                                            0.0s
 => => transferring context: 34B                                                                                                                             0.0s
 => [internal] load metadata for docker.io/jenkins/jenkins:2.263.1-lts                                                                                       0.0s
 => CACHED [1/2] FROM docker.io/jenkins/jenkins:2.263.1-lts                                                                                                  0.0s
 => ERROR [2/2] RUN jenkins-plugin-cli -p kubernetes:1.28.4 kubernetes-client-api:4.11.1 workflow-aggregator                                                13.0s
------
 > [2/2] RUN jenkins-plugin-cli -p kubernetes:1.28.4 kubernetes-client-api:4.11.1 workflow-aggregator:
#5 12.87 Plugin kubernetes:1.28.4 depends on kubernetes-client-api:5.4.1, but there is an older version defined on the top level - kubernetes-client-api:4.11.1
------
executor failed running [/bin/sh -c jenkins-plugin-cli -p kubernetes:1.28.4 kubernetes-client-api:4.11.1 workflow-aggregator]: exit code: 1
```

为什么`kubernetes`插件指定的依赖版本是`4.11.1`，却依然会报错呢？

尝试查看`jenkins-plugin-cli`的源码，它从哪里来呢？

```bash
$ docker run --rm -it --entrypoint /bin/bash jenkins/jenkins:2.263.1-lts
jenkins@503e021bb290:/$ which jenkins-plugin-cli
/bin/jenkins-plugin-cli
jenkins@503e021bb290:/$ cat /bin/jenkins-plugin-cli
#!/usr/bin/env bash

java -jar /usr/lib/jenkins-plugin-manager.jar "$@"

```

只需查看`jenkins-plugin-manager.jar`即可，在[docker hub](hub.docker.com)搜索`jenkins/jenkins`,
再搜索tag(2.263.1-lts)，到[IMAGE LAYERS](https://hub.docker.com/layers/jenkins/jenkins/2.263.1-lts/images/sha256-1433deaac433ce20c534d8b87fcd0af3f25260f375f4ee6bdb41d70e1769d9ce?context=explore),
结合[Jenkins Dockerfile](https://github.com/jenkinsci/docker/blob/master/8/debian/bullseye/hotspot/Dockerfile)可获取如下关键信息

> ARG PLUGIN_CLI_URL=https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/2.2.0/jenkins-plugin-manager-2.2.0.jar

继续尝试二分法，获取修复版本依赖问题最小版本

2.11.0, 2.10.2, 2.9.3, ... , 2.2.0

```bash
$ docker run --rm -it --entrypoint /bin/bash jenkins/jenkins:2.263.1-lts
jenkins@503e021bb290:/$ cd tmp
jenkins@503e021bb290:/$ wget https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/2.11.0/jenkins-plugin-manager-2.11.0.jar
jenkins@503e021bb290:/$ java -jar jenkins-plugin-manager-2.10.1.jar --verbose -p kubernetes:1.28.4 kubernetes-client-api:4.11.1
```

经过一番测试，最终确定修复问题最小版本为2.10.1，查看[Release log](https://github.com/jenkinsci/plugin-installation-manager-tool/releases/tag/2.10.1)，是[此PR, Latest true respect pinned versions](https://github.com/jenkinsci/plugin-installation-manager-tool/pull/359)顺带修复了此问题。但是如果不显式指定`kubernetes-client-api:4.11.1`，该工具依然会安装最新版本依赖。也就是说如果要规避此问题，安装`kubernetes:1.28.4`时需显式指定`kubernetes-client-api:4.11.1`。

## 验证修复问题环境

1. 准备`Dockerfile`
    覆盖基础镜像中的`jenkins-plugin-manager`

    ```Dockerfile
    $ cat Dockerfile
    FROM jenkins/jenkins:2.263.1-lts
    ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
    RUN curl -fsSL https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/2.10.1/jenkins-plugin-manager-2.10.1.jar -o /usr/lib/jenkins-plugin-manager.jar
    RUN jenkins-plugin-cli -p kubernetes:1.28.4 workflow-aggregator
    ```

2. 参考`构建可以复现问题的最小环境`章节，重新测试，问题解决

## 进一步思考

为何总是默认安装最新版本的依赖而不是安装插件[指定版本的依赖](https://github.com/jenkinsci/kubernetes-plugin/blob/kubernetes-1.28.4/pom.xml#L61)呢?

假定场景：

* Update Center中插件X的最新版本是3.0.0
* 插件A(1.0.0)源码中指定依赖插件X(1.0.0)
* 插件C(1.0.0)源码中指定依赖插件X(2.0.0)

如果需要同时安装插件A(1.0.0)和插件C(1.0.0)，该安装哪个版本的插件X？

需要再啃Jenkins工作方式和java代码，但到此，已有收获，升级插件需谨慎

## 参考

[Kubernetes plugin for Jenkins](https://github.com/jenkinsci/kubernetes-plugin)

[Official Jenkins Docker image](https://github.com/jenkinsci/docker)

[Plugin Installation Manager Tool for Jenkins](https://github.com/jenkinsci/plugin-installation-manager-tool)

[How do I create a logger in Jenkins for troubleshooting and diagnostic information?](https://support.cloudbees.com/hc/en-us/articles/204880580-How-do-I-create-a-logger-in-Jenkins-for-troubleshooting-and-diagnostic-information-)
