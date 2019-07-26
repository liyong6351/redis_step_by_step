# 概览

本文章记录HyperLogLog的命令含义及事件复杂度，HyperLogLog的标准误差是0.81%

|命令|说明|时间复杂度|版本|
|--|--|--|--|
|PFADD key element [element …]|将任意数量的元素添加到指定的 HyperLogLog 里面|O(1)|2.8.9|
|PFCOUNT key [key …]|当 PFCOUNT key [key …] 命令作用于单个键时， 返回储存在给定键的 HyperLogLog 的近似基数，如果键不存在，那么返回0|当命令作用于单个 HyperLogLog 时， 复杂度为 O(1) ， 并且具有非常低的平均常数时间。当命令作用于 N 个HyperLogLog时，复杂度为O(N)，常数时间也比处理单个 HyperLogLog 时要大得多|2.8.9|
|PFMERGE destkey sourcekey [sourcekey …]|将多个 HyperLogLog 合并（merge）为一个 HyperLogLog ， 合并后的 HyperLogLog 的基数接近于所有输入 HyperLogLog 的可见集合（observed set）的并集|O(N) |2.8.9|
