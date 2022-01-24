#  Flink写入HDFS

## 前因

Flink版本：1.10（因为华为云的DLI-Flink平台支持的最稳定的最高版本就是1.10）

因为本人工作需要，需要在阿里云Blink和华为云DLI-Flink两个平台上做任务迁移，

原Blink平台有部分任务的链路是这样的：

```shell
#链路
阿里云SLS日志服务->Blink(FlinkSQL)->MaxCompute内表(一级分区表/三级分区表)
```

现在任务改造为：

```shell
#链路
阿里云SLS日志服务->DLI-Flink(Jar)->OBS(华为云对象存储服务)->DLI外表(华为云的MaxCompute，同样是有一级分区表和三级分区表)

#为什么不直接写入DLI外表？
因为现在华为云没有支持

#建DLI的OBS外表和建DLI内表有什么区别？
区别类似Hive的外表和内表，可以把OBS理解为HDFS，DLI理解为Hive，当然做DLI的OBS外表的成本要贵一些
```

原因是Blink平台写FlinkSQL可以直接对接SLS和MC，而华为这边的FlinkSQL不支持连接阿里云的SLS和直接写到DLI的离线表里，只支持写OBS，最终选择了阿里提供的Flink Sreaming的SLS Connector以及Flink官方提供的FileStreamingSink算子，其实华为云官方DLI文档也是给了一个简单的demo给我们参考

Flink官方文档地址：

```html
https://nightlies.apache.org/flink/flink-docs-release-1.10/zh/dev/connectors/streamfile_sink.html
```

华为云官方文档地址：

```html
https://support.huaweicloud.com/devg-dli/dli_09_0191.html
```

## 总结一下：

我们可以使用FlieStreamingSink算子做哪些事呢？

1.写入textfile到外部文件系统（OBS、HDFS、本地磁盘）

2.写入parquet格式文件到外部文件系统（OBS、HDFS、本地磁盘）

3.写入parquet格式文件，并且选择压缩方式到外部文件系统（OBS、HDFS等支持压缩的文件系统）

补充：

1.可以自定义桶分配器

2.可以自定义文件滚动策略

3.还有其它格式可以参考Flink官网，我们这里建立DLI的OBS外表，选择parquet格式和snappy压缩

4.还可以定义文件的前缀和后缀

整个流程可以简单类比理解为，Flink从Kafka消费数据后写入HDFS，数据提交给HDFS后是parquet列存的格式，文件使用snappy进行压缩，Hive再以HDFS存储的数据作外部分区表

## 下面我们看Demo

Flink官网的Demo

```java
import org.apache.flink.api.common.serialization.SimpleStringEncoder;
import org.apache.flink.core.fs.Path;
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink;
import org.apache.flink.streaming.api.functions.sink.filesystem.rollingpolicies.DefaultRollingPolicy;

DataStream<String> input = ...;

final StreamingFileSink<String> sink = StreamingFileSink
    .forRowFormat(new Path(outputPath), new SimpleStringEncoder<String>("UTF-8"))
    .withRollingPolicy(
        DefaultRollingPolicy.builder()
            .withRolloverInterval(TimeUnit.MINUTES.toMillis(15))
            .withInactivityInterval(TimeUnit.MINUTES.toMillis(5))
            .withMaxPartSize(1024 * 1024 * 1024)
            .build())
	.build();

input.addSink(sink);
```

华为云的Demo

