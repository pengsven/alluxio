---
layout: global
title: 在HDFS上配置Alluxio
nickname: Alluxio使用HDFS
group: Storage Integrations
priority: 3
---

* 内容列表
{:toc}

该指南给出了使用说明以配置[HDFS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html)作为Alluxio的底层文件系统。

## 初始步骤

要在一组机器上运行一个Alluxio集群，需要在每台机器上部署Alluxio二进制服务端包。你可以[下载带有正确Hadoop版本的预编译二进制包](Running-Alluxio-Locally.html)，对于高级用户，也可[源码编译Alluxio](Building-Alluxio-From-Source.html)，

注意，在编译源码包的时候，默认的Alluxio二进制包适用于HDFS `2.7.3`，若使用其他版本的Hadoop，需要指定正确的Hadoop版本，并且在Alluxio源码目录下运行如下命令：

```console
$ mvn install -P<YOUR_HADOOP_PROFILE> -DskipTests
```

Alluxio提供了预定义配置文件，其包含`hadoop-1`，`hadoop-2.2`，`hadoop-2.3` ··· `hadoop-2.9`的Hadoop版本。如果你想编译特定Hadoop版本的Alluxio，你应该在命令中指定版本。
例如，

```console
$ mvn install -Phadoop-2.7 -Dhadoop.version=2.7.1 -DskipTests
```

