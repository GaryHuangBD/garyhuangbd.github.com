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

2.不上传Hadoop的jar  

如果依赖Hadoop的模块，pom中依赖改为`provided`，则不需要这一步。主要考虑到一些模块依赖不同的hadoop版本，例如`druid-hdfs-storage`依赖hadoop2.3，可能会引发MR不同版本的兼容性，所以在此处我修改了Druid的jobHelp类： 

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

## 2017-07-13 更新  

出现一个很奇怪的问题，druid在提交mr的时候不停的出现一下问题：  

```
Failing over to rm1
Failing over to rm2
Failing over to rm1
Failing over to rm2
Failing over to rm1
```

### 问题原因  

在提交MR的时候，需要从zookeeper获取主ResouceManager，然后得到的`rm1`，然后查找`yarn.resourcemanager.address.rm1`，这个时候找不到这个参数，默认的就访问`0.0.0.0:8032`，所以无法连接到ResouceManager，就会出现上面的问题，尝试切换到其他ResouceManager。  

### 解决办法  
修改druid classpath下的yarn-site.xml，增加一下参数：  

```
    <property>
          <name>yarn.resourcemanager.address.rm1</name>
          <value>hd-node-1:8032</value>
    </property>

    <property>
          <name>yarn.resourcemanager.address.rm2</name>
          <value>hd-node-2:8032</value>
    </property>

    <property>
          <name>yarn.resourcemanager.admin.address.rm1</name>
          <value>hd-node-1:8033</value>
    </property>

    <property>
          <name>yarn.resourcemanager.admin.address.rm2</name>
          <value>hd-node-2:8033</value>
    </property>
    <property>
          <name>yarn.resourcemanager.resource-tracker.address.rm1</name>
          <value>hd-node-1:8031</value>
    </property>

    <property>
          <name>yarn.resourcemanager.resource-tracker.address.rm2</name>
          <value>hd-node-2:8031</value>
    </property>

    <property>
          <name>yarn.resourcemanager.scheduler.address.rm1</name>
          <value>hd-node-1:8030</value>
    </property>

    <property>
          <name>yarn.resourcemanager.scheduler.address.rm2</name>
          <value>hd-node-2:8030</value>
    </property>
```

说明：`hd-node-1`和`hd-node-2`分别对应的rm1和rm2。  

## 2017-07-26 更新  

最近druid在提交MR作业的过程中又出现了两个问题，现在总结下。  

1. 发现druid的task会提交后，直接失败，在ResourceManager ui上发现AppMaster的Container中出现以下错误：  

```
log4j:ERROR setFile(null,true) call failed.
java.io.FileNotFoundException: /hadoop/yarn/log/application_***/container_**** (Is a directory)
    at java.io.FileOutputStream.open0(Native Method)
    at java.io.FileOutputStream.open(FileOutputStream.java:270)
    at java.io.FileOutputStream.<init>(FileOutputStream.java:213)
    at java.io.FileOutputStream.<init>(FileOutputStream.java:133)
    at org.apache.log4j.FileAppender.setFile(FileAppender.java:294)
    at org.apache.log4j.FileAppender.activateOptions(FileAppender.java:165)
    at org.apache.hadoop.yarn.ContainerLogAppender.activateOptions(ContainerLogAppender.java:55)
    at org.apache.log4j.config.PropertySetter.activate(PropertySetter.java:307)
    at org.apache.log4j.config.PropertySetter.setProperties(PropertySetter.java:172)
    at org.apache.log4j.config.PropertySetter.setProperties(PropertySetter.java:104)
    at org.apache.log4j.PropertyConfigurator.parseAppender(PropertyConfigurator.java:842)
    at org.apache.log4j.PropertyConfigurator.parseCategory(PropertyConfigurator.java:768)
```

### 原因分析    

我排查了一下Hadoop代码，这个问题应该是Log4j问题，如果我们仔细留意会发现在`hadoop-yarn-server-nodemanager-{version}.jar`里面有一个container-log4j.properties文件，然后如果我们在留意MR运行时生成的launch_container.sh的脚本里(或者用命令查看AppMaster进程的启动参数)，会有一个参数`-Dlog.configurationFile=container-log4j.properties`，这说明AppMaster进程使用的log4j是container-log4j.properties。  
container-log4j.properties中有一项非常重要的配置`log4j.appender.CLA.containerLogFile=${hadoop.root.logfile}`，而我们查看整个文件，却没有`hadoop.root.logfile`具体的值。知道这个，那我们自然很容易解决问题了。  

