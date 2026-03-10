​	

# 1. 基础环境准备

## 1.1 安装过程

省略

## 1.2 配置网络

**1. 查看当前分配的 IP**

```bash
ip addr
```

![image-20260310153545613](../学习文件/image/image-20260310153545613.png)

**2. 查看网络的网关**

```bash
ip route
```

![image-20260310153649412](../学习文件/image/image-20260310153649412.png)

**3. 配置网络**
修改 `/etc/netplan/01-network-manager-all.yaml`，目的是使得该虚拟机拥有一个静态 IP，方便后续连接访问。

```yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:                 
      dhcp4: false      # 禁止动态 IP 的分配   
      addresses:
        - 192.168.28.130/24 # 对应查到的 IP
      routes:
        - to: default
          via: 192.168.28.2 # 对应查到的网关 
      nameservers:
        addresses:          # 常用的 DNS
          - 192.168.28.2
          - 8.8.8.8
          - 1.1.1.1
```

然后使用下面的命令使得刚才配置的文件生效：

```bash
sudo netplan apply
```

## 1.3 安装与配置 SSH

```bash
sudo apt update
sudo apt install openssh-server
```

启动并设置开机自启：

```bash
sudo systemctl start ssh   # 启动服务
sudo systemctl enable ssh  # 设置开机自动启动
sudo systemctl status ssh  # 确认状态
```

关闭防火墙：

```bash
sudo ufw disable # 关闭防火墙
sudo ufw status  # 查看状态
```

## 1.4 尝试使用 Xshell 与 Xftp 进行连接

省略

---

# 2. 虚拟机克隆与集群互通

## 2.1 复制多台虚拟机

这里以三台为例（也就是再复制 2 个）。

> **注意：一定要在克隆时，选择完全克隆（与被克隆机再无瓜葛）**。

## 2.2 配置其他虚拟机网络

只需要修改查到的 IP。
比如第一台是 `192.168.28.130/24`，那么第二台就可以是 `192.168.28.131/24`，以此类推。

## 2.3 修改主机名称

我们把第一个电脑称为 hadoop01，以此类推 02，03。

查看当前主机名称：

```bash
hostname # 查看当前主机名称
```

![image-20260310155735304](../学习文件/image/image-20260310155735304.png)

修改命令：

```bash
sudo hostnamectl set-hostname hadoop01
```

## 2.4 配置主机与 IP 映射

修改 `/etc/hosts`，所有电脑都要配置。

```txt
127.0.0.1	localhost
127.0.1.1	dletc

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# 对应 IP 与主机名
192.168.28.130 hadoop01 
192.168.28.131 hadoop02
192.168.28.132 hadoop03
```

验证是否修改成功（在 hadoop01 上尝试）：

```bash
ping hadoop02
ping hadoop03
```

ping 通即成功。

## 2.5 配置 SSH 免密登录

这里以在 hadoop01 上为例，其他的以此类推。

生成密钥：

```bash
ssh-keygen -t ed25519 -b 4096 -f ~/.ssh/id_ed25519 -N ""
```

* `-t` 指定加密算法
* `-b` 指定密钥长度
* `-f` 指定储存路径以及文件名
* `-N` 指定密码，由于要免密登录，所以这里不设置

将公钥拷贝到远程服务器：

```bash
# ssh-copy-id 用户名@服务器IP
ssh-copy-id hadoop01 # 自己也要进行拷贝
ssh-copy-id hadoop02
ssh-copy-id hadoop03
```

检查方法：

```bash
ssh hadoop02
ssh hadoop03
```

看是否是不需要密码直接可以登录。

---

# 3. 集群软件安装

## 3.1 安装 JDK

> **注意：这里请注意 JDK 版本与 Hadoop 版本一定要兼容（这里选择的是 JDK 8, 以及 Hadoop 3.3.6）**。

有梯子的情况下，可以直接使用命令安装：

```bash
sudo apt update
sudo apt install openjdk-8-jdk -y
```

