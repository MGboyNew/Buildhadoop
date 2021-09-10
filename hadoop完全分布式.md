### 修改网络配置文件

1. 修改命令如下

   ```bash
   vi /etc/sysconfig/network-scripts/ifcfg-ens33 
   ```

2. 添加一下内容

   ```bash
   TYPE=Ethernet
   PROXY_METHOD=none
   BROWSER_ONLY=no
   BOOTPROTO=static  # <-------------- dhcp：动态获取ip地址。static静态获取Ip地址。none表示不指定，就是静态
   DEFROUTE=yes
   IPV4_FAILURE_FATAL=no
   IPV6INIT=yes
   IPV6_AUTOCONF=yes
   IPV6_DEFROUTE=yes
   IPV6_FAILURE_FATAL=no
   IPV6_ADDR_GEN_MODE=stable-privacy
   NAME=ens33 # 网卡名
   UUID=18670b85-c0d4-4515-9765-fcd035311c12
   DEVICE=ens33
   ONBOOT=yes #  <-------------------- # 在系统引导时是否激活此设备
   
   
   IPADDR=192.168.72.131  # <-------------------- # IP地址
   GATEWAY=192.168.72.2  # <-------------------- # 默认网关
   NETMASK=255.255.255.0  # <-------------------- # 子网掩码
   DNS1=114.114.114.114  # <-------------------- # 第一个DNS服务器指向
   ```

3. 重启网络

   ```bash
   systemctl restart network
   ```

### 添加用户

1.  添加

   ```bash
   usradd -d /home/sam -m sam
   ```

    此命令创建了一个用户`sam`，其中`-d`和`-m`选项用来为登录名`sam`产生一个主目录 `/home/sam`（`/home`为默认的用户主目录所在的父目录）。

2. 给予权限

   ```bash
   vim /etc/sudoers
   
   # 修改内容如下
   91 ## Allow root to run any commands anywhere 
   92 root    ALL=(ALL)       ALL
   93 ls      ALL=(ALL)       ALL  # <------------ 添加新增的用户
   ```

### 修改主机名

- `hostnamectl`

  ```bash
  sudo hostnamectl set-hostname hadoop
  ```

- `vim /etc/hostname`

  ```bash
  sudo vim /etc/hostname
  ```

### 主机映射

```bash
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.72.131 hadoop # <----------------- ip 主机名 组成。多台机器可向后追加映射
```

**作用**：实现多台机器之间的相互通信。

### `ssh`

