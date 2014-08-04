# Solr in Action 3.1

标签（空格分隔）： solr_in_action 翻译 chapter_3

[TOC]

---

这章内容涉及
> * Solr与传统数据库技术的差别
> * Solr内部索引的基本结构
> * Solr是如何实现使用字段、词组、模糊匹配来实现复杂查询的
> * Solr如何计算相关度score
> * 相关结果返回与所有可能结果返回的权衡
> * 如何将你的内容组装成模式化的文档
> * 扩展Solr以支持亿级文档查询

既然已经装好了Solr，下面有必要探索下搜索引擎是如何工作的。这章目的就是让你能理解内部机制，最大化得利用Solr。

若你在搜索以及信息检索领域已经有了坚实的基础，你可以跳过本章。若没有基础，推荐阅读。

本章的只是适用于大多数的搜索引擎，但还是会着重于Sorl中的相应实现。读完本章，你会对Solr的内部机制有扎实的理解，知道如何使用Solr完成复杂的布尔以及模糊匹配，理解Solr的相关度缺省算法，以及为何Solr在多台服务器构架下还能迅速处理百万级文档。

我们从Solr里的一些核心概念入手，包括搜索索引如何工作，搜索引擎如何匹配查询和文档。看看Solr是如何的高效，使得文本搜索这个难题成为历史。

---

## 3.1 搜索、匹配、寻找内容
已经有了很多处理数据存储于获取的系统：关系型数据库、key-value存储、处理硬盘上数据的map-reduce引擎、图形化数据库等等。搜索引擎，特别是Solr，为我们解决了在大量的非结构化文本中找到最相关的结果这一类查询问题。

本节描述了现代搜索引擎的一些核心特性，包括搜索'文档'的含义、Solr的快速文本匹配中的核心技术——反向代理的大致、反向搜索索引可以支持任意复杂的词、词、部分匹配的大致原因。

---
### 3.1.1 文档是什么

在第二章中我们向Solr提交了一些文档，之后通过Solr，我们做了一些示例查询，当然这不是我们第一次提到文档这个概念。然而，深刻理解Solr可用的查询文件的类型以及文件信息的组织结构十分重要。

Solr是文档存储与获取引擎，向Solr提交的每个数据都是*文档*。文档可以是一篇新闻报道、履历表、简历，再极端点它甚至可能就是一本书。

每个文档那个都有一个或多个字段，每个字段都被模式化为一个特性的字段类型：`string`，`tokenized text`，`Boolean`，`data/time`，`lat/long`等等。字段类型数是不确定的，因为确定一个字段的类型需要大于等于0步的分析，每一步的分析都会改变Solr处理和映射字段数据的方式。每个字段的字段类型都已经在Solr的schema配置中定义好了（第5章将讨论），Solr以此得知要如何处理它接受到的文本。清单3.1展示了一个示例文档那个，以及其定义的每个字段的值。

！[清单 3.1]()

使用Solr处理请求时，我们可以按多个字段值搜索（可以是上面这个文档中根本不存在的字段），Solr会返回文档中拥有该字段且字段值匹配查询的文档。

注意Solr虽然支持灵活的schema配置，但绝对不支持"无-schema"。所有的字段必须申明其字段类型，所有的字段的字段名（动态字段名）都必须在Solr里的schema.xml中定义好。这些在第5章中我们会详细介绍。这并不是说每个文档都必须要包含这些字段，而是所有可能在文档中出现的字段以及需要被处理的字段都必须在schema.xml中配置其字段名以及类型。虽然当solr第一次碰到未声明的字段名时，它会自动得检测字段的内容之后将它自己的判断加入到schema.xml中去。但当该未知字段的输入值很模糊，Solr很可能判断错误，所以最好还是自己去定义那些字段。

文档其实也就是对应着schema中定义的一些定义好的字段类型的一些映射集合。文档中的每个字段都有按照其字段类型分析过的内容，那些分析结果都被存在搜索索引中，用来处理之后的查询请求。Solr返回的最原始的查询请求结果就是是一堆包含已一个或多个字段的文档集。

---

### 3.1.2 基本查询问题

深入之前，首先搞清搜索引擎到底是解决什么问题的？

假设你被布置了一个任务，要完成一个搜索功能实现用户搜索书籍。你的原型图可能和图3.1相似。

**图3.1 示例搜索接口，经常在网页上看到的搜索框，展示了用户提交搜索请求的方式**
![图 3.1]()

假定用户想要搜索一本关于购房的书，对`buying a home`进行了搜索，你提供的结果返回可能如表3.1所示。