### 解决办法  

为了解决这个问题，只需要在AppMaster启动进程中加上`-Dhadoop.root.logfile=syslog`即可，而Hadoop的参数中正好有这一项，`yarn.app.mapreduce.am.command-opts`，我们可以直接修改Hadoop的mapred-site.xml，也可以在druid的task spec文件中直接增加：  
``` 
"jobProperties": {
  "mapreduce.job.classloader": "true",
  "yarn.app.mapreduce.am.command-opts": "-Xmx2048m -Dhadoop.root.logfile=syslog",
  "mapreduce.job.classloader.system.classes": "-javax.validation.,java.,javax.,org.apache.commons.logging.,org.apache.log4j.,org.apache.hadoop."
}
```

2. 引入druid-orc-extensions后，导入OrcFile的数据过程中又出现了一个问题：  
```
ERROR [main] org.apache.hadoop.mapred.YarnChild: Error running child : com.google.common.util.concurrent.ExecutionError: java.lang.IncompatibleClassChangeError: Implementing class
    at com.google.common.cache.LocalCache$Segment.get(LocalCache.java:2199)
    at com.google.common.cache.LocalCache.get(LocalCache.java:3934)
    at com.google.common.cache.LocalCache.getOrLoad(LocalCache.java:3938)
    at com.google.common.cache.LocalCache$LocalLoadingCache.get(LocalCache.java:4821)
    at com.google.common.cache.LocalCache$LocalLoadingCache.getUnchecked(LocalCache.java:4827)
    at com.google.inject.internal.FailableCache.get(FailableCache.java:48)
    at com.google.inject.internal.MembersInjectorStore.get(MembersInjectorStore.java:68)
    at com.google.inject.internal.InjectorImpl.getMembersInjector(InjectorImpl.java:993)
    at com.google.inject.internal.InjectorImpl.getMembersInjector(InjectorImpl.java:1000)
    at com.google.inject.internal.InjectorImpl.injectMembers(InjectorImpl.java:986)
    at io.druid.initialization.Initialization$ModuleList.addModule(Initialization.java:397)
    at io.druid.initialization.Initialization$ModuleList.addModules(Initialization.java:416)
    at io.druid.initialization.Initialization.makeInjectorWithModules(Initialization.java:320)
    at io.druid.indexer.HadoopDruidIndexerConfig.<clinit>(HadoopDruidIndexerConfig.java:99)
    at io.druid.indexer.HadoopDruidIndexerMapper.setup(HadoopDruidIndexerMapper.java:48)
    at io.druid.indexer.DetermineHashedPartitionsJob$DetermineCardinalityMapper.setup(DetermineHashedPartitionsJob.java:224)
    at io.druid.indexer.DetermineHashedPartitionsJob$DetermineCardinalityMapper.run(DetermineHashedPartitionsJob.java:282)
    at org.apache.hadoop.mapred.MapTask.runNewMapper(MapTask.java:787)
    at org.apache.hadoop.mapred.MapTask.run(MapTask.java:341)
    at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:164)
    at java.security.AccessController.doPrivileged(Native Method)
    at javax.security.auth.Subject.doAs(Subject.java:422)
    at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1693)
    at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:158)
```

### 问题原因  

这个问题并非一定会出现，从错误上来看应该是包冲突引起的。毕竟orc依赖hive，而hive的依赖有那么多，其实真的有必要把不必要的包exclude，在排查这个问题，我并没有花费提多时间。一般解决我不清楚的问题，我第一反应是google，搜索到后基本能够快速解决。如果10分钟时间，google还帮助不到我的时候，我才会自己想办法解决。还好这个问题我直接就搜索到了：https://groups.google.com/forum/#!topic/druid-user/MHkOVmLGPx8  

### 解决办法  

直接在druid-orc-extensions删除jetty-all这个包。
