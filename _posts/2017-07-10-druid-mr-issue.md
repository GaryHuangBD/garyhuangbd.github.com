---
layout: post
title:  "Druid使用MR的问题"
date:   2017-07-10 17:27:00
categories: Druid
tags: Druid问题 MR
---

* content
{:toc}

关于MR离线的方式生成Druid的索引，很多人都遇到问题，druid在google上面的User Group也是很多人在问，但貌似都没有很好的解决。
我最近也遇到这些坑爹的问题，没有办法自己来分析这些问题，下面是自己关于这些问题的总结。  






## 官方的解决办法  

在开始之前先来看看官方的解决办法[(http://druid.io/docs/0.10.0/operations/other-hadoop.html)](http://druid.io/docs/0.10.0/operations/other-hadoop.html)，总结来看官方给的方案有两种：  

1.通过参数设置  

```  
"jobProperties": {
  "mapreduce.job.classloader": "true",
  "mapreduce.job.classloader.system.classes": "-javax.validation.,java.,javax.,org.apache.commons.logging.,org.apache.log4j.,org.apache.hadoop."
}
```

2.打fat包  

官方给出了三种办法来打fat jar，此处不再细说。打好包后，可以直接使用这个fat jar，不再需要其他的jar了。例如可以直接执行下面命令启动MR：  

```
java -Xmx32m \
  -Dfile.encoding=UTF-8 -Duser.timezone=UTC \
  -classpath config/hadoop:config/overlord:config/_common:$SELF_CONTAINED_JAR:$HADOOP_DISTRIBUTION/etc/hadoop \
  -Djava.security.krb5.conf=$KRB5 \
  io.druid.cli.Main index hadoop \
  $config_path
```

### 存在问题  

作为一个有一定审美的程序猿，直接放弃第2种方案，太丑陋了。如果使用第1种方案，在使用druid-orc-extensions模块的时候，总是会报以下错误：  

```
Error: java.lang.RuntimeException: readObject can't find class
	at org.apache.hadoop.mapreduce.lib.input.TaggedInputSplit.readClass(TaggedInputSplit.java:136)
	at org.apache.hadoop.mapreduce.lib.input.TaggedInputSplit.readFields(TaggedInputSplit.java:120)
	at org.apache.hadoop.io.serializer.WritableSerialization$WritableDeserializer.deserialize(WritableSerialization.java:71)
	at org.apache.hadoop.io.serializer.WritableSerialization$WritableDeserializer.deserialize(WritableSerialization.java:42)
	at org.apache.hadoop.mapred.MapTask.getSplitDetails(MapTask.java:372)
	at org.apache.hadoop.mapred.MapTask.runNewMapper(MapTask.java:754)
	at org.apache.hadoop.mapred.MapTask.run(MapTask.java:341)
	at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:164)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1657)
	at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:158)
Caused by: java.lang.ClassNotFoundException: Class org.apache.hadoop.hive.ql.io.orc.OrcNewSplit not found
	at org.apache.hadoop.conf.Configuration.getClassByName(Configuration.java:2101)
	at org.apache.hadoop.mapreduce.lib.input.TaggedInputSplit.readClass(TaggedInputSplit.java:134)
```

### 原因分析  

Hadoop的task有两个类加载器，ApplicationClassLoader和JobClassLoader  
`mapreduce.job.classloader`为`true`表示每个Task的jvm使用独立的classLoader，对应ApplicationClassLoader   
`mapreduce.job.classloader.system.classes`表示哪些类使用JobClassLoader，而不使用独立的ApplicationClassLoader  

由于Druid使用了很多第三方的依赖包，这些jar很多与Hadoop的依赖的第三方包不一致，特别是Jacson的依赖，为了避免jar冲突，所以需要设置`mapreduce.job.classloader`为`true`。  

由于第1种解决方案中`mapreduce.job.classloader.system.classes`包含了`org.apache.hadoop.`，
而`org.apache.hadoop.hive.ql.io.orc.OrcNewSplit`正好在`org.apache.hadoop.`里面，同时`mapreduce.job.classloader`为`true`，只有ApplicationClassLoader
才含有`org.apache.hadoop.hive.ql.io.orc.OrcNewSplit`，JobClassLoader是没有这个类的，因此会出现找不到类的错。  

## 我的解决办法  

1.设置参数  

```
"jobProperties": {
  "mapreduce.job.classloader": "true",
  "mapreduce.job.classloader.system.classes": "-javax.validation.,java.,javax.,org.apache.commons.logging.,org.apache.log4j."
}
```

这个地方做个特别说明：`"mapreduce.job.classloader": "true"有bug，在hadoop2.6.0之后的版本才修复`。具体可见：[https://issues.apache.org/jira/browse/MAPREDUCE-5957](https://issues.apache.org/jira/browse/MAPREDUCE-5957)。
在使用时，切记hadoop版本在2.6.0以上。

2.MR程序时候不上传Hadoop的jar  

考虑到一些模块依赖不同的hadoop版本，例如`druid-hdfs-storage`依赖hadoop2.3，可能会引发MR不同版本的兼容性，所以在此处我修改了Druid的jobHelp类：  

```
public static void setupClasspath(
      final Path distributedClassPath,
      final Path intermediateClassPath,
      final Job job
  )
      throws IOException
  {
    ...
    for (String jarFilePath : jarFiles) {

      final File jarFile = new File(jarFilePath);
      if (jarFile.getName().startsWith("hadoop-") || jarFilePath.contains("hadoop-client")) {
        continue;
      }
    ...
```

另外需要注意的是最好保证`druid.indexer.task.defaultHadoopCoordinates=["org.apache.hadoop:hadoop-client:{version}"]`中的version与Hadoop集群的一致。

3.druid-orc-extensions的pom中hadoop的依赖改为`provided`，然后exclude掉所有的hadoop依赖，另外还需要exclude掉calcite-avatica，因为这个jar里面包含jackson的包，
导致覆盖了jackson包里面的类，会出现`java.lang.VerifyError: class com.fasterxml.jackson.datatype.guava.deser.HostAndPortDeserializer overrides final method deserialize.`的错误。以下是我的pom：  

```
<dependencies>
        <dependency>
            <groupId>io.druid</groupId>
            <artifactId>druid-indexing-hadoop</artifactId>
            <version>${project.parent.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-exec</artifactId>
            <version>${hive.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>commons-codec</groupId>
                    <artifactId>commons-codec</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>commons-httpclient</groupId>
                    <artifactId>commons-httpclient</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>commons-io</groupId>
                    <artifactId>commons-io</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-1.2-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.logging.log4j</groupId>
                    <artifactId>log4j-slf4j-impl</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.commons</groupId>
                    <artifactId>commons-compress</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-common</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-archives</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-mapreduce-client-core</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-mapreduce-client-common</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-hdfs</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-yarn-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-yarn-common</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-yarn-client</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-yarn-server-resourcemanager</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.curator</groupId>
                    <artifactId>curator-framework</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.curator</groupId>
                    <artifactId>apache-curator</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.calcite</groupId>
                    <artifactId>calcite-avatica</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-annotations</artifactId>
            <version>${hadoop.compile.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>${hadoop.compile.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-orc</artifactId>
            <version>${hive.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.google.inject</groupId>
            <artifactId>guice</artifactId>
        </dependency>
    </dependencies>
```

## 总结

为了避免Druid与Hadoop包冲突，主要做两件事情：  

1.上传jar时，不上传hadoop相关的jar；  
2.避免Druid引入的第三方包之间有冲突。