---
title: "使用pandas分析nginx日志"
date: 2019-06-28T16:07:59+08:00
draft: true

tags:
- python
- pandas
- 正则表达式
---




拿着榔头看什么都像钉子，拿着pandas看什么都像表格。

今天我打算向nginx日志下黑手。

虽然nginx日志看起来乱糟糟，但其实它不只像像而已，它根本就是表格，一个典型的时序多字段表格。最适合拿来做各种纬度的柱状图折线图大饼图小点图。

我们将会以jupyter notebook为开发环境，以pandas为工具，以一个真实站点的nginx日志为目标，开展我们的数据探索旅程。



## 解析

分析的第一步，当然是解析日志文件，那个有着严格字段格式但看起来有点凌乱的文本。

pandas对于纯文本文件读取有着比较好的支持，我们最方便的手段就是`pandas.read_csv()`函数，它接受一个csv格式的文本表格。

不幸的是，我们的nginx日志并不是标准csv格式。

我的nginx日志是这样定义的：

```nginx
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      '"$upstream_response_time" "$request_time" "$request_id"';
```

基本上是nginx的默认值，稍微扩展增加了几个字段。

具体的日志行长这样：

```
203.208.60.120 - - [03/Jul/2019:23:59:49 +0800] "GET /case-library/205 HTTP/1.1" 200 7110 "-" "Mozilla/5.0 (Linux; Android 6.0.1; Nexus 5X Build/MMB29P) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2272.96 Mobile Safari/537.36 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" "-" "0.077" "0.077" "c3d263d45fa2066e9097c582217bf60c"
```



整体上来说，各字段由空格分割开来。但某些字段内含空格，所以用`""`（request/ua）或者`[]`（time）括起来。

解析思路有两种：

1. 预处理：使用独立脚本进行数据预处理，把nginx日志转化为csv。

   这个思路简单，但额外需要一道流程，而且会产生中间文件。

2. 热处理：读入nginx日志的每一行，解析并处理为字段，作为一行追加进pandas表格。

   这个思路不产生中间文件，除此之外和1是一致的。额外的缺点是`DataFrame`的追加行操作并不友善。

3. Geek：利用`read_csv()`函数的`sep`参数可接受正则这一优势，使用正则表达式智能的处理多样复杂分隔符，一次读入。我们采取这种方式。

## 分隔符

`sep`本身支持正则表达式，那么怎么样的正则会成为满足我们需求的分隔符呢？

我们将使用 https://regex101.com/ 来分析和撰写正则表达式。

以及，选取以下几行日志作为测试用例：

```
192.0.102.40 - - [27/Jun/2019:08:51:33 +0800] "HEAD / HTTP/1.1" 200 0 "-" "jetmon/1.0 (Jetpack Site Uptime Monitor by WordPress.com)" "-" "-" "0.000" "58de48d633a02be9cd40ff05ebb256ee"
124.236.234.124 - - [25/Jun/2019:06:57:43 +0800] "GET /lib/photoswipe/photoswipe.min.css HTTP/2.0" 200 771 "https://blog.flyingghost.tech/post/setting-content-type-to-multipart-using-requests/" "Mozilla/5.0 (iPhone; CPU iPhone OS 11_4_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15G77 QQ/8.0.6.432 V1_IPH_SQ_8.0.6_1_APP_A Pixel/750 Core/WKWebView Device/Apple(iPhone 7) NetType/WIFI QBWebViewType/1 WKType/1" "-" "-" "0.000" "313817917da62341e884764f6a2411c5"
120.132.3.65 - - [24/Jun/2019:16:46:50 +0800] "\x00\x9C\x00\x01\x1A+<M\x00\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00\x03\x00\x00\x00\x03\xFF\xFF\x00\x01local\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00cananian\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00" 400 150 "-" "-" "-" "-" "0.020" "00b802296ef0a4892ceb66ae26a81845"
176.58.108.6 - - [24/Jun/2019:17:06:49 +0800] "GET /api/v1/pod HTTP/1.1" 404 178 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36" "-" "-" "0.000" "29c93fafbf0a3431dccdbb7ff0552f6a"
139.162.113.204 - - [24/Jun/2019:21:22:46 +0800] "GET / HTTP/1.1" 200 13084 "-" "HTTP Banner Detection (https://security.ipip.net)" "-" "-" "0.000" "bc58016188e104d817ac998255a569d6"
```

先上平民版正则：单空格 `/ /` 看看效果。

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190627221509.png)

可以说很糟糕。双引号内带空格的字段都被当成分隔符了。

所以我们的第一个目标：只有真正处于边界位置的空格，才是分隔符。

正则表达式里有一个概念叫零宽断言，顾名思义，它们和正常的匹配不一样，它们不消费字符，匹配的宽度为0，只匹配一个位置。

例如：

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190628153649.png)

`(?=ing)`断言匹配一个位置，这个位置宽度为零，后面有一个ing跟随。整个表达式匹配的是`一串单词字符+一个后面是ing的间隙位置`。



回到我们的问题：我们期望的`sep`不是所有空格，而是“直到行末为止后面跟着总共偶数个`"`的那些空格”。

于是正则可以修改为`/ (?=([^"]*"[^"]*")*[^"]*$)/`：

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190628163836.png)

好很多了，65个match。其中使用了零宽正预测先行断言`(?=exp)`，断言自身出现位置的后面能匹配表达式。不要在意这奇葩名字，“零宽”表示“匹配一个位置也就是宽度为零”，“正预测”表示“要符合条件”，“先行”表示“要往后匹配”，嫌啰嗦也不用记，理解会用就行了。



但最后一个65号match的空格后面，还跟了个绿色的区域，这是什么？

