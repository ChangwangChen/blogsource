---
title: Go Map简介
date: 2020-07-22 13:50:45
categories: Go
tags: [Go]
---

# 数据结构

1. 特殊的拉链法
2. hash(key) 之后， 最低几位(B-1位)找到 bucket, 然后遍历比较最高8位找到key的位置
3. bucket中没有， 接着遍历overflow溢出桶

#  扩容

## 条件

1. 装载因子已经超过 6.5， h.count > (1<<2^h.B) * 6.5； (h.count *= 2)

2. overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15。
(h.count 不变；sameSizeGrow)

## 规则

1. 渐进式扩容 （类似 Redis 的 hashtable）
2. 获取、插入（修改）、删除时需要考虑渐进扩容的影响