[link](https://github.com/MGboyNew/Buildhadoop#%E5%85%8D%E5%AF%86%E7%99%BB%E5%BD%95)

**作用**：`Hadoop`运行过程中需要管理远端`Hadoop`守护进程，在`Hadoop`启动以后，`NameNode`是通过SSH`（Secure Shell）`来启动和停止各个`DataNode`上的各种守护进程的。这就必须在节点之间执行指令的时候是不需要输入密码的形式，故我们需要配置`SSH`运用无密码公钥认证的形式，这样`NameNode`使用SSH无密码登录并启动`DataName`进程，同样原理，`DataNode`上也能使用SSH无密码登录到`NameNode`。

### `JDK`

1. 解压

   ```bash
   tar -zxvf jdk-8u301-linux-i586.tar_2.gz 
   ```

2. 配置环境变量

   ```bash
   #path jdk
   export JAVA_HOME=/opt/software/jdk
   export JRE_HOME=$JAVA_HOME/jre
   export CLASS_HOME=$JAVA_HOME/lib:$JRE_HOME/lib
   export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
   ```

3. 保存后刷新

   ```bash
   source /etc/profile
   ```

4. 查看安装结果

   ```bash
   java -version 
   ```

### `HADOOP`

1. 解压

   ```bash
    sudo tar hadoop-2.6.0.tar.gz -zxvf
   ```

2. 环境变量

   ```bash
   #path hadoop
   export HADOOP_HOME=/opt/software/hadoop
   export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
   ```

3. 刷新后查看

4. **修改配置文件**

   1. `hadoop-env.sh`

      ```bash
      # The java implementation to use.
      export JAVA_HOME=${JAVA_HOME} # ----------> 改为jdk绝对路径
      ```

   2. `core-site.xml`

      ```xml
      <configuration>
              <!--配置namenode的节点位置(主节点)-->
              <property>
                      <name>fs.defaultFS</name>
                      <value>hdfs://hadoop:9000</value>
              </property>
              <!--hadoop运行过程中产生的临时文件的位置-->
              <property>
                      <name>hadoop.tmp.dir</name>
                      <value>/opt/software/hadoop/tmp</value>
              </property>
      </configuration>
      ```

   3. `hdsf-site.xml`

      ```xml
      <configuration>
              <!--hdfs节点(主节点) 的元数据的存储目录。可以是多个目录。目录之间通过逗号分割。多个元数据之间的内容是相同的，相当于起到冗余备份节点，设置多个时，可以使用不同的路径-->
              <property>
                      <name>dfs.namenode.name.dir</name>
                      <value>/opt/software/hadoop/data/hadoopdata/name</value>
              </property>
              <!--HDFS从节点的数据存储目录-->
              <property>
                      <name>dfs.datanode.data.dir</name>
                      <value>/opt/software/hadoop/data/hadoopdata/data</value>
              </property>
              <!--其中2指定HDFS文件快副本数，用于提供文件冗余的功能-->
              <property>
                      <name>dfs.replication</name>
                      <value>2</value>
              </property>
              <!--设置HDFS助理节点的信息，不可以和HDFS主节点在同一节点,防止和主节点一起挂了-->
              <property>
                      <name>dfs.secondary.http.address</name>
                      <value>hadoop</value>
              </property>
      </configuration>
      ```

   4. `yarn-site.xml`

      ```xml
      <configuration>
      
      <!-- Site specific YARN configuration properties -->
              <!-- 设置yarn的主节点-->
              <property>
                      <name>yarn.resourcemanager.hostname</name>
                      <value>hadoop</value>
              </property>
              <!-- 设置yarn的数据的使用方式-->
              <property>
                      <name>yarn.nodemanager.aux-services</name>
                      <value>mapreduce_shuffle</value>
              </property>
      
      </configuration>
      ```

   5. `sudo cp mapred-site.xml.template mapreduce-site.xml;sudo vim mapreduce-site.xml`

      ```xml
      <configuration>
              <!--指定mapreducer任务运行在资源调度器yarn上-->
              <property>
                      <name>mapreduce.framework.name</name>
                      <value>yarn</value>
              </property>
      
      </configuration>
      ```

   6. `slaves`

      ```bash
      hadooo # 节点名
      ..
      ..
      ..
      ```

   位置：`$HADOOP_HOME/etc/hadoop`

### 格式化

```
hadoop namenode -format
```

###  启动节点

**启动`HDFS`进程**

```bash
start-dfs.sh
```

**启动`yarn`进程**

```bash
start-yarn.sh
```

### 查看节点

```js
jps
```

各节点的`hadoop`进程及角色如下：

| 节点名称   | `HDFS`                                                       | Yarn                                                        |
| ---------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| `adoop01`  | `HDFS`主节点(`Namenode`)    `HDFS`从节点(`DataNode`)         | `Yarn`从节点(`NodeManager`)                                 |
| `hadoop02` | `HDFS`助理节点(`SecondaryNamenode`)    `HDFS`从节点(`Datanode`) | `Yarn`从节点(`NOdeManager`)                                 |
| `hadoop03` | `HDFS`从节点                                                 | `Yarn`主节点(`ResourceManage`)    Yarn从节点(`Nodemanager`) |

**至此，安装`Hadoop`完成**

#### 关闭防火墙

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

#### 访问

```
http://Ip(hadoop):50070
```



### 报错记录

1. `java` 修改环境变量刷新查看后。

   `error message`

   ```bash
   bash: /opt/software/jdk/bin/java: /lib/ld-linux.so.2: bad ELF interpreter: 没有那个文件或目录
   ```

   `method`

   ```bash
   sudo yum install glibc.i686
   ```

   

2. 格式化失败

   权限问题，给与权限(`hadoop`安装目录的，及需要使用的目录)即可。