正则里每一对括号，都会生成一个捕获组，经常用来获取匹配字符串中最有意义最关注的那一部分。但我在这里有两个误区：

1. 所有括号都会生成捕获组，包括零宽断言里的括号。
2. 只要匹配成功，捕获组就会被保留，它不一定非得是匹配字符串的子串。例如本例。

以上两个误区还得感谢群里大神@大灰狼的解惑。



虽然对于匹配来说并没有太大影响，但我们依然可以使用`(?:exp)`语法来明确只匹配不产生捕获。

正则：`/ (?=(?:[^"]*"[^"]*")*[^"]*$)/`

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190628164926.png)



解决了""内的空格，还有日期字段[]内的空格也需要去除。

[]内的空格特点是：空格后面在遇到`[`之前会遭遇一个`]`字符。

依然可以使用零宽断言。这次我们要用的是零宽负预测先行断言`(?!exp)`，零宽、先行不变，“负预测”表示“不能匹配exp”，原英文其实是“zero-width negative lookahead assertion”，negative表否定，翻译成“负预测”也是够了。

正则：`/ (?=(?:[^"]*"[^"]*")*[^"]*$)(?![^\[]*\])/`

![](https://fg-public-1252239724.file.myqcloud.com/blog/image-20190628172357434.png)



检察一下，非常完美。



这也是一般撰写正则表达式的常用思路：

0. 准备好尽可能多样化的用例。
1. 分析匹配文本规律。
2. 只匹配真正需要匹配的文本，用断言来实现前向/后向断言逻辑。
3. 从宽泛到精准的逐步优化匹配。
4. 重复1-3步，每一次迭代做充分测试。



最后，我们使用这个智能分隔符把整个log读入到DataFrame中。

```python
df = pd.read_csv(
    'blog.access.log',
    sep=' (?=(?:[^"]*"[^"]*")*[^"]*$)(?![^\[]*\])',
    engine='python',
    header=None,
    index_col=False,
    names=['ip', '-', 'user', 'time', 'request', 'status', 'size', 
           'referer', 'ua', 'forwarded_for', 'upstream-response-time',
          'request_time', 'request_id'],
)
```

解析成功！



接下来我们将会对数据进行清洗和初步处理，把它做的更方便后续统计一些。

## 数据观察

先观察一下当前数据：

```python
import pandas as pd

df = pd.read_csv(
    'blog.access.log',
    sep=' (?=(?:[^"]*"[^"]*")*[^"]*$)(?![^\[]*\])',
    engine='python',
    header=None,
    index_col=False,
    names=['ip', '-', 'user', 'time', 'request', 'status', 'size', 
           'referer', 'ua', 'forwarded_for', 'upstream_response_time',
          'request_time', 'request_id'],
)
df.head()
df.info()
```

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190629234406.png)

![](https://fg-public-1252239724.file.myqcloud.com/blog/20190629234912.png)



问题很多啊！

- 有空列。这是由于原始nginx log配置里就有一列是单独的`-`号。
- time列有额外的`[]`字符，并且类型也不是datetime。
- 好几个列有`""`号。
- status列其实应为字符串，因为它并不作为数字更不能进行任何运算。
- referer列存在"-"值，这个值在语义来说其实应该是空值，也就是请求根本没带referer头。
- forwarded/upstream/request_id等几列没什么统计意义。
- 其实upstream非常重要，它直接反映了应用服务器处理请求的耗时。可惜我的博客没有upstream，hugo生成静态文件直接由nginx伺服。

## 解析预处理

针对以上问题，我们需要细化`read_csv()`的几个参数。

`usecols`，指定DataFrame将要使用的列，可传入列索引列表，或者列名的列表，列名可以来自于csv文件的表头或者`names`参数。

`na_values`，指定将会被解析为`NaN`值的字符串。

`converters`，指定解析某些列时使用的转换函数。

于是带了一堆预处理参数的`read_csv()`如下：

```python
from datetime import datetime

df = pd.read_csv(
    'blog.access.log',
    sep=' (?=(?:[^"]*"[^"]*")*[^"]*$)(?![^\[]*\])',
    engine='python',
    header=None,
    index_col=False,
    names=['ip', '-', 'user', 'time', 'request', 'status', 'size', 
           'referer', 'ua', 'forwarded_for', 'upstream_response_time',
           'request_time', 'request_id'],
    usecols=['ip', 'time', 'request', 'status', 'size', 'referer', 'ua', 'request_time'],
    na_values='-',
    converters={
        'time': lambda x: datetime.strptime(x, '[%d/%b/%Y:%H:%M:%S %z]'),
        'request': lambda x: x[1:-1],
        'status': str,
        'size': int,
        'referer': lambda x: x[1:-1],
        'ua': lambda x: x[1:-1],
        'request_time': lambda x: float(x[1:-1]),
    }
)
```

![image-20190630003119317](/Users/jacky/Library/Application%20Support/typora-user-images/image-20190630003119317.png)

## 复合数据拆分

上图的数据好看多了。但还有一个问题：

request列其实是复合信息，包含了请求方法、路径、HTTP协议版本等信息。

对于请求分析来说，这几个字段尤其是路径是必须独立统计的。所以有必要把他们拆开。

pandas中有一个方法`pandas.Series.str.extract()`可以把`Series`（一般也就是`DataFrame`的一个列）的每个值按照正则表达式拆成多部分，生成一张表格，然后可以把新的DataFrame合并回原始表格中。

```python
request_df = df.request.str.extract(r'(?P<method>\w+) (?P<url>.*) (?P<schema>.*)$', expand=True)
df = df.join(request_df)
```







