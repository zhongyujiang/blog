
---
title: "Bitmap 编码及应用"
date: 2023-09-29T11:04:00+08:00
draft: false
mathjax: true
---

### 前言
前段时间接触了很多关于 Bitmap 数据结构的知识，所以想写篇博客做个总结记录。本文的内容很大部分其实来自于：
[Using Bitmaps to Perform Range Queries](https://www.featurebase.com/blog/range-encoded-bitmaps)，一篇写的非常好的关于 Bitmap 索引的科普文章。本文主要是在原文的基础上添加了更详细的注解以及补充，尤其是在 Bit-sliced Indexes (BSI)
部分补充了来自 [Improved query performance with variant indexes
](https://dl.acm.org/doi/10.1145/253262.253268#:~:text=Management%20of%20data-,The%20read%2Dmostly%20environment%20of%20data%20warehousing%20makes%20it%20possible,where%20concurrent%20updates%20are%20present.) 一文中的使用 BSI 加速查询的内容。

### Bitmap 简介
Bitmap 是一种将「一组元素」 映射到「一组比特位」的数据结构。元素之间的关系可以被表示为布尔值 --- 关系存在或不存在被映射为 True 或 False --- 存储为一系列的 1 和 0。
这些布尔集合就是位图，通过在各种按位的操作中使用 Bitmap，我们可以非常快速地执行复杂的查询。此外，Bitmap 压缩技术允许我们以非常紧凑的格式表示大量数据，可以减少存储数据和执行查询所需的资源量。

Bitmap 有多种编码方式，分别适用于不同的查询场景。

### 等值编码 Bitmap(Equality-encoded Bitmaps)
在等值编码的 Bitmap 中，每一个 bit 都表示一个特征（值）是否存在。
假设我们有一组描述动物特征的数据集，我们可以将其编码为如下图的 Bitmap，横轴为动物种类，纵轴为特征，比特位为 1 表示对应关系存在，为 0 表示关系不存在，比如 Banana Slug 是 Invertebrate 的，那么这两条线的交集就会被编码为 1 ，表示这个关系是存在的。
基于等值编码的 Bitmap ，我们可以用位操作来快速实现一些查询，比如说如果我们想找到即是 Invertebrate 又是 Breaths Air 的动物，可以直接将 Invertebrate 和 Breaths Air 两行拿出来做「按位与」操作，其结果中 1 对应的横轴元素就是符合条件的查询结果了。 

![equality-encoded-bitmap.png](/img/bitmap-encoding/equality-encoded-bitmap.png)

上面举例的查询是一个等值查询操作，但是如果有范围查询呢？假如我们有一组记录各种动物被圈养数量的数据集，我们仍然可以将其进行等值编码。如果想要找出被圈养数量大于100的动物种类：Captive > 100 ，我们可以将所有 Captive 大于 100 的行拿出来做「按位或」操作，其结果中 1 对应的横轴元素就是符合条件的查询结果了。

![equality-encoded-bitmap-range-predicate](/img/bitmap-encoding/equality-encoded-bitmap-range-predicate.png "400px") 

这样操作确实可行，但是效率不高：
- 如果基数稍微大一些，就需要创建许多位图来表示所有可能的值
- 如果要做范围过滤，需要对范围内的每个可能值执行OR操作

#### 分桶等值编码
为了解决上述问题，我们可以对数值进行分桶：
![bucket-encoded-bitmap](/img/bitmap-encoding/bucket-encoded-bitmap.png  "400px") 

使用分桶编码的方式可以大幅提升范围查询的效率（「按位或」操作次数会减少很多），但是这同时也带来了新的问题：查询假阳性，因为分桶后数据无法被精准的记录了，返回的结果可能是不准确的。

### 范围编码 Bitmap （Range-Encoded Bitmaps）
如果既想要保证查询的精准，又不想做低效率的多次「按位或」操作，我们可以考虑使用Range-Encoded Bitmaps。

![range-encoded-bitmap](/img/bitmap-encoding/range-encoded-bitmap.png  "400px")  

Range-Encoded Bitmaps 的编码方式和 equality-encoding 类似，但是它不只设置与特定值对应的位为 1 ，还为每个大于实际值的位置都置为 1 。例如，因为 Koala Bears 被圈养的数量为 14 ，图中每个大于等于 14 的位置都被置为了 1 。
现在 Bitmap 不仅可以表示特定圈养数量的动物，还可以表示小于或大于某个数量的动物，比如说：
- 查询 Captive < 48 的动物，我们可以直接查看 Captive 为 47 这一行的值为 1 的列所对应的动物，因为值为 0 的动物一定都是圈养数量 > 47 的
- 查询 Captive > 48 的动物，可以用最大值一行的位图（第956行）减去第 48 行所在的位图，其结果就是所有圈养数量大于 48 的动物了
> 如果一种动物 Captive > 48 ，那么其在 Captive = 48 这一行的值一定为 0 ， 如果 Captive ≤ 48，那么其在 Captive = 48 这一行的值一定为 1 ， 如果Captive值存在，那么其在最大行的值一定为 1 ， 所以判断 > 查询时，两行相减可以得到结果

现在我们可以使用 Range-Encoded Bitmaps 同时做到「查询准确」 + 「效率高」 了，但是现在还有两个问题：
- 空间效率偏低：需要为每个 Captive 值记录一行 Bit 值
- 构建代价较高：构建 Bitmap 时，需要将每个 ≥ 值的位置都置 1
  
### Bit-sliced Indexes
上文提到的数值范围编码其实就是在纵轴记录了一个数字序列，现在想要高效地表示这一系列数字，我们可以很自然地联想到**进位制数字表示法**，下图是纵轴采用十进制记数的等值编码 Bitmap，30 行就可以表示 1000 以内的所有数值：  
![base-10-bit-sliced-bitmap](/img/bitmap-encoding/base-10-bit-sliced-bitmap.png  "400px") 

#### Range-Encoded Bit-Slice Indexes
然后接下来我们可以将「范围编码」和「进位制编码」结合起来（下左图），这样既节省空间，又可以支持范围查询。
另外我们还可以注意到，所有列，只要 Captive 值存在，那么其所有 comp 的最高位（9）的值一定是 1 。
范围编码要求将所有大于实际值的位置都置为 1 ，所以只要值存在，那么所有 comp 的最高位一定会是 1
所以我们还可以将最高位的一行省略掉，不过为了表示不存在的值，我们需要额外一行来表示 NOT NULL （下右图）。现在我们只需要（（3* 9）+ 1 ）= 28 行就可以表示 [0,  999] 区间以内的所有数值。
![range-encoded-base-10-bit-sliced-index](/img/bitmap-encoding/range-encoded-base-10-bit-sliced-index.png) 

#### Base-2, Bit-sliced Index
上面提到用十进制进行Bit-sliced编码记录 1000 个值，需要的行数是（（3* 9）+ 1 ）= 28，其实如果做一下归纳总结，可以发现如果有 n 个需要记录的元素，如果使用的进制为 a ，那么记录n条数据需要的行数是：
$$(\log_an * (a-1)) + 1$$

不难发现，a 值越小 ，记录 n 条数据需要的行数就越小，当然 a 必须大于 1，那么自然可以想到使用最小的 2 进制来进行 Bit-sliced 编码。  

同样，我们可以省略掉最高位的那一行，并且加上额外的一行来表示 NOT NULL，也就是说表示 n 个值最多只需要 \\( \log_2n + 1 \\) 行。从上图可以看到，使用二进制编码的 Bitmap 仅使用 11 行就可以记录 1000 个值了（其实是 1024 个值）！这样紧凑的编码结构在做到「高效」和「省空间」的同时，仍然可以支持快速的范围查询。

在 Base-2 的编码模式下，Bit-sliced Indexes 即使不做 Range 编码，其查询效率也可以通过合理利用其结构得到大幅度提升：[Evaluating the Range using a Bit-Sliced Index](https://dl.acm.org/doi/10.1145/253262.253268#:~:text=Management%20of%20data-,The%20read%2Dmostly%20environment%20of%20data%20warehousing%20makes%20it%20possible,where%20concurrent%20updates%20are%20present.)
![bsi-predicate](/img/bitmap-encoding/base-2-bsi-predicate.png "400px")

**等值查询 EQ**

求 EQ 集的算法比较容易理解：它要求 `c1` 中所有置为 1 的地方在 `Bi` 中也为 1，c1 中所有为 0 的地方 `Bi` 中也为0，这样的出来的 \\( B_{EQ} \\) 中的每一列值就自然都与 `c1`相等了。

**范围查询 GT**

如上图算法中，假设 `c1` 的比特位形式为：
$$ b_{N} b_{N-1} ... b_1 b_0 $$
`C` 中某一行的值 `r` 的比特位形式为：
$$ r_{N} r_{N-1} ... r_1 r_0 $$
算法是从 N 到 0 行遍历的（从最高位到最低位），那么对于 BSI 中的第 i 行来说，当 \\( b_i \\) 的值为 0 时， 如果 \\( c_i \\) 的值为 1，并且有:
$$ c_k = b_k  \quad   for \quad N <= k < i $$
即有 \\( r_N r_{N-1} ... r_{i+1} \\) 与 \\( b_{N} b_{N-1} ... b_{i+1} \\)
相等（这个可以由 \\(B_{EQ}\\) 确定，且这里的遍历顺序是自最高位到最低位，所以即使后面的低位上有 \\( b_j > r_j \\) 也可以忽视了），那么 `C` 一定大于 `c1` 。

这样一来 BSI 上的范围查询的时间复杂度可以由前面等值编码的 O(N) 降低到 O(logN)。

#### Range-Encoded, Base-2, Bit-sliced Index
当然，在 Base-2, Bit-sliced Index 的基础上，我们仍然可以使用范围编码。如下图所示，因为使用 Base-2, Bit-sliced Index 编码后需要的行数大大减少了，所以构建范围编码 Bitmap 时需要置 1 的操作也会大大降低。并且不管是进行等值查询还是范围查询，都只需要 O(1) 的时间复杂度！
![range-encoded-base-2-bit-sliced-index](/img/bitmap-encoding/range-encoded-base-2-bit-sliced-index.png) 