```java
import org.apache.flink.api.common.serialization.SimpleStringEncoder; 
import org.apache.flink.api.common.serialization.SimpleStringSchema; 
import org.apache.flink.api.java.utils.ParameterTool; 
import org.apache.flink.contrib.streaming.state.RocksDBStateBackend; 
import org.apache.flink.core.fs.Path; 
import org.apache.flink.runtime.state.filesystem.FsStateBackend; 
import org.apache.flink.streaming.api.datastream.DataStream; 
import org.apache.flink.streaming.api.environment.CheckpointConfig; 
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment; 
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink; 
import org.apache.flink.streaming.api.functions.sink.filesystem.bucketassigners.DateTimeBucketAssigner; 
import org.apache.flink.streaming.api.functions.sink.filesystem.rollingpolicies.OnCheckpointRollingPolicy; 
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer; 
import org.apache.kafka.clients.consumer.ConsumerConfig; 
import org.slf4j.Logger; 
import org.slf4j.LoggerFactory; 

import java.util.Properties; 

/** 
 * @author xxx 
 * @date 6/26/21 
 */ 
public class FlinkKafkaToObsExample { 
    private static final Logger LOG = LoggerFactory.getLogger(FlinkKafkaToObsExample.class); 

    public static void main(String[] args) throws Exception { 
        LOG.info("Start Kafka2OBS Flink Streaming Source Java Demo."); 
        ParameterTool params = ParameterTool.fromArgs(args); 
        LOG.info("Params: " + params.toString()); 

        // Kafka连接地址 
        String bootstrapServers; 
        // Kafka消费组 
        String kafkaGroup; 
        // Kafka topic 
        String kafkaTopic; 
        // 消费策略，只有当分区没有Checkpoint或者Checkpoint过期时，才会使用此配置的策略； 
        //          如果存在有效的Checkpoint，则会从此Checkpoint开始继续消费 
        // 取值有： LATEST,从最新的数据开始消费，此策略会忽略通道中已有数据 
        //         EARLIEST,从最老的数据开始消费，此策略会获取通道中所有的有效数据 
        String offsetPolicy; 
        // OBS文件输出路径，格式obs://ak:sk@obsEndpoint/bucket/path 
        String outputPath; 
        // Checkpoint输出路径，格式obs://ak:sk@obsEndpoint/bucket/path 
        String checkpointPath; 

        bootstrapServers = params.get("bootstrap.servers", "xxxx:9092,xxxx:9092,xxxx:9092"); 
        kafkaGroup = params.get("group.id", "test-group"); 
        kafkaTopic = params.get("topic", "test-topic"); 
        offsetPolicy = params.get("offset.policy", "earliest"); 
        outputPath = params.get("output.path", "obs://ak:sk@obs.cn-north-1.huaweicloud.com:443/bucket/output"); 
       checkpointPath = params.get("checkpoint.path", "obs://ak:sk@obs.cn-north-1.huaweicloud.com:443/bucket/checkpoint"); 

        try { 
            // 创建执行环境 
            StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment(); 
            RocksDBStateBackend rocksDbBackend = new RocksDBStateBackend(new FsStateBackend(checkpointPath), true); 
            streamEnv.setStateBackend(rocksDbBackend); 
            // 开启Flink CheckPoint配置，开启时若触发CheckPoint，会将Offset信息同步到Kafka 
            streamEnv.enableCheckpointing(300000); 
            // 设置两次checkpoint的最小间隔时间 
            streamEnv.getCheckpointConfig().setMinPauseBetweenCheckpoints(60000); 
            // 设置checkpoint超时时间 
            streamEnv.getCheckpointConfig().setCheckpointTimeout(60000); 
            // 设置checkpoint最大并发数 
            streamEnv.getCheckpointConfig().setMaxConcurrentCheckpoints(1); 
            // 设置作业取消时保留checkpoint 
            streamEnv.getCheckpointConfig().enableExternalizedCheckpoints( 
                    CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION); 

            // Source: 连接kafka数据源 
            Properties properties = new Properties(); 
            properties.setProperty("bootstrap.servers", bootstrapServers); 
            properties.setProperty("group.id", kafkaGroup); 
            properties.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, offsetPolicy); 
            String topic = kafkaTopic; 

            // 创建kafka consumer 
            FlinkKafkaConsumer<String> kafkaConsumer = 
                    new FlinkKafkaConsumer<>(topic, new SimpleStringSchema(), properties); 
            /** 
             * 从 Kafka brokers 中的 consumer 组（consumer 属性中的 group.id 设置）提交的偏移量中开始读取分区。 
             * 如果找不到分区的偏移量，那么将会使用配置中的 auto.offset.reset 设置。 
             * 详情 https://ci.apache.org/projects/flink/flink-docs-release-1.13/zh/docs/connectors/datastream/kafka/ 
             */ 
            kafkaConsumer.setStartFromGroupOffsets(); 

            //将kafka 加入数据源 
            DataStream<String> stream = streamEnv.addSource(kafkaConsumer).setParallelism(3).disableChaining(); 

            // 创建文件输出流 
            final StreamingFileSink<String> sink = StreamingFileSink 
                    // 指定文件输出路径与行编码格式 
                    .forRowFormat(new Path(outputPath), new SimpleStringEncoder<String>("UTF-8")) 
                    // 指定文件输出路径批量编码格式，以parquet格式输出 
                    //.forBulkFormat(new Path(outputPath), ParquetAvroWriters.forGenericRecord(schema)) 
                    // 指定自定义桶分配器 
                    .withBucketAssigner(new DateTimeBucketAssigner<>()) 
                    // 指定滚动策略 
                    .withRollingPolicy(OnCheckpointRollingPolicy.build()) 
                    .build(); 

            // Add sink for DIS Consumer data source 
            stream.addSink(sink).disableChaining().name("obs"); 

            // stream.print(); 
            streamEnv.execute(); 
        } catch (Exception e) { 
            LOG.error(e.getMessage(), e); 
        } 
    } 
}
```

