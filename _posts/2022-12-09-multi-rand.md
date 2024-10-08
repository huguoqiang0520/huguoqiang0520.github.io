---
layout: post
title: 有限连续范围内生成不重复随机数及其应用
categories: [随机数]
mermaid: true
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 背景/前言

最近遇到了两个可以转化为本文题目问题的需求点（具体需求在下面应用节选会讲到），决定整理和记录下来。

# 技术问题描述

- 问题：给定一个连续且有限的值范围（从min到max，步长为step），从中随机取出不重复的n个值。
- 举例：从108～10086的整数范围内，取出10个不重复的随机数。

# 技术问题方案

解决问题总会有一些通用的思想，比如：穷举、分治、迭代、倍增、回溯、贪心、动规

那运用这些思想可以想到如下解法（当然还会有别的解法）

---

## 粗暴随机法

> 1. 维护一个结果集
> 2. 在范围内生成一个随机数，并判断随机数是否在结果集内
> 3. 随机数不在结果集内则加入结果集，并回到2步骤直至结果集内数量满足
> 4. 随机数在结果集内则直接回到2步骤直至结果集内数量满足

```php
function unique_rand($min, $max, $num) {
    $count = 0;
    $return = [];
    while ($count < $num) {
        $return[] = mt_rand($min, $max);
        // 借助翻转法识别并剔除重复值
        $return = array_flip(array_flip($return));
        $count = count($return);
    }
    shuffle($return);
    return $return;
}
```

---

## 剔除随机法

> 1. 穷举全量范围并构建数组（k1=min,k2=min+step,kx=min+(x-1)*step,...kn=max）
> 2. 每次在k范围内产生一个随机数a，并将ka的值放入结果集，同时用kn的值填充ka，n自减
> 3. 重复2步骤直至结果集内数量满足

```php
function unique_rand($min, $max, $num) {
    $numbers = range($min, $max);
    // 实现上偷个懒，直接采用内置打乱函数，再取前多少位的方式
    shuffle($numbers);
    $result = array_slice($numbers, 0, $num);
    return $result;
}
```

---

## 均匀分布法

> 1. 将全量范围分为均匀的n段子范围
> 2. 从每一段中获取一个随机数放入结果集
>
> 再升级一下就是双倍随机法（但双倍随机法需要处理额度不足问题）

```php
// 代码就偷个懒不写了
```

---

## 语言内置函数/标准库

有的语言可能内置了相关方法，比如

```php
/**
 * Pick one or more random keys out of an array
 * @link https://php.net/manual/en/function.array-rand.php
 * @param array $array <p>
 * The input array.
 * </p>
 * @param int $num [optional] <p>
 * Specifies how many entries you want to pick.
 * </p>
 * @return int|string|array If you are picking only one entry, array_rand
 * returns the key for a random entry. Otherwise, it returns an array
 * of keys for the random entries. This is done so that you can pick
 * random keys as well as values out of the array.
 */
function array_rand(array $array, int $num = 1): array|string|int {}
```

# 技术问题场景

## 场景罗列

不同的方案肯定各有优劣，需要结合具体场景来分析。
我们先考虑一下影响实现的维度有：

- 范围基数: c = ((max-min)/step)+1；这个值有大/小的维度取值
- 目标结果比例: r = n/c；这个值有少/中/多的维度取值

---

因为我们可以罗列场景如下：

1. c小r少： 比如从100个里面产生3个
2. c小r中： 比如从100个里面产生48个
3. c小r多： 比如从100个里面产生97个
4. c大r少： 比如从1000000个里面产生10个
5. c大r中： 比如从1000000个里面产生489876个
6. c大r多： 比如从1000000个里面产生999997个

---

但考虑到3和6两种r多的情况可以时1和4情况的取反，因此合并：

1. c小r少： 比如从100个里面产生3个
2. c小r中： 比如从100个里面产生48个
3. c大r少： 比如从1000000个里面产生10个
4. c大r中： 比如从1000000个里面产生489876个

## 场景与解法匹配

可以多种解法混用

---

|       |  c小r少   |  c小r中  |  c大r少  |  c大r中  |
|:-----:|:-------:|:------:|:------:|:------:|
| 粗暴随机法 | 高 [^1]  | 低 [^2] | 高 [^1] | 低 [^2] |
| 剔除随机法 | 高️ [^3] | 高 [^3] | 低 [^4] | 低 [^4] |
| 均匀分布法 | 低️ [^5] | 高 [^6] | 低 [^5] | 高 [^6] |

[^1]: r少，不容易生成重复值导致时间复杂度升高

[^2]: r中，容易生成重复值导致时间复杂度升高

[^3]: c小，内存容纳得下，通过时间换空间

[^4]: c大，内存分配消耗增大，且可能内存不足

[^5]: r少，结果分布会比较均匀，如果要求随机性高则不适用

[^6]: r中，结果分布本身跟随概率分布就会比较均匀，因此此方法匹配度上升

# 业务应用

## 连抽N份奖品

- 需求描述：用户达成特定任务后有资格参与最终的抽奖环节，在抽奖环节通过管理后台进行中奖人员抽取，同等级奖品需同时抽出
- 转换关联：将有资格的用户放在一张表中，抽取时获取最新记录自增ID N，就转换成了从1～N中生成X个不重复随机数的问题

## 红包瓜分（抢红包）

- 需求描述：一个房间有N个座位，每个座位可以让一个人入座参与一场单人小游戏（不需要同时开局同时结束），且一个房间对应一定金额的红包A，每个人完成单人小游戏后可以瓜分其中一定的金额
- 转换关联：单人完成游戏的时间点是不同的，但可以将红包额度瓜分的时机前置到房间初始化之时，再结合线段分割法，就转换成了先从0～A中生成N-1个不重复随机数，再以这N-1个随机数排序座位切割点构成N段随机长度问题