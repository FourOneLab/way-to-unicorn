# YAML

[YAML](https://yaml.org/) 是一个可读性高，用来表达资料序列化的格式。YAML参考了其他多种语言，包括：C语言、Python、Perl，并从XML、电子邮件的数据格式（RFC 2822）中获得灵感。

YAML是"YAML Ain't a Markup Language"（YAML不是一种标记语言）的递归缩写。在开发的这种语言时，YAML 的意思其实是："Yet Another Markup Language"（仍是一种标记语言），但为了强调这种语言**以数据做为中心**，而不是以标记语言为重点，而用反向缩略语重命名。

yaml转换网站点[这里](https://nodeca.github.io/js-yaml/)。

来一段直观的对比吧：

```xml
<Servers>
    <Server>
        <name>Server1</name>
        <owner>John</owner>
        <created>123456</created>
        <status>active</status>
    </Server>
</Servers>
```

```json
{
  "Servers": {
    "Server": {
      "name": "Server1",
      "owner": "John",
      "created": "123456",
      "status": "active"
    }
  },
}
```

```yaml
# 一般用两个空格吧，一个空格看起来有点不清晰
Servers: 
  Server: 
   name: Server1
    owner: John
    created: 123456
    status: active
```

## 语法特点

- 大小写敏感
- 通过缩进表示层级关系
- 禁止使用tab缩进，只能使用空格键
- 缩进的空格数目不重要，只要相同层级左对齐
- 使用#表示注释

## 数据类型

### 集合

```yaml
# 数组
- Mark McGwire
- Sammy Sosa
- Ken Griffey

# Map(键值对)
hr:  65    # Home runs
avg: 0.278 # Batting average
rbi: 147   # Runs Batted In

---
# 上面的组合形式

# Map中的Value是一个数组
american:
  - Boston Red Sox
  - Detroit Tigers
  - New York Yankees
national:
  - New York Mets
  - Chicago Cubs
  - Atlanta Braves

# 数组的元素是一个结构体，
# 结构体中每一个成员以Map的形式保存
-
  name: Mark McGwire
  hr:   65
  avg:  0.278
-
  name: Sammy Sosa
  hr:   63
  avg:  0.288

---
# 变体形式，感觉不常用

# 变体1
- [name        , hr, avg  ]
- [Mark McGwire, 65, 0.278]
- [Sammy Sosa  , 63, 0.288]


# 变体2
Mark McGwire: {hr: 65, avg: 0.278}
Sammy Sosa: {
    hr: 63,
    avg: 0.288
  }
```

### 文档结构

YAML使用三个连续的短横（---）来分割文档的内容。

- （---）：表示文档的开头
- （...)：表示文档的结束

```yaml
# Ranking of 1998 home runs
---
- Mark McGwire
- Sammy Sosa
- Ken Griffey

# Team ranking
---
- Chicago Cubs
- St Louis Cardinals

---
time: 20:03:20
player: Sammy Sosa
action: strike (miss)
...
---
time: 20:03:47
player: Sammy Sosa
action: grand slam
...
```

## 语法

### 引号的区别

- 单引号('')：特殊字符作为普通字符处理
- 双引号("")：特殊字符作为本身想表示的意思

```yaml
# 单引号
name: 'Hi,\nTom'

# 双引号
name: "Hi,\nTom"
```

### 内置类型列表

YAML运行使用感叹号(!)强制转换数据类型：

- 单叹号(!)：自定义类型
- 双叹号(!!)：内置类型

|内置类型|说明|
|---|---|
|!!int|整数类型|
|!!float|浮点数类型|
|!!bool|布尔类型|
|!!str|字符串类型|
|!!null|空值|
|!!set|集合|
|!!seq|列表|
|!!map|键值对|
|!!binary|二进制类型|
|!!timestamp|时间戳类型|
|!!omap/!!pairs|键值列表|

### 纯量

纯量是最基本的不可在分割的值。

```yaml
# 字符串
# 不使用引号
name: Tom

# 使用单引号
name: 'Tom'

# 使用双引号
name: "Tom"

---
# 布尔值
debug: true
debug: false

---
# 数字
12       # 十进制整数
014      # 八进制整数
0xC      ＃十六进制整数
13.4     ＃浮点数
1.2e+34  ＃指数
.inf     ＃无穷大

---
# Null 空值
date: ~
date: null

---
# 时间戳
# 使用iso-8601标准表示日期
date: 2018-01-01t16:59:43.10-05:00
```

### 特殊类型

```yaml
# 文件块
# 注意“|”与文本之间须另起一行
# 使用|标注的文本内容缩进表示的块，可以保留块中已有的回车换行
value: |
  hello
  world!

# 输出结果
# hello 换行 world！

# +表示保留文字块末尾的换行
# -表示删除字符串末尾的换行
value: |
hello

value: |-
hello

value: |+
hello

# 输出结果
# hello\n hello hello\n\n

# 注意“>”与文本之间的空格
# 使用>标注的文本内容缩进表示的块，将块中回车替换为空格最终连接成一行
value: > hello
world!

# 输出结果
# hello 空格 world！


---
# 锚点和引用，kubernetes object 中 label 常用到
# 复制代码注意*引用部分不能追加内容
# 使用&定义数据锚点，即要复制的数据
# 使用*引用锚点数据，即数据的复制目的地
name: &a yaml
book: *a
books:
   - java
   - *a
   - python

# 输出结果
book： yaml
books：[java, yaml, python]

```
