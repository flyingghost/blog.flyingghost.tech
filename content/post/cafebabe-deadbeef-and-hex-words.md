---
title: "cafebabe、deadbeef 与HEX words"
date: 2019-06-20T21:31:41+08:00

---


# cafebabe、deadbeef 与HEX words

Java的class文件有一个固定的文件头：`CAFEBABE`，正好是一个16进制字符串，用来做二进制文件的头部非常适合，既隐含Java的咖啡文化，还有一点可爱和调皮，可以说是geek界著名的神来之笔了。

还有一个很有名的16进制字符串`deadbeef`，也很酷，经常被用来做ID，程序员们看到都会心领神会。

这些都是HEX words，顾名思义是可以用16进制允许的字符`abcdef`组成的单词。

那么，还有哪些好玩的HEX words呢？干脆翻翻词典把它们都找出来吧。


## 数据源准备

词典可以在开源项目 https://github.com/skywind3000/ECDICT/ 中找到(鸣谢)。65M的csv，77万+个单词，只嫌太多太专业。

打开csv文件，格式是这样的：

```
# 表格列
word,phonetic,definition,translation,pos,collins,oxford,tag,bnc,frq,exchange,detail,audio
# 样本数据
cafe,kɑ:'fei,n. a small restaurant where drinks and snacks are sold,"n. 咖啡馆, 酒店",,2,,cet4 cet6 ky ielts,5986,5837,s:cafes,,
```

每列的作用如下：

| 字段        | 解释                                                       |
| ----------- | ---------------------------------------------------------- |
| word        | 单词名称                                                   |
| phonetic    | 音标，以英语英标为主                                       |
| definition  | 单词释义（英文），每行一个释义                             |
| translation | 单词释义（中文），每行一个释义                             |
| pos         | 词语位置，用 "/" 分割不同位置                              |
| collins     | 柯林斯星级                                                 |
| oxford      | 是否是牛津三千核心词汇                                     |
| tag         | 字符串标签：zk/中考，gk/高考，cet4/四级 等等标签，空格分割 |
| bnc         | 英国国家语料库词频顺序                                     |
| frq         | 当代语料库词频顺序                                         |
| exchange    | 时态复数等变换，使用 "/" 分割不同项目，见后面表格          |
| detail      | json 扩展信息，字典形式保存例句（待添加）                  |
| audio       | 读音音频 url （待添加）                                    |

其中除了单词和释义以外，对我们筛选词可能有帮助的是collins、oxford、tag几列。毕竟原词库太大了。。。

## 数据探索和清洗

先读入整个文件，只要读入所需的列就可以了。


```python
import pandas as pd

df = pd.read_csv('ecdict.txt', usecols=['word', 'translation', 'collins', 'oxford', 'tag'])
```

我们来看一下数据的情况：


```python
df.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 770611 entries, 0 to 770610
    Data columns (total 5 columns):
    word           770607 non-null object
    translation    768739 non-null object
    collins        13633 non-null float64
    oxford         3461 non-null float64
    tag            14942 non-null object
    dtypes: float64(2), object(3)
    memory usage: 29.4+ MB


什么情况？770611行数据，只有770607个word non-null值？还有4个空值是怎么来的？


```python
df[df.word.isna()]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>word</th>
      <th>translation</th>
      <th>collins</th>
      <th>oxford</th>
      <th>tag</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>461828</th>
      <td>NaN</td>
      <td>（用于回答表格问题）不相关, 不适用</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>461830</th>
      <td>NaN</td>
      <td>北美洲, 国立研究院院士, 国立研究院, 国民军\n[医] 钠(11号元素)</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>462678</th>
      <td>NaN</td>
      <td>n. 奶奶（小孩儿语）；印度, 巴基斯坦式的微微发酵的面包</td>
      <td>1.0</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>481491</th>
      <td>NaN</td>
      <td>a. 无效力的, 无效的, 无价值的\nn. 零, 空, 零位\n[计] 空</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>toefl ielts</td>
    </tr>
  </tbody>
</table>
</div>



追溯一下源文件中的这四行，原来word依次是`n/a`、`NA`、`nan`和`null`。pandas很“体贴”的把这几个特殊字符串解析成为NaN了。翻一下pandas文档：

>By default the following values are interpreted as NaN: ‘’, ‘#N/A’, ‘#N/A N/A’, ‘#NA’, ‘-1.#IND’, ‘-1.#QNAN’, ‘-NaN’, ‘-nan’, ‘1.#IND’, ‘1.#QNAN’, ‘N/A’, ‘NA’, ‘NULL’, ‘NaN’, ‘n/a’, ‘nan’, ‘null’.

呵呵。指定参数重新读一遍。


```python
df = pd.read_csv('ecdict.txt', usecols=['word', 'translation', 'collins', 'oxford', 'tag'], keep_default_na=False, na_values='')
```


