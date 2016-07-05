---
title: Setting up a Storm Cluster
layout: documentation
documentation: true
---
文中部分内容来自[weyo/Storm-Documents](https://github.com/weyo/Storm-Documents/blob/master/Manual/zh/Setting-Up-A-Storm-Cluster.md)

本文详细介绍了 Storm 集群的安装配置方法。如果需要在 AWS 上安装 Storm，你应该看一下 [storm-deploy](https://github.com/nathanmarz/storm-deploy/wiki) 项目。[storm-deploy](https://github.com/nathanmarz/storm-deploy/wiki) 可以自动完成 E2 上 Storm 集群的准备、配置、安装的全部过程，同时还设置好了 Ganglia，方便监控 CPU、磁盘以及网络的使用信息。

如果你在使用 Storm 集群时遇到问题，请先查看 [Troubleshooting](Troubleshooting.html) 一文中是否已有相应的解决方案。如果检索不到有效的解决方法，请向社区的邮件列表发送关于问题的邮件。

以下是安装 Storm 的步骤：

1. 安装 ZooKeeper 集群；
2. 在各个机器上安装运行集群所需要的依赖组件；
3. 下载 Storm 安装程序并解压缩到集群的各个机器上；
4. 在 storm.yaml 中添加集群配置信息；
5. 使用 `storm` 脚本启动各机器后台进程。

### 安装 ZooKeeper 集群

Storm 使用 ZooKeeper 来保证集群的一致性。集群中 ZooKeeper 并不是用来进行消息传递的，所以 Storm 对 ZooKeeper 的负载相当低。虽然在大部分场景下单点 ZooKeeper 也勉强够用，但是如果你需要更可靠的 HA 机制或者需要部署大规模 Storm 集群，你最好配置一个 ZooKeeper 集群。ZooKeeper 集群的部署说明请参考[此文](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html)。

关于 ZooKeeper 部署的几点说明：

A few notes about Zookeeper deployment:

1. ZooKeeper 必须在监控模式下运行。因为 ZooKeeper 是个快速失败系统，如果遇到了故障，ZooKeeper 服务会主动关闭。更多详细信息请参考[此文](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_supervision) 。
2. 需要设置一个 cron 服务来定时压缩 ZooKeeper 的数据与事务日志。因为 ZooKeeper 的后台进程不会处理这个问题，如果不配置 cron，ZooKeeper 的日志会很快填满磁盘空间。更多详细信息请参考[此文](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_maintenance)。

### 安装必要的依赖组件

接下来你需要在集群中的所有机器上安装必要的依赖组件，包括：

1. Java 6
2. Python 2.6.6

以上均为在 Storm 上测试通过的版本。Storm 并不保证对其他版本的 Java 或 Python 的支持。

### 下载 Storm 安装程序并解压

接下来就要下载需要的 Storm 发行版，并将 zip 安装文件解压缩到集群中的各个机器上。Storm 的发行版可以在[这里下载](http://github.com/apache/storm/releases)。

### 配置 storm.yaml

The Storm release contains a file at `conf/storm.yaml` that configures the Storm daemons. You can see the default configuration values [here]({{page.git-blob-base}}/conf/defaults.yaml). storm.yaml overrides anything in defaults.yaml. There's a few configurations that are mandatory to get a working cluster:

1) **storm.zookeeper.servers**: This is a list of the hosts in the Zookeeper cluster for your Storm cluster. It should look something like:

```yaml
storm.zookeeper.servers:
  - "111.222.333.444"
  - "555.666.777.888"
```

If the port that your Zookeeper cluster uses is different than the default, you should set **storm.zookeeper.port** as well.

2) **storm.local.dir**: The Nimbus and Supervisor daemons require a directory on the local disk to store small amounts of state (like jars, confs, and things like that).
 You should create that directory on each machine, give it proper permissions, and then fill in the directory location using this config. For example:

```yaml
storm.local.dir: "/mnt/storm"
```
If you run storm on windows,it could be:
```yaml
storm.local.dir: "C:\\storm-local"
```
If you use a relative path,it will be relative to where you installed storm(STORM_HOME).
You can leave it empty with default value `$STORM_HOME/storm-local`

3) **nimbus.seeds**: The worker nodes need to know which machines are the candidate of master in order to download topology jars and confs. For example:

```yaml
nimbus.seeds: ["111.222.333.44"]
```
You're encouraged to fill out the value to list of **machine's FQDN**. If you want to set up Nimbus H/A, you have to address all machines' FQDN which run nimbus. You may want to leave it to default value when you just want to set up 'pseudo-distributed' cluster, but you're still encouraged to fill out FQDN.

4) **supervisor.slots.ports**: For each worker machine, you configure how many workers run on that machine with this config. Each worker uses a single port for receiving messages, and this setting defines which ports are open for use. If you define five ports here, then Storm will allocate up to five workers to run on this machine. If you define three ports, Storm will only run up to three. By default, this setting is configured to run 4 workers on the ports 6700, 6701, 6702, and 6703. For example:

```yaml
supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

### Monitoring Health of Supervisors

Storm provides a mechanism by which administrators can configure the supervisor to run administrator supplied scripts periodically to determine if a node is healthy or not. Administrators can have the supervisor determine if the node is in a healthy state by performing any checks of their choice in scripts located in storm.health.check.dir. If a script detects the node to be in an unhealthy state, it must print a line to standard output beginning with the string ERROR. The supervisor will periodically run the scripts in the health check dir and check the output. If the script’s output contains the string ERROR, as described above, the supervisor will shut down any workers and exit.

If the supervisor is running with supervision "/bin/storm node-health-check" can be called to determine if the supervisor should be launched or if the node is unhealthy.

The health check directory location can be configured with:

```yaml
storm.health.check.dir: "healthchecks"

```
The scripts must have execute permissions.
The time to allow any given healthcheck script to run before it is marked failed due to timeout can be configured with:

```yaml
storm.health.check.timeout.ms: 5000
```

### Configure external libraries and environmental variables (optional)

If you need support from external libraries or custom plugins, you can place such jars into the extlib/ and extlib-daemon/ directories. Note that the extlib-daemon/ directory stores jars used only by daemons (Nimbus, Supervisor, DRPC, UI, Logviewer), e.g., HDFS and customized scheduling libraries. Accordingly, two environmental variables STORM_EXT_CLASSPATH and STORM_EXT_CLASSPATH_DAEMON can be configured by users for including the external classpath and daemon-only external classpath.


### Launch daemons under supervision using "storm" script and a supervisor of your choice

The last step is to launch all the Storm daemons. It is critical that you run each of these daemons under supervision. Storm is a __fail-fast__ system which means the processes will halt whenever an unexpected error is encountered. Storm is designed so that it can safely halt at any point and recover correctly when the process is restarted. This is why Storm keeps no state in-process -- if Nimbus or the Supervisors restart, the running topologies are unaffected. Here's how to run the Storm daemons:

1. **Nimbus**: Run the command "bin/storm nimbus" under supervision on the master machine.
2. **Supervisor**: Run the command "bin/storm supervisor" under supervision on each worker machine. The supervisor daemon is responsible for starting and stopping worker processes on that machine.
3. **UI**: Run the Storm UI (a site you can access from the browser that gives diagnostics on the cluster and topologies) by running the command "bin/storm ui" under supervision. The UI can be accessed by navigating your web browser to http://{ui host}:8080.

As you can see, running the daemons is very straightforward. The daemons will log to the logs/ directory in wherever you extracted the Storm release.
