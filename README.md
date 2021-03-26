## Hadoop完全分布式搭建

### 修改网络设置

修改文件`/etc/sysconfig/network-scripts/ifcfg-ens33`

- 修改uuid

  ```bash
  sed -i "/UUID/c UUID=$(uuidgen)" /etc/sysconfig/network-scripts/ifcfg-ens33
  ```

- 修改以下字段

  ```bash
  BOOTPROTO=static
  ONBOOT=yes
  ```

- 添加字段

  ```bash
  #ip
  IPADDR=192.168.200.131
  #网关
  GATEWAY=192.168.200.2
  #掩码
  NETMASK=255.255.255.0
  ```

- 重启网络配置

  ```
  systemctl restart network
  ```

  **修改作用：用于ssh连接软件连接作准备**

### 修改主机名

​	以下两种方法均可

- `hostnamectl set-hostname 主机名`

- 修改/etc/hostname文件直接修改手动输入主机名即可

  **修改作用：虚拟机的主机名不修改那么默认应该是一样的，这样在集群中很不妥当，难于执行后续操作**

### 修改主机间的映射(简称修改映射)

​	修改/etc/hosts文件并写入如下代码

```bash
192.168.10.131 hadoop01
192.168.10.132 hadoop02
192.168.10.133 hadoop03
//第一个字段是ip 第二个字段是主机名
```

​	**修改作用：用于三台机器的映射，从而三台机器达到简单通讯的功能**

### 免密登录

1. 生成`.ssh`文件(生成密钥)


```bash
ssh-keygen -t rsa
```

2. 拷贝密钥：

- 方法一

  ```bash
  #在hadoop2执行(将id_rsa.pub文件发送到hadoop01并命名为i2)
  scp id_rsa.pub root@hadoop01:/root/.ssh/i2
  #在hadoop3执行
  scp id_rsa.pub root@hadoop01:/root/.ssh/i3
  #在hadoop1执行(将文件内容分别写入authorized_keys)
  cat id_rsa.pub >> authorized_keys
  cat i2 >> authorized_keys
  cat i3 >> authorized_keys
  #最后在hadoop1执行
  scp authorized_keys root@hadoop02:/root/.ssh/
  scp authorized_keys root@hadoop03:/root/.ssh/
  ```

- 方法二

  ```bash
  #在Hadoop01执行(直接通过命令将密钥生成到指定的机器)
  ssh-copy-id hadoop01
  #在hadoop02执行
  ssh-copy-id hadoop01
  #在Hadoop03执行
  ssh-copy-id hadoop01
  #最后在hadoop1执行
  scp authorized_keys root@hadoop02:/root/.ssh/
  scp authorized_keys root@hadoop03:/root/.ssh/
  ```

  **其实ssh免密登录是为了集群间机器的免密登录，也就是不用密码也可以互相访问。可以认为以上操作前几步是将密码都存放在authorized_keys文本文件中，然后发送到每台机器，从而实现达到免密登录。**

### 安装jdk

1. 在linux中安装压缩文件中的应用，只需要解压就可以安装了。但是为了对应用进行一些必要的修改，请将应用安装在指定的文件夹。

请在你存放压缩文件的目录下执行以下代码。

```bash
#创建/export/servers目录;将jdk-8u161-linux-x64.tar.gz文件解压到目录;进入到/export/servers/目录;修改jdk1.8.0_161名字为jdk
mkdir /export/servers;tar -zxvf jdk-8u161-linux-x64.tar.gz -C /export/servers/;cd /export/servers/;mv jdk1.8.0_161 jdk
```

2. 以上代码并未结束，因为你还需配置环境变量。你可以编辑`/etc/profile`或者`~/.bashrc`文件，前者是为你电脑的所有用户配置，后者是为单独用户(当前用户配置)。添加如下代码至前者。

```bash
#JAVA
export JAVA_HOME=/export/servers/jdk/
export JRE_HOME=/export/servers/jdk/jre
export CLASS_HOME=$JAVA_HOME/lib:$JRE_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
```

3. 最后执行。

```bash
#刷新环境变量(生效)。
source /etc/profile
```

**至此,jdk安装完成**

### 安装Hadoop

1. 解压并修改名字，请执行下述命令。