即然华为云给了Demo，说明StreamingFileSink算子是可以写入OBS的，我们可以看到Flink官方的Demo缺少了很多细节；
StreamingFileSink中第一个.forRowFormat()方法，一个参数是路径，一个参数是编码器，这样使用写入到OBS的文件就是可读的textfile文件，对应Hive表的行存模式；
withRollingPolicy()方法故名思义，即滚动策略，可以使用DefaultRollingPolicy进行定义滚动策略的时间、间隔和文件大小，也可以使用OnCheckpointRollingPolicy，即当提交checkpoint成功时，进行滚动文件，当然还可以自定义，自定义的使用有待有空进行研究；
withBucketAssigner()方法，可以使用DateTimeBucketAssigner自定义年月日，最低到小时的文件桶分配，也可以使用BasePathBucketAssigner，这样生成的文件都会在你提供的路径下，当然也可以自定义BuccketAssigner，只需要用匿名内部类重写方法即可

但是这两种方式都不是我们想要的，因为我们要以parquet的格式写入文件，那么，再看官网这个Demo

```java
import org.apache.flink.streaming.api.functions.sink.filesystem.StreamingFileSink;
import org.apache.flink.formats.parquet.avro.ParquetAvroWriters;
import org.apache.avro.Schema;


Schema schema = ...;
DataStream<GenericRecord> stream = ...;

final StreamingFileSink<GenericRecord> sink = StreamingFileSink
	.forBulkFormat(outputBasePath, ParquetAvroWriters.forGenericRecord(schema))
	.build();

input.addSink(sink);
```

将avro数据以parquet格式写入文件，使用的是forBulkFormat，官网也提示了，这种方法只支持 OnCheckpointRollingPolicy滚动策略

```shell
重要: 批量编码模式仅支持 OnCheckpointRollingPolicy 策略, 在每次 checkpoint 的时候切割文件。
```

那我们需要再定义一个文件滚动策略即可实现，因为一级分区表和三级分区表在HDFS和OBS上都是以子目录来区分，比如这样

```sql
/warehouse/test_table_a/dt=20211219
/warehouse_test_table_b/year=2021/month=12/day=19

即test_table_a是一个一级分区表，它有一个分区是dt=20211219
test_table_b是一个三级分区表，它有一个分区是year=2021 and month = 12 and day = 19
```

然后需要解决的一个问题是，Flink官方Demo里面，在forGenericRecord(schema)中传了一个schema，那这个schema是什么呢？我们看第6行，官网直接给省略了，那我们看导包信息，原来是org.apache.avro.Schema，导入相关的包后，我们看到这个Schema，其实是定义avro数据的一个Schema类，这里我们需要了解一个avro这个序列化技术，详细可以参考官网

avro官网，直接用浏览器的翻译功能对比看看就好