将会编译Hadoop 2.7.1版本的Alluxio。如果想获取更多的版本支持，请访问[编译Alluxio主分支](Building-Alluxio-From-Source.html#distro-support)。

## 配置Alluxio

你需要通过修改`conf/alluxio-site.properties`来配置Alluxio使用底层存储系统，如果该配置文件不存在，则根据模版创建一个配置文件

```console
$ cp conf/alluxio-site.properties.template conf/alluxio-site.properties
```

### 基本配置

修改`conf/alluxio-site.properties`文件，将底层存储系统的地址设置为HDFS namenode的地址以及你想挂载到Alluxio根目录下的HDFS目录。例如，若你的HDFS namenode是在本地默认端口运行，并且HDFS的根目录已经被映射到Alluxio根目录，则该地址为`hdfs://localhost:9000`；若只有`/alluxio/data`这一个HDFS目录被映射到Alluxio根目录，则该地址为`hdfs://localhost:9000/alluxio/data`。

```
alluxio.master.mount.table.root.ufs=hdfs://<NAMENODE>:<PORT>
```

### HDFS namenode HA模式


要配置Alluxio在HA模式下HDFS的namenode，你应该正确配置Alluxio的服务端以访问HDFS。请注意一旦设置，你使用Alluxio客户端的应用程序不再需要任何特殊的配置。

有两种可能的方法：
- 将hadoop安装目录下的`hdfs-site.xml`和`core-site.xml`文件拷贝或者符号连接到`${ALLUXIO_HOME}/conf`目录下。确保在所有正在运行Alluxio的服务端上设置了。

- 或者，你可以在`conf/alluxio-site.properties`文件中将`alluxio.underfs.hdfs.configuration`指向`hdfs-site.xml`或者`core-site.xml`。确保配置在所有正在运行Alluxio的服务端上设置了。

```
alluxio.underfs.hdfs.configuration=/path/to/hdfs/conf/core-site.xml:/path/to/hdfs/conf/hdfs-site.xml
```

然后，如果你需要将HDFS的根目录映射到Alluxio，则将底层存储地址设为`hdfs://nameservice/`（`nameservice`是在`core-site.xml`文件中已配置的HDFS服务的名称），或者如果你仅仅需要把HDFS目录`/alluxio/data`映射到Alluxio，则将底层存储地址设置为`hdfs://nameservice/alluxio/data`。

```
alluxio.master.mount.table.root.ufs=hdfs://nameservice/
```

### 确保用户/权限映射

Alluxio支持类POSIX文件系统[用户和权限检查]({{ '/cn/operation/Security.html' | relativize_url }})，这从v1.3开始默认启用。
为了确保文件/目录的权限信息，即HDFS上的用户，组和访问模式，与Alluxio一致，(例如，在Alluxio中被用户Foo创建的文件在HDFS中也以Foo作为用户持久化)，用户**需要**以以下方式启动:

1. [HDFS超级用户](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-hdfs/HdfsPermissionsGuide.html#The_Super-User)。即，使用启动HDFS namenode进程的同一用户也启动Alluxio master和worker进程。也就是说，使用与启动HDFS的namenode进程相同的用户名启动Alluxio master和worker进程。

2. [HDFS超级用户组](http://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-hdfs/HdfsPermissionsGuide.html#Configuration_Parameter)的成员。编辑HDFS配置文件`hdfs-site.xml`并检查配置属性`dfs.permissions.superusergroup`的值。如果使用组（例如，“hdfs”）设置此属性，则将用户添加到此组（“hdfs”）以启动Alluxio进程（例如，“alluxio”）;如果未设置此属性，请将一个组添加到此属性，其中Alluxio运行用户是此新添加组的成员。

注意，上面设置的用户只是启动Alluxio master和worker进程的标识。一旦Alluxio服务器启动，就**不必**使用此用户运行Alluxio客户端应用程序。

### 安全认证模式下的HDFS

Alluxio支持安全认证模式下的HDFS作为底层文件系统，通过[Kerberos](http://web.mit.edu/kerberos/)认证。

#### Kerberos配置

可选配置项，你可以为自定义的Kerberos配置设置jvm级别的系统属性：`java.security.krb5.realm`和`java.security.krb5.kdc`。这些Kerberos配置将Java库路由到指定的Kerberos域和KDC服务器地址。如果两者都设置为空，Kerberos库将遵从机器上的默认Kerberos配置。例如：

* 如果你使用的是Hadoop，你可以将这两项配置添加到`${HADOOP_CONF_DIR}/hadoop-env.sh`文件的`HADOOP_OPTS`配置项。

```console
$ export HADOOP_OPTS="$HADOOP_OPTS -Djava.security.krb5.realm=<YOUR_KERBEROS_REALM> -Djava.security.krb5.kdc=<YOUR_KERBEROS_KDC_ADDRESS>"
```

* 如果你使用的是Spark，你可以将这两项配置添加到`${SPARK_CONF_DIR}/spark-env.sh`文件的`SPARK_JAVA_OPTS`配置项。

```bash
SPARK_JAVA_OPTS+=" -Djava.security.krb5.realm=<YOUR_KERBEROS_REALM> -Djava.security.krb5.kdc=<YOUR_KERBEROS_KDC_ADDRESS>"
```

* 如果你使用的是Alluxio Shell，你可以将这两项配置添加到`conf/alluxio-env.sh`文件的`ALLUXIO_JAVA_OPTS`配置项。

```bash
ALLUXIO_JAVA_OPTS+=" -Djava.security.krb5.realm=<YOUR_KERBEROS_REALM> -Djava.security.krb5.kdc=<YOUR_KERBEROS_KDC_ADDRESS>"
```

#### Alluxio服务器Kerberos认证

在`alluxio-site.properties`文件配置下面的Alluxio属性：

```properties
alluxio.master.keytab.file=<YOUR_HDFS_KEYTAB_FILE_PATH>
alluxio.master.principal=hdfs/<_HOST>@<REALM>
alluxio.worker.keytab.file=<YOUR_HDFS_KEYTAB_FILE_PATH>
alluxio.worker.principal=hdfs/<_HOST>@<REALM>
```

## 使用HDFS在本地运行Alluxio

在开始本步骤之前，请确保HDFS集群已经启动运行并且映射到Alluxio根目录下的HDFS目录已经存在。配置完成后，你可以在本地启动Alluxio，观察一切是否正常运行：

```console
$ ./bin/alluxio format
$ ./bin/alluxio-start.sh local
```

该命令应当会本地启动一个Alluxio master和一个Alluxio worker，可以在浏览器中访问[http://localhost:19999](http://localhost:19999)查看master Web UI。

接着，你可以运行一个简单的示例程序：

```console
$ ./bin/alluxio runTests
```

运行成功后，访问HDFS Web UI [http://localhost:50070](http://localhost:50070)，确认其中包含了由Alluxio创建的文件和目录。在该测试中，在[http://localhost:50070/explorer.html](http://localhost:50070/explorer.html)中创建的文件名称应像这样：`/default_tests_files/BASIC_CACHE_THROUGH`。

运行以下命令停止Alluxio：

```console
$ ./bin/alluxio-stop.sh local
```