```python
df.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 770611 entries, 0 to 770610
    Data columns (total 5 columns):
    word           770611 non-null object
    translation    768739 non-null object
    collins        13633 non-null float64
    oxford         3461 non-null float64
    tag            14942 non-null object
    dtypes: float64(2), object(3)
    memory usage: 29.4+ MB



```python
df.word.hasnans
```




    False



很满意。可以开始分析单词了。

## 单词搜索

严格意义上的HEX words只会出现`abcdef`六个字符。这个搜索非常简单，直接用正则搜索word字段就好。


```python
hex_words = df[df.word.str.match('^[a-fA-F]+$')]

hex_words.tail()  # 节省篇幅只输出部分
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>word</th>
      <th>translation</th>
      <th>collins</th>
      <th>oxford</th>
      <th>tag</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>258072</th>
      <td>ffc</td>
      <td>abbr. 柔性扁平电缆（flexible flat cable, 等于软性排线）；外资管理...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>258073</th>
      <td>ffd</td>
      <td>[医][=focal film distance]焦点胶片距离</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>258074</th>
      <td>ffdca</td>
      <td>[医][=Federal Food,Drug and Cosmetic Act]联邦食品、...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>258077</th>
      <td>FFF</td>
      <td>[化] 场流分级法</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>258078</th>
      <td>ffff</td>
      <td>n. 飞行符（梦幻西游）；“土”字的五笔编码</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



大小写都算在内的话有460个单词。既有`add`这种正常单词，又有`FFA`这种非常冷门的缩写。看来数据源太大也不完全是好事。

好在我们可以参考其他几个列。一般来说核心常用词都会被柯林斯、牛津等字典收录，也会纳入cet4/cet6等常见范畴。

那么我们再根据这三个字段过滤一下非NaN的数据。


```python
hex_words_common = hex_words[hex_words.tag.notna() | hex_words.collins.notna() | hex_words.oxford.notna()]

hex_words_common.tail()  # 节省篇幅只输出部分
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>word</th>
      <th>translation</th>
      <th>collins</th>
      <th>oxford</th>
      <th>tag</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>249381</th>
      <td>face</td>
      <td>n. 脸, 面容, 正面, 外观\nvt. 面对, 朝, 正视, 面临\nvi. 朝, 向\...</td>
      <td>5.0</td>
      <td>1.0</td>
      <td>zk gk toefl ielts</td>
    </tr>
    <tr>
      <th>250172</th>
      <td>fad</td>
      <td>n. 时尚\n[化] 黄素腺嘌呤二核苷酸</td>
      <td>1.0</td>
      <td>NaN</td>
      <td>gre</td>
    </tr>
    <tr>
      <th>250187</th>
      <td>fade</td>
      <td>vi. 褪色, 消失, 凋谢\nvt. 使褪色\nn. 淡入, 淡出\na. 平淡的</td>
      <td>3.0</td>
      <td>NaN</td>
      <td>gk cet4 cet6 ky toefl ielts gre</td>
    </tr>
    <tr>
      <th>255514</th>
      <td>fee</td>
      <td>n. 费用, 小费, 封地, 所有权\nvt. 付费给</td>
      <td>4.0</td>
      <td>1.0</td>
      <td>gk cet4 ky ielts</td>
    </tr>
    <tr>
      <th>255560</th>
      <td>feed</td>
      <td>n. 饲料, 一餐, 饲养\nvt. 喂, 饲养, 放牧, 靠...为生\nvi. 吃东西,...</td>
      <td>4.0</td>
      <td>1.0</td>
      <td>zk gk cet4 cet6 ky toefl</td>
    </tr>
  </tbody>
</table>
</div>




```python
hex_words_common.info()
```

    <class 'pandas.core.frame.DataFrame'>
    Int64Index: 41 entries, 607 to 255560
    Data columns (total 5 columns):
    word           41 non-null object
    translation    41 non-null object
    collins        34 non-null float64
    oxford         16 non-null float64
    tag            30 non-null object
    dtypes: float64(2), object(3)
    memory usage: 1.9+ KB


很好，眼熟很多了。虽然量少了点，但确实是精准了一些。把这些词再自行组合，就可以得到很多Hex word。例如cafebabe/deadbeef/badface等随意发挥吧！

## 谐音和相似形

所谓人心不足蛇吞象。兴奋过后，还是觉得意犹未尽意兴阑珊。太少了，总共三四十个，有意义的组合也不够玩啊。

好在我们还有10个数字可以用。

发音方面，`2`可以谐音代替`to`或者`too`，`4`可以谐音代替`for`，`8`可以谐音代替`ate`，这些都是挺常用的单词音节。

外形角度来说，`0`和`o`长相极其相似，`1`和`l`在某些字体中简直一毛一样，这么一来可用字符又扩充了不少。

我们来做一个映射表，并生成对应的正则表达式：


```python
import re