```html
https://avro.apache.org/
```

我们要把上游传出来的数据封装成一个avro类才能进行我们FileStreamingSink算子的工作，如何定义一个avro的schema，通过avro官网学习，原来可以定义一个后缀为avsc的json文件来定义我们的avro类

```java
Schema schema = new Schema.Parser().parse(Demo.class.getClassLoader().getResourceAsStream("AvroData.avsc"));
```

这样，我们就可以使用ParquetAvroWriters.forGenericRecord(schema)方法了，注意的是，上游要把数据封装成GenericRecord类

当然，通过Idea, 我们点开ParquetAvroWriters，我们发现，还有一种方法是

```java
public static <T> ParquetWriterFactory<T> forReflectRecord(Class<T> type){}
```

可以将上游数据封装成我们自定义的JavaBean类，然后在forReflectRecord()方法中传入我们定义的Bean.class即可在方法内部自动通过反射的方式生成我们自定义类的avro schema了。

但是，你以为这样就万事大吉了吗？No,No,No,笔者在测试程序的时候，发现我们上游本来有一些没有的数据，因为离线表的字段是固定死的，没有写入的数据就是null值，而我们的程序呢？在上游处理数据的时候，如果这个字段没有赋值，那么会报空指针异常，如果你以为用\N来替换null值，那表里本来是null的字段，会变成字符串\N（为什么是\N呢，因为Hive默认空值存储为\N），这个其实用写textfile的方式，将null写为\N,Hive建表后是可以识别的，但是写parquet文件就不行。那怎么办呢？其实我们可以自定义avro的schema，然后将字段的type定义为联合（avro schema类型的一种，详见官网），联合中给一个null类型的即可，而且这个null一定要放在最前边，然后使用avro-tools，对我们的bean.avsc文件进行命令行操作，或者使用avro-maven插件进行打包，都会生成一个avro_bean类，我们将这个.java文件导入我们的项目中，使用这个avro类进行数据封装，然后传入avro_bean.class到forReflectRecord方法即可，关于这个avro_bean类，有待后续再深入研究学习。

avro官网的解释

```shell
(Note that when a default value is specified for a record field whose type is a union, the type of the default value must match the first element of the union. Thus, for unions containing "null", the "null" is usually listed first, since the default value of such unions is typically null.)
（请注意，当为类型为union的记录字段指定默认值时，默认值的类型必须与union的第一个元素匹配。因此，对于包含"null"的union，通常首先列出"null"，因为此类union的默认值通常为空。)
```

如何定义一个avro schema

```json
{"type":"record",#avro schema中的一种类型，这里对应一个类，record
  "name":"AvroData",#类名
  "namespace":"cn.shihuo.avro",#包名，可以没有，但是最好写上写准确
  "fields":[#字段详细信息
    {"name":"id","type":"string"},#指定一个String source;
    {"name":"name","type":["null","string"]}#指定一个String name;且可以为空，注意null要写在前面
  ]
}
```

avro-tools工具类的下载地址

```html
<!--可以maven-repository网站搜索avro-tools，选择对应版本点进去，点击file栏的jar进行下载-->
https://repo1.maven.org/maven2/org/apache/avro/avro-tools/1.11.0/avro-tools-1.11.0.jar
<!--maven-repository中的avro-tools-->
https://mvnrepository.com/artifact/org.apache.avro/avro-tools
```

avro-tools工具类的使用，注意avro-tools工具类版本要和avro包的版本一致才行

```shell
java -jar /path/to/avro-tools-1.11.0.jar compile schema <schema file> <destination>

java -jar /path/to/avro-tools-1.11.0.jar compile schema bean.avsc .
```

avro-maven插件的使用

```xml
<!--可以参考avro官网，以使用maven打包时，会先生成这个avro schema的类，再进行打包-->
<plugin>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro-maven-plugin</artifactId>
    <version>1.10.2</version><!--目前官网可以看到有两个版本>
    <executions>
        <execution>
            <phase>generate-sources</phase>
            <goals>
                <goal>schema</goal>
            </goals>
            <configuration>
                <sourceDirectory>${project.basedir}/src/main/resources/</sourceDirectory><!--这里指定xxx.avsc文件的目录-->
                <outputDirectory>${project.basedir}/src/main/java/</outputDirectory><!--这里指定avro bean生成的目录-->
            </configuration>
        </execution>
    </executions>
</plugin>
```

