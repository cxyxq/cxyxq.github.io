---
title: 倒排索引 Inverted Index
categories: [Elasticsearch, 倒排索引]
tags: [分布式]
author: [xiao_e]
---

## 索引

假设我们要做一道“西红柿炒鸡蛋”的菜，不会做，但是我们有一本烹饪食谱，如果我们从食谱的第一页开始查找，直到找到“西红柿炒鸡蛋”的菜谱， 等找到了估计也不想做这道菜了..... 好在烹饪食谱有菜品和对应菜谱的目录索引，且菜品名称是按照首字母排序的，那我们想找某道菜时就会快很多。

```
Tartar, beef tartar……………………page 67
Tomato chutney……………………page 645
Tomato soup…………………………page 23, 78
Umami burger…………………………page 378
```

搜索引擎中的索引工作方式跟菜谱目录很相似， 搜索引擎创建terms和terms出现的位置(菜品名称和出现的页数)。

## Document Parsing and Tokenization

下面看下什么是`文档解析`和`分词`。

### Document Parsing

以上面的菜谱为例，搜索引擎需要创建菜谱中的`单词 words`或者`term`，和包含这些term的菜谱的引用或映射，这基本上就构成了index。最后把index保存到磁盘或者部分保存到内存以方便检索。

### Full-Text Search Engines

### Tokenization

把一个文本分割为不同的单词或者句子，这个过程称为：`Tokenization`，分割后的单词被称为：`tokens`。 

以下面的例子来解释下：
```
Heat slowly, stir gently while adding twenty-five grams of flour.
```
{: file="文本"}

上面的句子分割为不同的token:
```
[heat] [slowly] [stir] [gently] [while] [adding] [twenty-five] [grams] [of] [flour]
```
{: file="tokens"}

#### Tokenizers

Tokenizer 分词器可以用来分割文本中的单子或者句子，一般识别通过空格和标点符号来分割，但是有些单词处理起来比较棘手，比如* twenty-five* 是分割成2个token：[twenty] 和 [five] 呢，还是一个token [twenty-five] 呢？ 假如你想搜索 25呢？

Tokenizers 可以是很复杂的，它们需要处理如`缩写(ca., kg., dr.)`、`数字(10 or ten, 3x10^9 or 3 billion)`、`连字符 (twenty-five)`、`行尾连字符（一个单词分成两行）`、`词汇连字符(pre-, multi-)`、`缩写 (I'm, it's)`、`复合词(motorboat or Kraftfahrzeug)` 等其他特殊文本。

此外，在一些情况下 Tokenizers 需要有语言感知能力，比如中文分词规则就不太一样。

有许多专门的Tokenizer来解决不同场景下的分词，比如有 `CamelCase tokenizer`, `URL tokenizer`, `path tokenizer` and `N-gram tokenizer`.

看一个path tokenizer的示例
```
C:\WINDOWS\system32\rundll32.exe
```
分词后：
```
[c:] [c:/windows] [c:/windows/system32] [c:/windows/system32/rundll32.exe]
```

### Stop Words

在许多场景下，例如 `a, the, us, who`等这些单词是没有实际意义的，我们可以选择不把它们进行索引，这些单词被称为：`Stop Words`。但是在大多数现代搜索引擎中，不会忽略Stop Words， 因为我们可能会去搜索乐队`The Who`，或者搜索某个短语、完整句子时里面带有`a, the, us, who`这些单词。


### Relevancy

TODO

## 正排索引

| Document              | Terms                                                        |
| --------------------- | ------------------------------------------------------------ |
| Grandma’s tomato soup | peeled, tomatoes, carrot, basil, leaves, water, salt, stir, and, boil, … |
| African tomato soup   | 15, large, tomatoes, baobab, leaves, water, store, in, a, cool, place, … |
| Good ol’ tomato soup  | tomato, garlic, water, salt, 400, gram, chicken, fillet, cook, for, 15, minutes, … |


创建正排索引时很快，一个新文档追加到index并且不需要重建索引。但是查询效率不高，因为搜索引擎需要遍历index中所有记录去查找包含指定term的文档。


## 倒排索引

