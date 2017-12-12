
就如上一篇文章介绍的那样，如果输入文件是压缩文件，当MapReduce程序读取压缩文件时，根据文件名的后缀来选择codes，输入文件自动解压缩（我们不需要指定压缩文件是哪一种压缩格式）。

下面我们列出了一些代码，为Hadoop中常用的压缩格式设置输出压缩

## 1. Gzip

对于最终输出，我们可以使用FileOutputFormat上的静态方便方法来设置属性：

```
FileOutputFormat.setCompressOutput(job, true);
FileOutputFormat.setOutputCompressorClass(job, GzipCodec,class);
对于Map输出：

Configuration conf = new Configuration();
conf.setBoolean("mapred.compress.map.output",true);
conf.setClass("mapred.map.output.compression.codec", GzipCodec.class, CompressionCodec.class);
Job job=new Job(conf);
```

## 2. LZO

对于最终输出：

```
FileOutputFormat.setCompressOutput(conf, true);
FileOutputFormat.setOutputCompressorClass(conf, LzoCodec.class);
```
另外，为了使LZO可分割，我们需要生成一个LZO索引文件。

## 3. Snappy

对于最终输出：

```
conf.setOutputFormat(SequenceFileOutputFormat.class); 
SequenceFileOutputFormat.setOutputCompressionType(conf, CompressionType.BLOCK);
SequenceFileOutputFormat.setCompressOutput(conf, true);
conf.set("mapred.output.compression.codec","org.apache.hadoop.io.compress.SnappyCodec");
```
对于Map输出：

```
Configuration conf = new Configuration(); 
conf.setBoolean("mapred.compress.map.output", true);
conf.set("mapred.map.output.compression.codec","org.apache.hadoop.io.compress.SnappyCodec");
```


原文：http://comphadoop.weebly.com/how-to-use-compression.html
