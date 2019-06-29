---
title: "使用pandas分析nginx日志：一，解析"
date: 2019-06-28T16:07:59+08:00
draft: false


tags:
- python
- pandas
- 正则表达式

series:
- 使用pandas分析nginx日志
---




拿着榔头看什么都像钉子，拿着pandas看什么都像表格。

今天我打算向nginx日志下黑手。

虽然nginx日志看起来乱糟糟，但其实它不只像像而已，它根本就是表格，一个典型的时序多字段表格。最适合拿来做各种纬度的柱状图折线图大饼图小点图。

由于涉及到的技术点比较多，我们会以一个系列文章的形式展开这段探索。

首先，我们要介绍的是，如何解析。



## 解析

pandas对于csv文件有着比较好的支持。我们最方便的目标就是`pandas.read_csv()`函数，它接受一个csv格式的文本表格。

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
124.236.234.124 - - [25/Jun/2019:06:57:43 +0800] "GET /lib/photoswipe/photoswipe.min.css HTTP/2.0" 200 771 "https://blog.flyingghost.tech/post/setting-content-type-to-multipart-using-requests/" "Mozilla/5.0 (iPhone; CPU iPhone OS 11_4_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15G77 QQ/8.0.6.432 V1_IPH_SQ_8.0.6_1_APP_A Pixel/750 Core/WKWebView Device/Apple(iPhone 7) NetType/WIFI QBWebViewType/1 WKType/1" "-" "-" "0.000" "313817917da62341e884764f6a2411c5"
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
    names=['ip', 'user', 'time', 'request', 'status', 'size', 
           'referer', 'ua', 'forwarded_for', 'upstream-response-time',
          'request_time', 'request_id'],
)
```

解析成功！

下一篇文章我们将会对数据进行清洗和初步处理。