笔者在测试的时候，也是踩坑无数，花费大量时间研究这个问题，同时也请教了华为云的研发同学，华为云研发同学呢，在认真听了我的描述以及我做的一些研究工作并在我做了演示之后呢，也是经过一番工作，终于提出我们最后一种解决方案，当时，华为云的同学提出要把生成的avro类中的一个customDecode()方法中的in.readeFieldOrderIfDiff()方法换成另一个方法readFiledOrder方法，因为华为云的同学在拿到我的脱敏Demo之后打包时这里报错了，然而笔者没有发生这个错误，而且经过我的测试，两种方法没有测出有什么不一样来，都可以写入null值，并被数仓的外表正确识别到。

需要注意的一点是，数据写入后，外表是没有新分区数据的元数据的，需要进行下面的操作

```sql
--以下三种任选一种，都需要上调度，按照分区表的业务需求适时增加分区的元数据信息
--增加新分区，其实是增加外表新分区的元数据，比后者速度性能要好
alter table test_table_a add partition(dt = 20211219);
alter table test_table_b add partition(year = 2021,mont=12,day=19);
--修复分区表元数据信息
alter table test_table recover partitions;(华为云DLI提供的语法)
或者
MSCK REPAIR TABLE test_table;(华为云DLI支持，Hive也支持)

```

还需要注意的是，不管是写入本地磁盘，还是HDFS，还是OBS，写入的文件是有三种状态的，只有第三种状态的文件才是真正可用的能被Hive等工具读取到，而以parquet格式写入的时候，文件滚动策略是只能选择依赖于Flink checkpoint的，也就是说文件并不是以流式数据实时写入的，其实还是批处理，不管你的checkpoint1秒一次还是1分钟一次还是1小时一次（估计生产上都是几分钟或者几十分钟一次吧），而且sink算子的并行度有多少，一次checkpoint写入的文件就有多少，这个需要实际确认我们生产环境中数据量的情况，选择一个合适的checkpoin时间和sink并行度，可以减少小文件写入以及提高程序的稳定性。当然，也可以自定义滚动策略，（Flink1.12到Flink1.15的文档说了，批量编码模式必须使用继承自CheckpointRollingPolicy的滚动策略，这些策略必须在每次checkpoint的时候滚动文件，但是用户也可以进一步指定额外的基于文件大小和超时时间的策略）具体能不能成功，需要再继续研究一下。

```html
Flink1.14的文档
https://nightlies.apache.org/flink/flink-docs-release-1.14/zh/docs/connectors/datastream/file_sink/
```



到这一步，我们已经大体完成需求了，只需要解决压缩的问题就好了，那么怎么做呢？
这里感谢群里大佬提供的工具类，我参考这个工具类写了一个我们需要的版本，如下

