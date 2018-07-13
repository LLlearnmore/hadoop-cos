## 功能说明
Hadoop cosn 插件实现了以腾讯云 COS 作为底层存储文件系统运行上层计算任务的功能，使用 Hadoop 大数据处理引擎，如 MapReduce，Hive、Spark、Tez 等，可以处理存储在腾讯云对象存储 COS 上的数据。

## 使用限制
只适用于 COS V5 版本

## 使用环境
### 系统环境
Linux 或 Windows 系统

### 软件依赖
Hadoop-2.7.2及以上版本

### 获取 hadoop-cos 插件
下载地址：[hadoop-cos 插件](https://github.com/tencentyun/hadoop-cosn-v5)


### 插件安装方法
#### 修改 hadoop_env.sh
在 `$HADOOP_HOME/etc/hadoop`目录下，进入 hadoop_env.sh，增加如下内容，将 cosn 相关 jar 包加入 Hadoop 环境变量：

```shell
for f in $HADOOP_HOME/share/hadoop/tools/lib/*.jar; do
  if [ "$HADOOP_CLASSPATH" ]; then
    export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$f
  else
    export HADOOP_CLASSPATH=$f
  fi
done
```
将dep目录下的cos_hadoop_api-5.2.5.jar 和 hadoop-cos-2.7.2.jar 拷贝到 `$HADOOP_HOME/share/hadoop/tools/lib`下。

#### 修改配置文件使用插件
修改 $HADOOP_HOME/etc/hadoop/core-site.xml，增加 COS 相关用户和实现类信息，例如：

```xml
<configuration>

    <property>
        <name>fs.defaultFS</name>
        <value>cosn://<bucket-appid></value>
    </property>

    <property>
        <name>hadoop.tmp.dir</name>
        <value>${HADOOP_PATH}/tmp</value>
    </property>

    <property>
        <name>dfs.name.dir</name>
        <value>${HADOOP_PATH}/name</value>
    </property>

    <property>
        <name>fs.cosn.userinfo.secretId</name>
        <value>xxxxxxxxxxxxxxxxxxxxxxxxx</value>
        <description>Tencent COS Secret Id </description>
    </property>

    <property>
        <name>fs.cosn.userinfo.secretKey</name>
        <value>xxxxxxxxxxxxxxxxxxxxxxxx</value>
        <description>Tencent COS Secret Key</description>
    </property>

    <property>
        <name>fs.cosn.userinfo.region</name>
        <value>ap-guangzhou</value>
        <description>The region where the bucket is located</description>
    </property>

    <property>
        <name>fs.cosn.impl</name>
        <value>org.apache.hadoop.fs.CosFileSystem</value>
        <description>The implementation class of the CosN Filesystem</description>
    </property>

    <property>
        <name>fs.AbstractFileSystem.cosn.impl</name>
        <value>org.apache.hadoop.fs.CosN</value>
        <description>The implementation class of the CosN AbstractFileSystem.</description>
    </property>

    <property>
        <name>fs.cosn.buffer.dir</name>
        <value>${hadoop.tmp.dir}/cos</value>
        <description>The local path where the cosn filesystem should store files before write to cos.</description>
    </property>

    <property>
    	<name>fs.cosn.upload.buffer</name>
        <value>disk</value>
        <description>
        	There are two options to choose from:
            "disk": Use some temporary files as buffer pool, and use the memory mapping technology to obtain a memory access speed for buffer IO approximately.
            "memory": Use some direct byte buffers as buffer pool, and obtain a memory access speed for buffer IO.
            Default is disk mode.
        </description>
    </property>

    <property>
    	<name>fs.cosn.upload.buffer.size</name>
        <value>134217728</value>
        <description>The total size of the buffer pool</description>
    </property>

    <property>
    	<name>fs.cosn.block.size</name>
        <value>8388608</value>
        <description>Block size to use cosn filesysten, which is the part size for MultipartUpload. Considering the COS supports up to 10000 blocks, user should estimate the maximum size of a single file. for example, 8MB part size can allow  writing a 78GB single file.</description>
    </property>

    <property>
    	<name>fs.cosn.maxRetries</name>
        <value>3</value>
        <description>The maximum number of retries for reading or writing files to
    COS, before we signal failure to the application.</description>
    </property>

    <property>
    	<name>fs.cosn.retry.interval.seconds</name>
        <value>10</value>
        <description>The number of seconds to sleep between each COS retry.</description>
    </property>

</configuration>

```

**配置项说明**：

- fs.cosn.userinfo.secretId/secretKey 属性：填写您账户的API 密钥信息。可通过 [云 API 密钥 控制台](https://console.cloud.tencent.com/capi) 查看；
- fs.cosn.impl 为 cosn 的实现类，固定为 org.apache.hadoop.fs.CosFileSystem；
- fs.AbstractFileSystem.cosn.impl为CosN对AbstractFileSystem的实现类，固定为org.apache.hadoop.fs.CosN；
- fs.cosn.userinfo.region 请填写您的地域信息，枚举值为 [可用地域](https://cloud.tencent.com/document/product/436/6224) 中的地域简称，如	ap-beijing、ap-guangzhou 等。
- fs.cosn.buffer.dir 请设置一个实际存在的目录，运行过程中产生的临时文件会暂时放于此处；
- fs.cosn.upload.buffer为流式上传时，使用的缓冲区类型。当前支持两种缓冲区类型：disk和memory，其中disk将会在fs.cosn.buffer.dir选项指定的文件系统目录中生成若干个文件临时文件，并使用内存映射技术将其包装为上传缓冲池。请根据机器的磁盘和内存大小，合理选择缓冲池的形式；
- fs.cosn.upload.buffer.size 向COS上传文件时，本地使用的缓冲区的总大小；
- fs.cosn.block.size CosN文件系统每个block的大小，也是分块上传的每个part size的大小。由于COS的分块上传最多只能支持10000块，因此需要预估最大可能使用到的单文件大小。例如，block size为8MB时，最大能够支持78GB的单文件上传。 block size最大可以支持到2GB，即单文件最大可支持19TB；
- fs.cosn.upload_thread_pool 文件流式上传到COS时，并发上传的线程数目；
- fs.cosn.maxRetries 访问COS出现错误时，最多重试的次数；
- fs.cosn.retry.interval.seconds 每次重试的时间间隔

### 使用软件（以 Linux 为例）
#### 使用 hadoop fs 常用命令
命令格式为：`hadoop fs- -ls cosn://Bucket 路径`，下例中以名称为 example-1252681929 的 Bucket 为例，可在其后面加上具体路径。
```
hadoop fs -ls -R cosn://hdfs-test-1252681929/
-rw-rw-rw-   1 root root       1087 2018-06-11 07:49 cosn://hdfs-test-1252681929/LICENSE
drwxrwxrwx   - root root          0 1970-01-01 00:00 cosn://hdfs-test-1252681929/hdfs
drwxrwxrwx   - root root          0 1970-01-01 00:00 cosn://hdfs-test-1252681929/hdfs/2018
-rw-rw-rw-   1 root root       1087 2018-06-12 03:26 cosn://hdfs-test-1252681929/hdfs/2018/LICENSE
-rw-rw-rw-   1 root root       2386 2018-06-12 03:26 cosn://hdfs-test-1252681929/hdfs/2018/ReadMe
drwxrwxrwx   - root root          0 1970-01-01 00:00 cosn://hdfs-test-1252681929/hdfs/test
-rw-rw-rw-   1 root root       1087 2018-06-11 07:32 cosn://hdfs-test-1252681929/hdfs/test/LICENSE
-rw-rw-rw-   1 root root       2386 2018-06-11 07:29 cosn://hdfs-test-1252681929/hdfs/test/ReadMe
```
#### 运行 MapReduce 自带的 wordcount
> <font color="#0000cc">**注意：** </font>
以下命令中 hadoop-mapreduce-examples-2.7.2.jar 是以 2.7.2 版本为例，如版本不同，请修改成对应的版本号。

```shell
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar wordcount cosn://example/mr/input cosn://example/mr/output3
```
执行成功会返回统计信息，示例如下：
```
File System Counters
        COSN: Number of bytes read=72
        COSN: Number of bytes written=40
        COSN: Number of read operations=0
        COSN: Number of large read operations=0
        COSN: Number of write operations=0
        FILE: Number of bytes read=547350
        FILE: Number of bytes written=1155616
        FILE: Number of read operations=0
        FILE: Number of large read operations=0
        FILE: Number of write operations=0
        HDFS: Number of bytes read=0
        HDFS: Number of bytes written=0
        HDFS: Number of read operations=0
        HDFS: Number of large read operations=0
        HDFS: Number of write operations=0
    Map-Reduce Framework
        Map input records=5
        Map output records=7
        Map output bytes=59
        Map output materialized bytes=70
        Input split bytes=99
        Combine input records=7
        Combine output records=6
        Reduce input groups=6
        Reduce shuffle bytes=70
        Reduce input records=6
        Reduce output records=6
        Spilled Records=12
        Shuffled Maps =1
        Failed Shuffles=0
        Merged Map outputs=1
        GC time elapsed (ms)=0
        Total committed heap usage (bytes)=653262848
    Shuffle Errors
        BAD_ID=0
        CONNECTION=0
        IO_ERROR=0
        WRONG_LENGTH=0
        WRONG_MAP=0
        WRONG_REDUCE=0
    File Input Format Counters 
        Bytes Read=36
    File Output Format Counters 
        Bytes Written=40
```