**表3.1 与`buying a home`相关的书籍**
|  可能相关的书籍 |
| ---- |
| The Beginner’s Guide to Buying a House |
| How to Buy Your First House|
|Purchasing a Home|
|Becoming a New Home Owner|
|Buying a New Home|
|Decorating Your Home|

表3.2中的书籍你大概就不想返回给用户了
| 不相关的书籍 |
| ---- |
| A Fun Guide to Cooking |
| How to Raise a Child |
| Buying a New Car |

很简单的一个做法即传统的数据库查询语句：

    SELECT * FROM Books WHERE Name = 'buying a new home';

弊端就是上述可能相关的结果中没有书名与`buying a new home`匹配，则你返回结果为空。

一个折衷的方法就是匹配用户查询词组的每一个词，即

    SELECT * FROM Books WHERE Name LIKE '%buying%' AND Name LIKE '%a%' AND Name LIKE '%home%';
    
这种查询用不上数据库的索引，不仅对于数据库来说特别的expensive，而且返回的结果将会是对单个词的所有匹配，如下表
**表3.3 Like模糊查询结果返回**
|匹配的书籍| 不匹配的书籍|
|---|---|
|Buying a New Home|The Beginner’s Guide to **Buying a** House|
||How to Buy Your First House|
||Purchasing **a Home**|
||Becoming **a** New **Home** Owner|
||**A** Fun Guide to Cooking|
||How to Raise **a**Child|
||**Buying a** New Car|
||Decorating Your **Home**|

或许你觉得要文档匹配所有的字段约束性太强了，现在变成OR匹配，放宽约束：

    SELECT * FROM Books WHERE Name LIKE '%buying%' OR Name LIKE '%a%' OR Name LIKE '%home%';

这次的结果如表3.4。这次返回的结果太多了，由于只需要单个匹配，书籍名只包含`a`的都有返回。

**表3.4 Like只需要匹配单个词的返回结果**
|匹配的书籍| 不匹配的书籍|
|---|---|
|**A** Fun Guide to Cooking|How to Buy Your First House|
|Decor **a** ting Your **Home**||
|How to R **a** ise **a** Child||
|**Buying a** New Car||
|**Buying a** New **Home**||
|The Beginner’s Guide to **Buying a** House||
|Purch **a** sing **a** Home||
|Becoming **a** New **Home** owner||

第一种查询中很多相关的书籍都没有返回，第二种查询（匹配单个或者全部匹配）返回了更多的书籍，但很多不想关的书籍也被返回了。

这些例子显示了这种方法实现这个功能的几个弊端：
> * 只能进行子字符串的匹配不能辨别词之间的差别
> * 不能解释语义差别，比如`buying`和`buy`
> * 不区分同义词，如`buying`、`purchasing`以及`home`、`house`
> * 不重要的词`a`极大地影响了查询结果
> * 结果返回木有相关度区分

由于没用用到索引，需要遍历每本书籍的题目，这种查询时间会随着数据种类以及用户查询数目的增长而增长。

像Solr这样的搜索引擎就是为此而生，Solr可以基于内容对文本进行分析，进行语义差别辨认、同义词处理等，同时还会处理掉不重要的分割词如`a`、`the`、`of`等，并对相关度进行评分排序。Solr的这些实现都是基于索引的，通过索引Solr将内容映射到文档，而不是像传统数据库查询那样文档映射内容。反向索引即搜索引擎的关键。

### 3.1.3 反向索引

Solr使用Lucene的反向索引实现高效查询，同时还有一些传统挂的bells 和 whistles技术。这本书不会过于深入Lucene的数据结构，但是在高层级上理解Lucene的索引也很重要，若想深入理解，推荐Lucene in Action。

回顾之前的书籍搜索示例，我们大概可以想我们的索引映射类似表3.5
**表3.5 多个文件文本到反向索引的映射，右边的表格显示了每一项的搜索索引**
|原始文件| | Lucene索引文件|||
|---|---|---|---|---|
|Doc #|Content field |Term| Doc #| (Continued)...|
|123 |A Fun Guide to| a becoming |1,3,4,5,6,7,8 |... guide ... 1,6|
|456| Cooking |beginner's |8 6 9 4,5,6 4 |home house  2,5,7,8| 
|789| Decorating Your |buy buying |3 1 2 9 1 ... | how new  6,9 3,9| 
||Home How to|car child ||owner 4,5,8 8 |
||Raise a Child |cooking ||purchasing 7 3 6|
||Buying a New Car |decorating ||raise the to 1,6,9 |
||Buying a New |first fun ... ||your 2,9 |
||Home The |
||Beginner’s Guide |
||to Buying a House |
||Purchasing a |
||Home Becoming a |
||New Home Owner |
||How to Buy Your |
||First House |