安装好之后，需要配置 JDK 的环境变量。
配置 `/etc/profile`，在最后面添加：

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 # 这里需要改成你安装的 java 的目录
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```

保存退出后 source 一下环境变量：

```bash
source /etc/profile
```

验证是否成功：

```bash
java -version
```

![image-20260310161437724](../学习文件/image/image-20260310161437724.png)

## 3.2 安装 Hadoop

创建目录 `/usr/hadoop`。
在网上下载好的安装包通过 xftp 传输到 `/usr/hadoop`，并在该目录下进行解压。

配置环境变量（在 `/etc/profile` 文件中）：

```bash
export HADOOP_HOME=/usr/hadoop/hadoop-3.3.6
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

配置好后，source 一下环境：

```bash
source /etc/profile
```

验证是否成功：

```bash
hadoop version
```

![image-20260310161855312](../学习文件/image/image-20260310161855312.png)

---

# 4. Hadoop 集群配置

> **提示：以下步骤三台电脑的操作完全一样，不需要更改配置文件中的内容**。

## 4.1 配置 hadoop-env.sh

打开 `/usr/hadoop/hadoop-3.3.6/etc/hadoop/hadoop-env.sh` 文件。
![image-20260310162158856](../学习文件/image/image-20260310162158856.png)
把对应行注释取消，并在后面加上你电脑中 java 的位置。

## 4.2 配置 core-site.xml

**定义了 HDFS 的入口和本地数据存放路径**。
打开 `/usr/hadoop/hadoop-3.3.6/etc/hadoop/core-site.xml` 文件：

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>  
        <value>hdfs://hadoop01:8020</value>
    </property>

    <property>
        <name>hadoop.tmp.dir</name> 
        <value>/usr/hadoop/hadoop-3.3.6/data</value>
    </property>
    
    <property>
        <name>hadoop.http.staticuser.user.name</name> 
        <value>dletc</value> </property>
</configuration>
```

**具体参数解释：**

* **fs.defaultFS**：协议层 (hdfs://) 告诉客户端访问的是 Hadoop 专用的分布式协议；逻辑层 (hadoop01) 是 NameNode 所在地；端口层 (8020) 是 NameNode 监听 RPC 请求的内部端口。
* **hadoop.tmp.dir**：它是所有存储的“父目录”。Hadoop 默认指向 Linux 的 `/tmp` 目录，重启会自动清理导致集群崩溃，因此需要显式配置。
* **hadoop.http.staticuser.user.name**：通过此配置告诉 Hadoop 从 Web 浏览器过来的请求默认视为你的登录用户，避免权限问题。

## 4.3 配置 hdfs-site.xml

专门负责定义**文件系统的存储规则**，决定了数据如何被切分、保护以及监控。
打开 `/usr/hadoop/hadoop-3.3.6/etc/hadoop/hdfs-site.xml` 文件：

```xml
<configuration>
    <property>
        <name>dfs.replication</name> 
        <value>3</value>
    </property>

    <property>
        <name>dfs.webhdfs.enabled</name> 
        <value>true</value>
    </property>

    <property>
        <name>dfs.namenode.http-address</name> 
        <value>hadoop01:9870</value>
    </property>

    <property>
        <name>dfs.datanode.http.address</name> 
        <value>0.0.0.0:9864</value> </property>

    <property>
        <name>dfs.namenode.secondary.http-address</name> 
        <value>hadoop02:9868</value>
    </property>
</configuration>
```

**具体参数解释：**

* **dfs.replication**：容错处理，设置为 3 意味着 HDFS 会把每一个数据块在集群的不同机器上存 3 份。
* **dfs.namenode.http-address**：Web 控制台，支撑浏览器访问 9870 端口查看文件系统。
* **dfs.namenode.secondary.http-address**：配在 hadoop02，用于定期把主节点的 edits 日志和旧镜像文件合并，防止主节点性能瓶颈。
* **dfs.webhdfs.enabled**：开启后，任何支持 HTTP 的设备都可以通过 RESTful API 读取或上传文件，也是高级组件通信关键。
* **dfs.datanode.http.address**：配置为 `0.0.0.0:9864`，意思就是监听所有节点的 9864 端口。这意味着 DataNode 的 Web 服务会绑定该节点上的所有网卡（IP），允许外部通过浏览器（如 `http://<DataNode_IP>:9864`）直接访问并查看数据节点的状态。

