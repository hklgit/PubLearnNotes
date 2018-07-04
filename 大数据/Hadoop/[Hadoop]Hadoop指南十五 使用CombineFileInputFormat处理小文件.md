Hadoop内置提供了一个 `CombineFileInputFormat` 类来专门处理小文件，其核心思想是：根据一定的规则，将HDFS上多个小文件合并到一个`InputSplit`中，然后会启用一个Map来处理这里面的文件，以此减少MR整体作业的运行时间。
`CombineFileInputFormat`类继承自`FileInputFormat`，主要重写了`List getSplits(JobContext job)`方法；这个方法会根据数据的分布，`mapreduce.input.fileinputformat.split.minsize.per.node`、`mapreduce.input.fileinputformat.split.minsize.per.rack`以及`mapreduce.input.fileinputformat.split.maxsize`参数的设置来合并小文件，并生成List。其中`mapreduce.input.fileinputformat.split.maxsize`参数至关重要：
- 如果用户没有设置这个参数（默认就是没设置），那么同一个机架上的所有小文件将组成一个InputSplit，最终由一个Map Task来处理；
- 如果用户设置了这个参数，那么同一个节点（node）上的文件将会组成一个InputSplit。
同一个 InputSplit 包含了多个HDFS块文件，这些信息存储在 CombineFileSplit 类中，它主要包含以下信息：

同一个 InputSplit 包含了多个HDFS块文件，这些信息存储在 `CombineFileSplit` 类中，它主要包含以下信息：
```java
private Path[] paths;
private long[] startoffset;
private long[] lengths;
private String[] locations;
private long totLength;
```
从上面的定义可以看出，`CombineFileSplit`类包含了每个块文件的路径、起始偏移量、相对于原始偏移量的大小以及这个文件的存储节点，因为一个`CombineFileSplit`包含了多个小文件，所以需要使用数组来存储这些信息。

`CombineFileInputFormat`是抽象类，如果我们要使用它，需要实现`createRecordReader`方法，告诉MR程序如何读取组合的`InputSplit`。内置实现了两种用于解析组合InputSplit的类：`org.apache.hadoop.mapreduce.lib.input.CombineTextInputFormat` 和 `org.apache.hadoop.mapreduce.lib.input.CombineSequenceFileInputFormat`，我们可以把这两个类理解是 `TextInputFormat` 和 `SequenceFileInputFormat`。为了简便，这里主要来介绍CombineTextInputFormat。

在 `CombineTextInputFormat` 中创建了 `org.apache.hadoop.mapreduce.lib.input.CombineFileRecordReader`，具体如何解析`CombineFileSplit`中的文件主要在`CombineFileRecordReader`中实现。

`CombineFileRecordReader`类中其实封装了`TextInputFormat的RecordReader`，并对`CombineFileSplit`中的多个文件循环遍历并读取其中的内容，初始化每个文件的`RecordReader`主要在`initNextRecordReader`里面实现；每次初始化新文件的`RecordReader`都会设置`mapreduce.map.input.file`、`mapreduce.map.input.length`以及`mapreduce.map.input.start`参数，这样我们可以在Map程序里面获取到当前正在处理哪个文件。

```java
package com.sjf.open.example;


import java.io.IOException;

import com.sjf.open.utils.ConfigUtil;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.io.compress.CompressionCodec;
import org.apache.hadoop.io.compress.GzipCodec;
import org.apache.hadoop.mapred.JobPriority;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.MRJobConfig;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.lib.input.CombineTextInputFormat;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

/**
 * Combine Input Example
 *
 * Created by xiaosi on 16-11-7.
 */
public class CombineInputExample extends Configured implements Tool {

    public static void main(String[] args) throws Exception {
        int status = ToolRunner.run(new CombineInputExample(), args);
        System.exit(status);
    }

    private static class CombineInputExampleMapper extends Mapper<LongWritable, Text, Text, Text> {

        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

            Configuration configuration = context.getConfiguration();
            Text newKey = new Text(configuration.get(MRJobConfig.MAP_INPUT_FILE));
            context.write(newKey, value);
        }
    }

    @Override
    public int run(String[] args) throws Exception {
        if (args.length != 2) {
            System.err.println("./run <input> <output>");
            System.exit(1);
        }

        String inputPath = args[0];
        String outputPath = args[1];

        Configuration conf = this.getConf();

        ConfigUtil.checkAndDel(outputPath, conf);
        conf.set("mapreduce.input.fileinputformat.split.maxsize", "33554432");
        conf.set("mapreduce.job.queuename", "wirelessdev");
        conf.set("mapreduce.map.memory.mb", "1024");
        conf.set("mapreduce.reduce.memory.mb", "8192");
        conf.set("mapreduce.job.priority", JobPriority.VERY_HIGH.name());
        conf.setBoolean("mapreduce.output.fileoutputformat.compress", true);
        conf.setClass("mapreduce.output.fileoutputformat.compress.codec", GzipCodec.class, CompressionCodec.class);

        Job job = Job.getInstance(conf);
        job.setJobName("combine_input_example");
        job.setJarByClass(CombineInputExample.class);

        // map
        job.setInputFormatClass(CombineTextInputFormat.class);
        job.setMapperClass(CombineInputExampleMapper.class);
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(Text.class);

        // 输入输出路径
        FileInputFormat.addInputPath(job, new Path(inputPath));
        FileOutputFormat.setOutputPath(job, new Path(outputPath));

        boolean success = job.waitForCompletion(true);
        return success ? 0 : 1;
    }

}

```
会mapreduce.input.fileinputformat.split.maxsize参数的设置，大家可以不设置这个参数并且和设置这个参数运行情况对比，观察Map Task的个数变化。

https://www.iteblog.com/archives/2139.html
http://www.idryman.org/blog/2013/09/22/process-small-files-on-hadoop-using-combinefileinputformat-1/