在传统的数据库形式中，文档会包含一个文档ID，此文档ID会映射到一个或多个字段（包含了文档中所有的words/terms）。反向索引是反过来的，将每个words/terms映射到每个它出现过的文档中。观察表3.5，原始文本都被空格键分隔开，在插入索引之前每个字段都变成小写，其他的保持一致。需要注意的是可能还存在其他的文本转换，在文本分析过程中，terms可能被修改、增加或者删除，在第6章中我们会继续讨论。

反向代理还有两个重要知识点：
> * 索引中每个字段都被映射到一个或多个文档
> * 反向索引中的字段按照字段顺序降序排列

这里只是很粗略得看了一遍反向索引，在章节3.1.6中继续研究。

---

### 3.1.4 词，词组，布尔逻辑

我们已经知道在Lucene的索引中，文本内容是什么样，下面介绍如何用反向索引执行查询。本节中，过一遍词、词组、布尔逻辑、模糊查询。在之前的数据查询示例中查询`new house`。

从上一章中得知，在字段内的内容插入索引时都被分成单个的词。既然有了一个请求，那么在查询索引之前，你先要对此次搜索做些选择：
> * 搜索两个词，`new`和`house`，两个词都要存在在文档中
> * 搜索两个词，`new`和`house`，至少有一个在文档中
> * 搜索词组`new house`

根据不同的用户案例，这三种选择都是有效的。多亏了基于Lucene查询的Solr引擎，我们用Boolean 逻辑即可实现这三种查询。

#### **Required Terms**

首先看第一种搜索，把搜索项拆分成单个词，进行且匹配。在Solr中表达这种查询有两种写法：
> * +new+house
> * new AND house

这两种表达在逻辑上等价，事实上第二种最终会被解析成第一种。`+`为一元操作符，表示接在其后的词即为需要在文档中存在的词，`AND`为二元操作符，即其前后的参数都是需要在文档中查询的词。

####**Optional Terms**

Solr 不止提供了AND，还提供了OR。没有显示的操作符时，Solr自动辨认为或操作。即以下等价
> * new house
> * new OR hosue

####**Negated terms**

有时候搜索不仅需要查找存在项，还要查询不存在。语法如下：
> * new hosue -rental
> * new house NOT reantal

这种查询下，只返回不含有rental 且包含new 或者house的文档
> **扩展——Solr缺省操作符**
> 大意就是默认Solr缺省为OR，但你可以怎么怎么样去该成AND什么的

####**Prases**
Solr不仅支持搜索单个词，还支持搜索词组，确保单词按顺序挨在一起出现。
> * `"new home"` OR `"new house"`
> * `"3 bedrooms"`AND `"walk in closet"`AND`"granite coutertops"`

####**Grouped expressions**

Solr支持的最后一个基本的Boolean 构造是对词、词组、查询表达式进行分组。Solr的语法可以支持任意复杂的查询，方法时使用各种父级别的group，示例如下：
> * `New AND (house OR （home NOT improvement NOT depot NOT grown）)`
> * `(+(buy purchasing -renting) + (home houose residence -(+property-bedroom)))`

---

###3.1.5 查找文档集
基本了解了词、词组、Boolean查询后，终于可以深入理解Solr如何使用Lucene所以进行查询。回顾表3.5的内容，表3.6抓取了一部分。

**表3.6 图书标题的反向索引**
|Term|Document|(Continued...)|...|
|---|---|---|---|
|a|1,2,3,4,5,6,7,8|...|...|
|becoming|8|guide|1,6|
|beginner's|6|home|2,5,7,8|
|buy|9|house|6,9|
|buying|4,5,6|how|3,9|
|car|4|new|4,5,8|
|child|3|owner|8|
|...|...|...|...|
若现在用户查询`new home`，Solr是如何进行匹配的呢？

查询`new home`，这个查询其实为两个词的查询（缺省或查询），每个词都会在Lucene索引中查找一遍。
|Term|Document|
|---|---|
|home|2,5,7,8|
|new|4,5,8|
每个词的索引都找到之后，Solr会生产最终的搜索结果。如果查询逻辑为OR，则最终结果就是每个词的搜索结果的并集。
同理可得AND。
图3.5可以验证下你的理解，学过初中数学都懂，pdf图不对，不上图了。

