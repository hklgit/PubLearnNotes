### 1. 读取行
读取文件，可以使用scala.io.Source对象的`fromFile`方法．如果读取所有行可以使用`getLines`方法:
```
val source = Source.fromFile("/home/xiaosi/exception.txt", "UTF-8")
val lineIterator = source.getLines()
for(line <- lineIterator){
  println(line)
}
source.close()
```
source.getLines返回结果为一个迭代器，可以遍历迭代器逐条处理行．

如果想吧整个文件当做一个字符串处理，可以调用`mkString`方法:
```
val content = source.mkString
```

**备注**

在用完Source对象后，记得调用close方法进行关闭

### 2. 读取字符

读取字符，可以直接把Source对象当做迭代器使用，因为Source类扩展了`Iterator[Char]`:
```
val source = Source.fromFile("/home/xiaosi/exception.txt", "UTF-8")
for(c <- source){
  print(c + " ")
}
```

### 3. 从URL或其他源读取数据

Source对象有读取非文件源的方法:
```
// 从URL中读取数据
val sourceUrl = Source.fromURL("http://xxx", "UTF-8")
// 从字符串中读取数据
val sourceStr = Source.fromString("Hello World!")
// 从标准输入读取数据
val sourceStd = Source.stdin
```

### 4. 读取二进制文件

Scala并没有提供读取二进制文件的方法．但是你可以使用Java类库来完成读取操作:
```
val file = new File(fileName)
val in = new FileInputStream(file)
val bytes = new Array[byte](file.length.toInt)
in.read(bytes)
in.close()
```

### 5. 写入文本文件

Scala并没有内置的对写入文件的支持．但是可以使用`java.io.PrintWriter`来完成:
```
val out = new PrintWriter("/home/xiaosi/exception.txt")
out.println("Hello World")
out.println("Welcome")
out.close()
```

### 6. 访问目录

目前Scala并没有用来访问某个目录中的所有文件，或者递归的遍历所有目录的类，我们只能寻求一些替代方案.

利用如下代码可以实现递归遍历所有的子目录:
```
// 递归遍历目录
def subDirs(dir: File) : Iterator[File] = {
  val children = dir.listFiles().filter(_.isDirectory)
  children.toIterator ++ children.toIterator.flatMap(subDirs _)
}

val file = new File("/home/xiaosi/test")
val iterator = subDirs(file)
for(d <- iterator){
  println(d)
}
```
