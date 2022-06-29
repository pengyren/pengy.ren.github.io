---
title: MySQL时间戳索引失效SQL慢查问题排查
date: 2022/06/25 17:37:27
categories: code
tags: mysql
description:  MySQL时间戳索引失效SQL慢查问题排查
---

# MySQL时间戳索引失效SQL慢查问题排查

<!-- toc -->


最近一次sql慢查的排查过程其中一个点是自己之前没有关注过的，分享一下



首先有一张表tableA {

  id 主键 自增

  create_time 创建时间 默认mysql当前时间自动填充

  ..... 其他字段

}，

表结构中含有常规的id, create_time, update_time之类的字段，create_time字段建了索引；



tableA每天会在上午6点50左右接收一批同步数据，数据量大约在40000+

该表保留一周的数据，因此该表数据的总行数目前维持在35W+

查询的sql使用了create_time的查询条件，查询的时间区间时1天，理论上能查出的数据在40000+，占数据总量的约11%，理应走索引查询



但是，实际的执行过程显示，该sql还是走了全表扫描；经验证，即使查询条件只有create_time也是走了全表扫描；

我们尝试了缩短查询时间区间，当查询区间内不含数据时才走了索引查询，但是一旦涵盖了当天同步过来的数据，就变成了全表扫描；

于是我们对**数据的内容**进行了分析，发现由于这批数据基本都是在那个时间点同步过来，create_time是数据insert到表中时的当前时间自动获取的默认值，因此create_time会有大量的重复；

我们select count(distinct create_time) from tableA后得到35W+的数据，create_time只有**97**种

我们再慢查SQL的部分统计了count(distinct create_time)，得到的create_time是**11**种，

**结论：**

由此我们断定，像这样的根茎节点非常少，一个父节点下面挂了3700多个叶子节点的情况，即便是我们对这个create_time建了索引，但是走索引和不走索引相差无几，故mysql引擎选择了全表扫描；

> 在建立索引的时候，要根据列基数来建立，列基数=列中不同的数据/除以总数据，越接近1表示重复数据越少，越适合建立索引。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAyMC83LzEwLzE3MzM3YjI5MGNhNWJiZmY?x-oss-process=image/format,png)


