

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