```java
import org.apache.avro.Schema;
import org.apache.avro.generic.GenericData;
import org.apache.avro.reflect.ReflectData;
import org.apache.flink.formats.parquet.ParquetBuilder;
import org.apache.flink.formats.parquet.ParquetWriterFactory;
import org.apache.parquet.avro.AvroParquetWriter;
import org.apache.parquet.hadoop.ParquetWriter;
import org.apache.parquet.hadoop.metadata.CompressionCodecName;
import org.apache.parquet.io.OutputFile;

import java.io.IOException;
public class MyParquetAvroWriters {

    /**
     *
     * @param type
     * @param compressionCodecName CompressionCodecName.(UNCOMPRESSED,SNAPPY,GZIP,LZO,BROTLI,LZ4,ZSTD)
     * @param <T>
     * @return
     */

    /*
      public enum CompressionCodecName {
      UNCOMPRESSED(null, CompressionCodec.UNCOMPRESSED, ""),
      SNAPPY("org.apache.parquet.hadoop.codec.SnappyCodec", CompressionCodec.SNAPPY, ".snappy"),
      GZIP("org.apache.hadoop.io.compress.GzipCodec", CompressionCodec.GZIP, ".gz"),
      LZO("com.hadoop.compression.lzo.LzoCodec", CompressionCodec.LZO, ".lzo"),
      BROTLI("org.apache.hadoop.io.compress.BrotliCodec", CompressionCodec.BROTLI, ".br"),
      LZ4("org.apache.hadoop.io.compress.Lz4Codec", CompressionCodec.LZ4, ".lz4"),
      ZSTD("org.apache.hadoop.io.compress.ZStandardCodec", CompressionCodec.ZSTD, ".zstd");
    */
    public static <T> ParquetWriterFactory<T> forReflectRecord(Class<T> type,
                                                               CompressionCodecName compressionCodecName) {
        final String schemaString = ReflectData.get().getSchema(type).toString();
        final ParquetBuilder<T> builder = (out) -> createAvroParquetWriter(schemaString,
                ReflectData.get(), out, compressionCodecName);
        return new ParquetWriterFactory<>(builder);
    }

    private MyParquetAvroWriters(){}

    private static <T> ParquetWriter<T> createAvroParquetWriter(
            String schemaString,
            GenericData dataModel,
            OutputFile out,
            CompressionCodecName compressionCodecName) throws IOException {
        final Schema schema = new Schema.Parser().parse(schemaString);
        return AvroParquetWriter.<T>builder(out)//使用AvroParquetWriter，细节大家可以点进这个类里阅读源码学习
                .withSchema(schema)
                .withDataModel(dataModel)
                .withCompressionCodec(compressionCodecName)
                .build();
    }
}
```

使用方法

```java
        /*
        OutputFileConfig fileNameConfig = OutputFileConfig
                .builder()
                .withPartPrefix("前缀-")//前缀
                .withPartSuffix("-后缀")//后缀
                .build();
         */

StreamingFileSink<AvroData> sink = StreamingFileSink
                //指定文件输出路径指编码格式，以parquet格式输出
                .forBulkFormat(new Path("hdfs://xxx"), MyParquetAvroWriters.forReflectRecord(AvroData.class, CompressionCodecName.SNAPPY))
 
                .withBucketAssigner(new BucketAssigner<AvroData, String>() {//自定义桶分配器
                    @Override
                    public String getBucketId(AvroData element, Context context) {
                        String time = element.getTime().toString();//使用数据中的事件时间作为分区依据
                        return "dt=" + DateUtil.ts2yyyyMMdd(time);//分区的子路径前面必须是分区字段名+'='+分区值
                        //三级分区
                        //return "year=" + element.getYear() +"/month=" + element.getMonth() + "/day=" + element.getDay();
                    }
                    @Override
                    public SimpleVersionedSerializer<String> getSerializer() {//使用默认的序列化器
                        return SimpleVersionedStringSerializer.INSTANCE;
                    }
                })
                .withRollingPolicy(OnCheckpointRollingPolicy.build())//当CheckPoint完成的时候，本次写入OBS数据才会从隐藏文件转换为可读的非隐藏文件
                //.withOutputFileConfig(fileNameConfig)//添加文件名前后缀，可选
                .build();
```



## 下面我们写几个真实案例供大家参考

我的maven pom.xml是这样的