subs = {
    'too': '2',
    'to': '2',
    'o': '0',      # 注意顺序，因为是依次查找并匹配，所以长串在前短串在后
    'for': '4',
    'ate': '8',
    'l': '1',
}
substitution = dict((re.escape(k), v) for k, v in subs.items())
pattern = re.compile("|".join(substitution.keys()))
```

然后准备好单词的替换函数：


```python
repl = lambda word: pattern.sub(lambda m: substitution[re.escape(m.group(0))], word)
```

可以针对全量字典做替换啦！`word`字段经过查找替换后存储在`replaced`字段。


```python
df['replaced'] = df['word'].map(repl)
```

按照规则替换完毕，重新搜索所有符合Hex字符集的单词，当然这次是需要在`replaced`字段上搜索了。


```python
hex_words = df[df.replaced.str.match('^[a-fA-F0-9]+$')]
hex_words_common = hex_words[hex_words.tag.notna() | hex_words.collins.notna() | hex_words.oxford.notna()]
```


```python
hex_words_common.info()
```

    <class 'pandas.core.frame.DataFrame'>
    Int64Index: 170 entries, 607 to 707190
    Data columns (total 6 columns):
    word           170 non-null object
    translation    170 non-null object
    collins        143 non-null float64
    oxford         59 non-null float64
    tag            139 non-null object
    replaced       170 non-null object
    dtypes: float64(2), object(4)
    memory usage: 9.3+ KB


170个词，我们来逐个看看。


```python
hex_words_common.tail()  # 节省篇幅只输出部分
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>word</th>
      <th>translation</th>
      <th>collins</th>
      <th>oxford</th>
      <th>tag</th>
      <th>replaced</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>705961</th>
      <td>toe</td>
      <td>n. 足趾, 趾部, 脚趾\nvt. 以趾踏触, 用脚尖走\nvi. 动脚尖</td>
      <td>2.0</td>
      <td>1.0</td>
      <td>cet4 cet6 ky toefl</td>
      <td>2e</td>
    </tr>
    <tr>
      <th>706003</th>
      <td>toed</td>
      <td>a. 有趾的, 斜着钉进去的</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>toefl</td>
      <td>2ed</td>
    </tr>
    <tr>
      <th>706365</th>
      <td>toll</td>
      <td>n. 通行费, 代价, 钟声\nvt. 征收, 敲钟, 鸣钟, 勾引, 引诱\nvi. 征税...</td>
      <td>2.0</td>
      <td>NaN</td>
      <td>ky toefl ielts gre</td>
      <td>211</td>
    </tr>
    <tr>
      <th>707155</th>
      <td>too</td>
      <td>adv. 也, 非常, 太</td>
      <td>5.0</td>
      <td>1.0</td>
      <td>zk gk</td>
      <td>2</td>
    </tr>
    <tr>
      <th>707190</th>
      <td>tool</td>
      <td>n. 工具, 机床, 傀儡\nvt. 用工具加工\nvi. 使用工具</td>
      <td>3.0</td>
      <td>1.0</td>
      <td>zk gk cet4 cet6 ky</td>
      <td>21</td>
    </tr>
  </tbody>
</table>
</div>



矮油，好多纯数字的“词”（例如`too -> 2`、`tool -> 21`），看起来已经不像单词了，倒像个纯数字。弃之。


```python
hex_words_common = hex_words_common[~hex_words_common.replaced.str.match('^[0-9]+$')]

hex_words_common.tail()  # 节省篇幅只输出部分
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
    
    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>word</th>
      <th>translation</th>
      <th>collins</th>
      <th>oxford</th>
      <th>tag</th>
      <th>replaced</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>705704</th>
      <td>toad</td>
      <td>n. 蟾蜍, 癞蛤蟆, 讨厌的家伙</td>
      <td>1.0</td>
      <td>NaN</td>
      <td>cet6 toefl</td>
      <td>2ad</td>
    </tr>
    <tr>
      <th>705772</th>
      <td>tobacco</td>
      <td>n. 烟草, 香烟\n[医] 烟草</td>
      <td>2.0</td>
      <td>NaN</td>
      <td>gk cet4 cet6 ky toefl</td>
      <td>2bacc0</td>
    </tr>
    <tr>
      <th>705936</th>
      <td>toddle</td>
      <td>vi. 东倒西歪地走, 蹒跚学步, 溜达\nn. 东倒西歪的走路, 蹒跚</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>ielts</td>
      <td>2dd1e</td>
    </tr>
    <tr>
      <th>705961</th>
      <td>toe</td>
      <td>n. 足趾, 趾部, 脚趾\nvt. 以趾踏触, 用脚尖走\nvi. 动脚尖</td>
      <td>2.0</td>
      <td>1.0</td>
      <td>cet4 cet6 ky toefl</td>
      <td>2e</td>
    </tr>
    <tr>
      <th>706003</th>
      <td>toed</td>
      <td>a. 有趾的, 斜着钉进去的</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>toefl</td>
      <td>2ed</td>
    </tr>
  </tbody>
</table>
</div>



还剩161个，比起上一波来可玩性提升3倍！

## 拓展阅读

其实翻翻英文geek界，老前辈们早就把类似的小把戏玩出花来了。wiki上还有专门的[介绍](https://en.wikipedia.org/wiki/Hexspeak)，历史那是相当的悠久，甚至直到现在还有软件/系统在借鉴这个思路。细细读起来不禁脑洞大开啊。

最后有人问我，知道这个能干嘛？喝喝，啥用也没有，图个乐呗。