---

###3.1.6 词组查询和词位置

基于Lucene的索引，Solr完成了词查询，但是索引都只包含了单个的词，那如何执行词组的查询？

假设输入从`new house`变成了`"new house"`，情况怎么样？简单来说还是对Lucene的索引单个查询，只不过用到了我们之前没有提到的一个索引属性，即*term position*。`term position`记录了词在文档中的相对位置。表3.7展示了文档（表左侧）携带`term position`如何映射到索引（表右侧）中。

**表3.7 带term position的索引**
pdf的表又跪了，不急看。
从表3.7中可看出，`new AND home`查询会返回集合{5,8}，进一步用`term position`来查看词出现的位置。表3.8展示了第一步集合的信息
**表3.8 带term position的简明表**
|Term|Document|Term position|
|---|---|---|
|home|5|4|
||8|4|
|new|5|3|
||8|3|
在这个例子中，恰好匹配的文件中都是home在第4位，new在第3位。由于两个词在文档位置差1，则Solr可认为他们在源文档中就是一个词组。这就是`ter position`的厉害之处，摆脱源文件即可还原词的位置，进行词组匹配。

`term position`的好处不止这一个，下一节你会看到通过使用`term position`我们可以极大地提高搜索质量。

---

###3.1.7. 模糊匹配

有时候可能你在查询之前自己都不能特别明确搜索的项目，因此Solr还提供几种模糊搜索。比如，人们可能想搜索匹配某个前缀的所有词(*wildcard searching*)，或者有1，2个拼写不同的条目搜索（*fuzzy searching* 或说 *edit distance searching*），或者想在某个差距范围内两两匹配（*proximity searching*）。

本节中将探索以上所有的搜索。

####**Wildcard searching**

假设你想搜索所有以`offic`开头的文档，一个方法时在查询参数里枚举所有可能的类型：
> Query: office OR officer OR official OR officiate OR ...

显然对你和对用户都不合适。

可以用通配符（`*`）进行匹配：
> Query: offi* Matches office, officer, official, and so on 

通配符不仅能出现在末尾，可以出现在中间，比方你想匹配`officer`和`offer`
> Query: off*r Matches offer, officer, officiator, and so on 

通配符（`*`）表示匹配0或者多个字符，想匹配单个字符，用(`?`)
>  Query: off?r Matches offer, but not officer 

这个扩展有点意思。

>**扩展——Leading wildcard**
>虽然统配符很通用，但有时候使用通配符会很耗资源。wildcard search执行时候，term的查询肯定在通配符匹配之前。匹配了term的那些结果集再去检查是否符合通配符。因此，你定义越明确的term，匹配会更快。比方说，`engineer* `查询会比` e*`查询轻松。

>执行通配符开头的查询会十分耗费资源，如果你要执行所有匹配`ing`结束的词，这很可能会降低性能。
>> Query: *ing

>如果你实在是需要执行leading-wildcard匹配，更好的方法是在配置文件中进行配置，增加`ReversedWildcardFilterFactory`到你的字段分析类型链中（第6章）。

>`ReversedWildcardFilterFactory`会在Solr引擎中再存一份反序索引。即：
>  
    Index: caring liking smiling
           #gnirac #gnikil #gnilims

>这种做法需要再存一份索引，会占用更大的存储，也会降低全盘的搜索。若非必须不推荐开始这个功能。

最后对于wildcard search搜索还要注意的是，它只使用于词的匹配，不适用于词组的匹配，即
> 有效: `softwar* eng?neering`
> 无效: `"softwar* eng?neering"`

若你想对词组进行这种查询，那么你需要在索引中存储词组，第6章有介绍。
####**Range searching**

Solr还提供了在范围内的搜索，比如，你希望查找在这6个月内内创建的文档2012.2.2-2012.8.2，可以进行如下搜索
> Query: created:[2012-02-01T00:00.0Z TO 2012-08-02T00:00.0Z]

range查询对于其他数据类型使用如下：
> Query: yearsOld:[18 TO 21] Matches 18, 19, 20, 21 
> Query: title:[boat TO boulder] Matches boat, boil, book, boulder, etc. 
> Query: price:[12.99 TO 14.99] Matches 12.99, 13.000009, 14.99, etc. 

提供封闭范围和不封闭范围
> Query: yearsOld:{18 TO 21} Matches 19 and 20 but not 18 or 21 

两者可以组合
> Query: yearsOld:[18 TO 21} Matches 18, 19, 20, but not 21 

####**Fuzzy/edit-distance searching**

