| Term    | Documents                                                    |
| ------- | ------------------------------------------------------------ |
| baobaob | African tomato soup                                          |
| basil   | Grandma’s tomato soup                                        |
| leaves  | African tomato soup, Grandma’s tomato soup                   |
| salt    | African tomato soup, Good ol’ tomato soup                    |
| tomato  | African tomato soup, Good ol’ tomato soup, Grandma’s tomato soup |
| water   | African tomato soup, Good ol’ tomato soup, Grandma’s tomato soup |
| ...     | ...                                                          |

上面的这种`Term : Documents`的映射方式称为:`倒排索引 inverted index`， 称为倒排是因为它是正排的反向。当使用倒排索引检索term时，term只需要被查找一次即可获取到包含此term的所有docums。

通常正排索引和倒排索引都会在搜索引擎中进行使用，`倒排索引`是对`正排索引中的term`进行**排序**后进行构建的。

在一些搜索引擎中索引还会包含额外的信息：term在document出现的频率和位置。term出现的频率通常用来计算搜索结果的相关性，位置通常方便在docum中搜索短语。

### 倒排索引的结构 

由：`term index`、`term dictionary`、`postling list`
- term dictionary: 词项字典，就是一个个的单词
- postling list： 包含了当前term的docId列表
- term index: 快速检索term

#### postings list 压缩算法

##### Frame Of Reference
假设有一个数组：**`[73, 300, 302, 332, 343, 372]`**，

Frame Of Reference压缩算法的过程如下图所示：

![img](https://cdn.jsdelivr.net/gh/cxyxq/images/es_frame_of_reference_algorithm.png)


压缩效果：
> 由24bytes 压缩到了 7bytes
> 
> 未压缩前：每个数字int占用4byte， 6个数字共 24 bytes
{: .prompt-tip}


拆解下FOR的算法步骤：

1. 计算每个值的和前一个值的差值`delta` ：`[73, 227, 2, 30, 11, 29]`

2. 将delta数组分割到不同的block中：`[73, 227, 2]`和`[30,11,29]`

3. 分别存储每个block中最大的`bits`:

   - block-0: 8bits , 占用1byte
   - block-1: 5bits, 占用1byte，最小1byte

4. 最终占用的字节数：7bytes

   > 1byte + 3x8bits(1byte) + 1byte + 3x5bits(2byte) = 7bytes
   >  3x5bits = 15bits，但是换算字节后至少要2byte
   >  

##### Roaring bitmaps

假设有一个数组：**`[1000, 62101, 131385, 132052, 191173, 196658]`**，

Roaring bitmaps压缩算法的过程如下图所示：

![roaring_bitmaps_algorithm](https://cdn.jsdelivr.net/gh/cxyxq/images/es_roaring_bitmaps_algorithm.png)

拆解下上面的步骤：

1.将每个数字分别进行2个运算： `N/65536`得到除数、`N%65536`得到余数

> `(0,1000)  (0,62101) (2,313) (2,980) (2,60101) (3, 50) `
{: .prompt-tip}

2.将除数相同的ID对应的余数存放到同一个`block`块中，听着比较绕，看示例：

> - block-0: [1000, 62101]
- *block-1: [] 空*, 没有除数为1的
- block-2: [313, 980，60101]
- block-2: [50]
{: .prompt-tip}

3.根据每个`block`块中的元素的个数来决定使用哪种压缩
> - 小于4096时使用`short[]`， 为什么使用`short`，因为每个值的大小不会超过`65535`
- 大于4096时使用 `bitmap`
{: .prompt-tip}

至于为什么选择4096这个值，是因为当个数超过4096时，使用 `bitmap` 更节省内存，如下图所示：
![内存占用](https://cdn.jsdelivr.net/gh/cxyxq/images/es_roaring_bitmaps_memory_compare.png)



## 参考文章
[What is an Index (1)](https://www.elastic.co/cn/blog/found-indexing-for-beginners-part1){:target="_blank"}

[Document Parsing and Tokenization (2)](https://www.elastic.co/cn/blog/found-indexing-for-beginners-part2){:target="_blank"}

[What Does an Index Look Like? (3)](https://www.elastic.co/cn/blog/found-indexing-for-beginners-part3#inverted-index){:target="_blank"}

[Frame of Reference and Roaring Bitmaps](https://www.elastic.co/cn/blog/frame-of-reference-and-roaring-bitmaps){:target="_blank"}