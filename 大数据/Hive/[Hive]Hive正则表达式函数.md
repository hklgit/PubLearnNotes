

### 1. regexp

语法:
```
A REGEXP B
```
返回值:
```
string
```
说明:
```
功能与RLIKE相同
```

### 2.regexp_extract

语法:
```
regexp_extract(string subject, string pattern, int index)
```
返回值:
```
string
```
说明：
```
将字符串subject按照pattern正则表达式的规则拆分，返回index指定的字符。
```

### 3. regexp_replace

语法:
```
regexp_replace(string A, string B, string C)
```
返回值:
```
string
```
说明：
```
将字符串A中的符合java正则表达式B的部分替换为C。注意，在有些情况下要使用转义字符,类似oracle中的regexp_replace函数。
```
