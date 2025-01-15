---
date: '2025-01-14T20:07:06+08:00'
title: 'HdrHistogram算法和实现'
---

## 背景

想必大家对延迟（latency）指标一定不会不陌生，它衡量的是事件响应时间的快慢，常见的有网络延迟等。[考虑到平均延迟会掩盖真实的问题](https://www.elastic.co/cn/blog/averages-can-dangerous-use-percentile)，监控中一般都采用百分位（Quantile）指标进行评价，如P50/P90/P99，表示前50%/90%/99%的延迟数据。

简单地，我们可以将延迟数据排序，然后取对应百分位的数据即可得到Quantile指标，但显然这样的空间开销和排序时间开销都比较大，不是一个合适的做法。目前工业界和学术界都有比较成熟的百分位计算方法，这些方法都有不同的侧重点，有的更关注并行计算能力，有的更关注准确度。常见的百分位算法可以分为如下几类：

- 基于样本的
  - 蓄水池采样
- 基于桶的
  - 基于静态桶
    - HdrHistogram
  - 基于动态桶
    - CKMS, t-digest

简单来说，百分位算法需要支持接口：

```cpp
struct quantile_algo {
  void add(int x);
  int percentile_at(double percentile);
}
```

`add`方法将某个数据加入统计，`percentile_at`方法计算对应的百分位指标。

本文关注的是如何更快地在线计算延迟。这里的更快，指的是也是更低的延迟，更准确的说是`add`方法的延迟。一般来说`percentile_at`方法相对调用频率较少，且不在关键路径上。

不同Quantile算法的介绍、时间和空间复杂度对比可见[这篇文章](https://caorong.github.io/2020/08/03/quartile-%20algorithm/)。这里我们直接说结论，HdrHistogram更符合我们的场景，所以本文着重关注的也是HdrHistogram。

## HdrHistogram

上面我们提到，HdrHistogram基于静态桶做百分位计算。那么什么叫基于桶的算法呢？考虑到空间开销和时间的开销，我们不能把所有的数据都准确记录下来，那么只能用关键点来sketch多个数据，桶就是sketch数据特征的手段。
举个例子，统计延迟时，如果数据都分布在300ms内，那么可以划分为3个桶:

```
Bucket           1                2               3
Range         [0, 100)        [100, 200)      [200, 300)
```

当新遇到一个延迟数据，我们只需要将对应的桶计数+1即可。当有90%的数据都分布在前两个桶中时，我们可以说P90是200ms了。显然这种sketch肯定是有损的，很多情况下不能准确得到Quantile的值，但桶越多，精确度也会越高。

上面的例子中我们提前知道数据分布范围，但是如果数据的变化较大，不好提前划分桶该怎么办呢？HdrHistogram采用了一种讨巧的方式，结合了指数划分和等间距划分两种分桶方式，将区间内的桶压缩表示出来。取一段代码[注释](https://github.com/HdrHistogram/HdrHistogram_rust/blob/a3818d6ce81556010c7f04f423e1fa28a9ff1b5c/src/lib.rs#L221-L245) （修改了typo后）：

```cpp
/// `Histogram` is the core data structure in HdrSample. It records values, and performs analytics.
///
/// At its heart, it keeps the count for recorded samples in "buckets" of values. The resolution
/// and distribution of these buckets is tuned based on the desired highest trackable value, as
/// well as the user-specified number of significant decimal digits to preserve. The values for the
/// buckets are kept in a way that resembles floats and doubles: there is a mantissa and an
/// exponent, and each bucket represents a different exponent. The "sub-buckets" within a bucket
/// represent different values for the mantissa.
///
/// To a first approximation, the sub-buckets of the first
/// bucket would hold the values `0`, `1`, `2`, `3`, …, the sub-buckets of the second bucket would
/// hold `0`, `2`, `4`, `6`, …, the third would hold `0`, `4`, `8`, and so on. However, the low
/// half of each bucket (except bucket 0) is unnecessary, since those values are already covered by
/// the sub-buckets of all the preceeding buckets. Thus, `Histogram` keeps the top half of every
/// such bucket.
///
/// For the purposes of explanation, consider a `Histogram` with 2048 sub-buckets for every bucket,
/// and a lowest discernible value of 1:
///
/// <pre>
/// The 0th bucket covers 0...2047 in multiples of 1, using all 2048 sub-buckets
/// The 1st bucket covers 2048..4095 in multiples of 2, using only the top 1024 sub-buckets
/// The 2nd bucket covers 4096..8191 in multiple of 4, using only the top 1024 sub-buckets
/// ...
/// </pre>
```

HdrHistogram将桶分成了2层，bucket和sub-bucket，同一bucket内不同sub-bucket等间隔划分，不同bucket指数级别划分。

注释中的例子，每个bucket中有2048个sub-bucket：

- 第0个bucket负责`[0, 2048)`的数据，每个sub-bucket等间隔`1`划分，即桶只负责表示`(1, 2, 3, ..., 2047)`
- 第1个bucket负责`[0, 4096)`的数据，每个sub-bucket等间隔`2`划分。但由于前面的bucket已经cover了前半部分数据，即桶只负责表示`(2048, 2050, 2052, ... , 4094)`
- 第2个bucket负责`[0, 8192)`的数据，每个sub-bucket等间隔`4`划分。但由于前面的bucket已经cover了前半部分数据，即桶只负责表示`(4096, 4100, 4104, ... , 8188)`

那么如何将一个数放到对应的bucket中呢？还是以注释中的例子为例。因为`2048=2^11`，数字二进制中前11位表示sub-bucket index, 剩下的位数表示bucket index。

举个例子，对于数据`4099`来说，其二进制表示为`0b0001'0000'0000'0011`，前半部份`0b1'0000'0000'00`, 表示sub-bucket为`1024`（注意除0号bucket外，其余bucket的sub-bucket都只用后半部分），后半部分`0b11`长度为2，表示bucket-index为`2`，根据上面的说明，我们会用表示`4096`的这个桶去存储它。

HdrHistogram的bucket和sub-bucket数量是如何确定的呢？有三个参数，`lowest_discernible_value`, `highest_trackable_value`, `significant_figures`，分别控制了最小分辨单位、最大统计数据值，精确度大小，进而控制了bucket和sub-bucket的数量。

如何取Quantile值就相对简单一些了。根据数据总量和每个桶内的数量，从小到大找到百分位的桶和它对应的值即可。

## 性能

HdrHistogram的add操作没有复杂的计算操作，只是简单通过位运算得到桶下标，并将计数+1，这个过程中只访问内存1次。
理论上来讲，add操作的开销约等于访存开销，根据参数不同，桶的数目不同，访问内存开销也不同，但一般也在ns级别，在各种Quantile算法中是很小的了。（参考[这里](https://www.p99conf.io/session/data-structures-for-high-resolution-real-time-telemetry-at-scale/)的benchmark）

如果想让HdrHistogram更快，可以在编译时就确定参数，那么运行时就可以通过立即数进行计算，减少可能的寄存器读写。这里夹带一些私货，[hdrcpp](https://github.com/zingdle/hdrcpp)魔改了HdrHistogram_c的实现，在我的机器上可以将单次add操作从\~4ns降低到\~2.5ns。