## 4.4 配置 yarn-site.xml

打开 `/usr/hadoop/hadoop-3.3.6/etc/hadoop/yarn-site.xml` 文件：

```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop01</value>
    </property>

    <property>
        <name>yarn.resourcemanager.address</name>
        <value>hadoop01:8032</value>
    </property>

    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>

    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

**具体参数解释：**

* **yarn.resourcemanager.xxx**：告诉所有从节点分配资源的权力在 hadoop01 手里，8032 是 RM 的 RPC 通信端口。
* **yarn.nodemanager.aux-services**：相当于在每个 NodeManager 安装了插件，允许 YARN 在底层协助处理 MapReduce 特有的数据洗牌请求。
* **yarn.nodemanager.vmem-check-enabled**：设为 false 关闭虚拟内存检查，防止 MapReduce 任务刚跑就被 YARN 强制杀掉。
* **yarn.nodemanager.env-whitelist**：特许通行证，告诉 NodeManager 把关键环境变量传递进隔离的容器中，避免任务找不到 Java。

## 4.5 配置 mapred-site.xml

明确了 MapReduce 任务必须在 YARN 平台上运行。
打开 `/usr/hadoop/hadoop-3.3.6/etc/hadoop/mapred-site.xml` 文件：

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name> <value>yarn</value>
    </property>

    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>hadoop01:10020</value>
    </property>

    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>hadoop01:19888</value>
    </property>

    <property>
        <name>mapreduce.application.classpath</name>
        <value>/usr/hadoop/hadoop-3.3.6/share/hadoop/mapreduce/*:/usr/hadoop/hadoop-3.3.6/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

**具体参数解释：**

* **mapreduce.framework.name**：委派管理，所有的资源申请必须通过 YARN 来完成。
* **mapreduce.jobhistory.xxx**：指定后台进程把跑完的任务详情记录下来，10020 为 RPC 端口，19888 为浏览器查看的可视化复盘界面地址。
* **mapreduce.application.classpath**：显式指定路径，保证任务运行时能精准找到需要的 .jar 包，解决找不到类的报错。

## 4.6 配置 workers

`workers` 文件是集群的**“员工花名册”**，也是实现**“一键群控”**的底层依据。
打开 `/usr/hadoop/hadoop-3.3.6/etc/hadoop/workers` 文件，把原内容删除并放进去对应节点：

```txt
hadoop01
hadoop02
hadoop03
```

---

# 5. 启动集群

> **这些操作仅在 hadoop01 (主节点) 中进行**。

## 5.1 格式化 HDFS

```bash
hdfs namenode -format
```

## 5.2 启动 Hadoop 集群

不建议使用 start-all 命令暴力启动。

```bash
start-dfs.sh
start-yarn.sh
```

## 5.3 关闭 Hadoop 集群

不建议使用 stop-all 命令暴力停止。

```bash
stop-yarn.sh
stop-dfs.sh
```

---

# 6. 常见问题排查

## 6.1 网络相关问题

**解决右上角网络图标消失：**

```bash
sudo service NetworkManager stop
sudo rm /var/lib/NetworkManager/NetworkManager.state
sudo service NetworkManager start
```

## 6.2 SSH 免密失败

### 1. 权限问题

如果执行了 `ssh-copy-id` 仍然需要密码，90% 的情况是由于服务器端的**目录权限过大**（SSH 安全策略规定：如果权限太开，密钥就可能被他人篡改，因此会失效）。

```bash
# 1. 确保 .ssh 目录权限为 700 (drwx------)
chmod 700 ~/.ssh

# 2. 确保 authorized_keys 文件权限为 600 (-rw-------)
chmod 600 ~/.ssh/authorized_keys
```

### 2. 忘记把公钥赋值给自己

```bash
ssh-copy-id hadoop01
```