```xml
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!--Flink 版本 华为云 只有1.10版本Flink才能parquet写入OBS，1.11版本报错未知-->
        <flink.version>1.10.0</flink.version>
        <!--JDK 版本-->
        <java.version>1.8</java.version>
        <!--Scala 2.11 版本-->
        <scala.binary.version>2.11</scala.binary.version>
        <slf4j.version>2.15.0</slf4j.version>
        <!-- log4j 2.15.0 已解决漏洞 -->
        <log4j.version>2.15.0</log4j.version>
        <!-- log4j 2.x 2.15.0之前有远程加载类的风险 -->
        <!-- <log4j.version>2.10.0</log4j.version> -->
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencies>

        <!-- flink -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- flink 1.11开始要加client包，否则无法本地运行-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>


        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-statebackend-rocksdb_2.11</artifactId>
            <version>${flink.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- 以parquet格式批量写OBS必须导入该包-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-parquet_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>

		<!--这个可以不用导入-->
<!--    
		<dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-filesystem_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
-->

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-avro</artifactId>
            <version>${flink.version}</version>
        </dependency>

	    <!-- 在StreamingFileSink算子中直接指定schema的方式时，并且本地运行时需要导入，还需要在代码中重新指定env的kryo序列化器-->
        <!-- 参考 https://stackoverflow.com/questions/65841025/flink-throwing-com-esotericsoftware-kryo-kryoexception-java-lang-unsupportedope-->
<!--        
		<dependency>
            <groupId>de.javakaffee</groupId>
            <artifactId>kryo-serializers</artifactId>
            <version>0.45</version>
        </dependency>
-->

        <!-- 使用的avro-tools版本要avro版本一致 -->
        <dependency>
            <groupId>org.apache.avro</groupId>
            <artifactId>avro</artifactId>
            <version>1.11.0</version>
        </dependency>

        <!-- 使用压缩的方式的时候，需要使用hadoop中的Pash类，故导入此包，导入avro-tools包也可行-->
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>3.3.1</version>
        </dependency>

		<!--kryo包，可以不用导入-->
<!--        
		<dependency>
            <groupId>com.esotericsoftware</groupId>
            <artifactId>kryo</artifactId>
            <version>4.0.0</version>
        </dependency>
-->

        <!-- https://mvnrepository.com/artifact/org.apache.avro/avro-tools -->
        <!--使用StreamingFileSink + schema的方式本地运行不导入会失败-->
<!--        
		<dependency>
            <groupId>org.apache.avro</groupId>
            <artifactId>avro-tools</artifactId>
            <version>1.11.0</version>
            <version>1.10.2</version>
        </dependency>-->

        <dependency>
            <groupId>org.apache.parquet</groupId>
            <artifactId>parquet-avro</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.12</version>
        </dependency>

        <!--  kafka  -->
        <!--        <dependency>-->
        <!--            <groupId>org.apache.flink</groupId>-->
        <!--            <artifactId>flink-connector-kafka_2.11</artifactId>-->
        <!--            <version>${flink.version}</version>-->
        <!--        </dependency>-->
        
        <!--  logging  -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-slf4j-impl</artifactId>
            <version>${slf4j.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>${log4j.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>${log4j.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-jcl</artifactId>
            <version>${log4j.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- 阿里云SLS连接器需要，最新版本0.1.27，文档是0.1.13 -->
        <dependency>
            <groupId>com.aliyun.openservices</groupId>
            <artifactId>flink-log-connector</artifactId>
            <version>0.1.27</version>
        </dependency>

        <!-- 阿里云SLS连接器需要 -->
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>2.5.0</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.3.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>cn.shihuo.app.ODS_Exposure_SLS_Null_Demo</mainClass>
                        </manifest>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
            </plugin>
			<!-- 这是avro-maven插件，如果不想用，也可以在命令行手动生成avro bean类导入粘贴到项目中使用-->
<!--            
			<plugin>
                <groupId>org.apache.avro</groupId>
                <artifactId>avro-maven-plugin</artifactId>
                <version>1.10.2</version>
                <executions>
                    <execution>
                        <phase>generate-sources</phase>
                        <goals>
                            <goal>schema</goal>
                        </goals>
                        <configuration>
                            <sourceDirectory>${project.basedir}/src/main/resources/</sourceDirectory>
                            <outputDirectory>${project.basedir}/src/main/java/</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
-->
        </plugins>
        <resources>
            <resource>
<!-- 
				<directory>../main/config</directory>
-->
                <directory>src/main/resources</directory><!-- 需要注意，默认读取的配置文件路径要在这里写，写对 -->
                <filtering>true</filtering>
                <includes>
                    <include>**/*.*</include>
                </includes>
            </resource>
        </resources>
    </build>

</project>
```