```bash
#解压;进入目录;修改名字
tar -zxvf hadoop-2.7.4.tar.gz -C /export/servers/;cd /export/servers/;mv hadoop-2.7.4/ hadoop
```

2. 修改环境变量，请将以下代码添加至/etc/profile

```bash
export HADOOP_HOME=/export/servers/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
```

3. 刷新。在修改/etc/profile文件后都要进行刷新，以后该行代码省略。

```
source /etc/profile
```

#### 4. 配置文件修改

   - hadoop-env.sh

     ```bash
     export JAVA_HOME=/export/servers/jdk/
     ```

   - core-site.xml

     ```xml
     <property>
     	<!--配置namenode的节点位置(主机名和节点)(主节点)-->    
         <name>fs.defaultFS</name>
     	<!--主节点的URL链接-->    
         <value>hdfs://hadoop01:9000</value>
     </property>
     <property>
     	<!--配置hadoop运行过程中产生的临时文件的位置，需要具有读写权限-->    
         <name>hadoop.tmp.dir</name>
         <value>/export/servers/hadoop/tmp</value>
     </property>
     ```

   - hdfs-site.xml

     ```xml
     <property>
     	<!--"/export/servers/..."是HDFS主节点的元数据的存储目录，可以是多个目录，目录之间通过逗号分隔，多个元数据之间的内容是相同的，相当于起到冗余备份的节点，建议设置多个时，将数据指定在不停的地方-->
     	<name>dfs.namenode.name.dir</name>
     	<value>/export/servers/hadoop/data/hadoopdata/name</value>
     </property>
     <property>
     	<!--"/export/servers/..."是HDFS从节点的数据存储目录，注意这个目录的权限为可读-->    
     	<name>dfs.datanode.data.dir</name>
         <value>/export/servers/hadoop/data/hadoopdata/data</value>
     </property>
     <property>
         <!--其中2表示HDFS文件块副本数，用于提供文件的冗余功能-->
     	<name>dfs.replication</name>
         <value>2</value>
     </property>
     <property>
         <!--设置HDFS助理节点的信息，不能和HDFS主节点在同一节点-->
     	<name>dfs.secondary.http.address</name>
         <value>hadoop02:50090</value>
     </property>
     ```

   - yarn-site.xml

     ```xml
     <property>
         <!--设置yarn的主节点，此节点为hadoop03-->
     	<name>yarn.resourcemanager.hostname</name>
         <value>hadoop03</value>
     </property>
     <property>
         <!--设置yarn的数据使用方式，这里默认为Mapreduce_shuffle-->
     	<name>yarn.nodemanager.aux-services</name>
         <value>mapreduce_shuffle</value>
     </property>
     ```

   - mapreduce-site.xml

     ```xml
     <!--复制模板文件-->
     <property>
         <!--指定mapreducer任务运行在资源调度器yarn上-->
     	<name>mapreduce.framework.name</name>
         <value>yarn</value>
     </property>
     ```

   - slaves

     ```xml
     hadoop01
     hadoop02
     hadoop03
     ```

#### 5.远程拷贝到其他虚拟机

```bash
//进入软件包所在目录
scp jdk root@hadoop02:/export/servers
scp jdk root@hadoop03:/export/servers


scp hadoop root@hadoop02:/export/servers
scp hadoop root@hadoop03:/export/servers
```

#### 6. 格式化HDFS分布式文件系统

```js
hadoop namenode -format
```

#### 7.  启动节点

**启动HDFS进程**

```bash
start-dfs.sh
```

**启动yarn进程**

```bash
start-yarn.sh
```

#### 8. 查看节点

```js
jps
```

各节点的hadoop进程及角色如下：

| 节点名称 | HDFS                                                    | Yarn                                                  |
| -------- | ------------------------------------------------------- | ----------------------------------------------------- |
| hadoop01 | HDFS主节点(Namenode)    HDFS从节点(DataNode)            | Yarn从节点(NodeManager)                               |
| hadoop02 | HDFS助理节点(SecondaryNamenode)    HDFS从节点(Datanode) | Yarn从节点(NOdeManager)                               |
| hadoop03 | HDFS从节点                                              | Yarn主节点(ResourceManage)    Yarn从节点(Nodemanager) |

**至此，安装Hadoop完成**

#### 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

#### 9. 访问

```
http://hadoop01:50070
```

