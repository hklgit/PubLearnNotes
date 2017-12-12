### 1. Regex对象

我们可以使用`scala.util.matching.Regex`类使用正则表达式．要构造一个Regex对象，使用String类的r方法即可:
```
val numPattern = "[0-9]+".r
```
如果正则表达式包含反斜杠或引号的话，那么最好使用"原始"字符串语法`"""..."""`:
```
val positiveNumPattern = """^[1-9]\d*$"""
```
如果在Java中使用上述正则表达式，则应该使用下面方式(需要进行转义):
```
val positiveNumPattern = "^[1-9]\\d*$"
```
相对于在Java中的使用方式，Scala这种写法可能更易读一些．

### 2. findAllIn

findAllIn方法返回遍历所有匹配项的迭代器．可以在for循环中使用它:
```
val str = "a b 27 c 6 d 1"
val numPattern = "[0-9]+".r
for(matchingStr <- numPattern.findAllIn(str)){
  println(matchingStr)
}
```
或者将迭代器转成数组:
```
val str = "a b 27 c 6 d 1"
val numPattern = "[0-9]+".r
val matches = numPattern.findAllIn(str).toArray
// Array(27,6,1)
```
### 3. findPrefixOf

检查某个字符串的前缀是否能匹配，可以使用findPrefixOf方法:
```
val str = "3 a b 27 c 6 d 1"
val str2 = "a b 27 c 6 d 1"
val numPattern = "[0-9]+".r
val matches = numPattern.findPrefixOf(str)
val matches2 = numPattern.findPrefixOf(str2)
println(matches) // Some(3)
println(matches2) // None
```
### 4. replaceFirstIn replaceAllIn

可以使用如下命令替换第一个匹配项或者替换全部匹配项:
```
val str = "3 a b 27 c 6 d 1"
val numPattern = "[0-9]+".r
val matches = numPattern.replaceFirstIn(str, "*")
val matches2 = numPattern.replaceAllIn(str, "*")
println(matches) // * a b 27 c 6 d 1
println(matches2) // * a b * c * d *
```

### 5. 正则表达式组

分组可以让我们方便的获取正则表达式的子表达式．在你想要提取的子表达式两侧加上圆括号:
```
val str = "3 a"
val numPattern = "([0-9]+) ([a-z]+)".r
val numPattern(num, letter) = str
println(num) // 3
println(letter) // a
```
上述代码将num设置为3，letter设置为a

如果想从多个匹配项中提取分组内容，可以使用如下命令:
```
val str = "3 a b c 4 f"
val numPattern = "([0-9]+) ([a-z]+)".r
for(numPattern(num, letter) <- numPattern.findAllIn(str)){
  println(num + "---"+letter)
}
// 3---a
// 4---f
```
