# 概览

本文章记录Set的命令含义及事件复杂度

|命令|说明|时间复杂度|版本|
|--|--|--|--|
|SADD key member [member …]|将一个或多个member元素加入到集合key当中,已经存在于集合的member元素将被忽略|O(N)|1.0.0|
|SISMEMBER key member|判断member元素是否集合key的成员|O(1)|1.0.0|
|SPOP key|移除并返回集合中的一个随机元素|O(1)|1.0.0|
|SRANDMEMBER key [count]|如果 count 为正数，且小于集合基数，那么命令返回一个包含 count 个元素的数组，数组中的元素各不相同。如果 count 大于等于集合基数，那么返回整个集合。如果 count 为负数，那么命令返回一个数组，数组中的元素可能会重复出现多次，而数组的长度为 count 的绝对值|O(N)或O(1)|1.0.0|
|SREM key member [member …]|移除集合 key 中的一个或多个member元素，不存在的member元素会被忽略|O(N)|1.0.0|
|SMOVE source destination member|如果source集合不存在或不包含指定的member元素,则SMOVE命令不执行任何操作,仅返回0.否则,member元素从source集合中被移除,并添加到destination集合中去|O(1)|1.0.0|
|SCARD key|返回集合key集合中元素的数量|O(N)|1.0.0|
|SMEMBERS key|返回集合key中的所有成员|O(N)|1.0.0|
|SCAN cursor [MATCH pattern] [COUNT count]|SCAN命令及其相关的SSCAN命令、HSCAN命令和ZSCAN命令都用于增量地迭代(incrementally iterate)一集元素|O(N)或O(1)|2.8.0|
|SINTER key [key …]|返回一个集合的全部成员，该集合是所有给定集合的交集|O(N*M) N:Set的成员数，M需要给定集合的个数|1.0.0|
|SINTERSTORE destination key [key …]|这个命令类似于 SINTER key [key …] 命令，但它将结果保存到 destination集合，而不是简单地返回结果集|O(N*M) N:Set的成员数，M需要给定集合的个数|1.0.0|
|SUNION key [key …]|返回一个集合的全部成员，该集合是所有给定集合的并集|O(N)，N为给定集合的成员数之和|1.0.0|
|SUNIONSTORE destination key [key …]|这个命令类似于SUNION key [key …]命令，但它将结果保存到 destination集合|O(N)，N为给定集合的成员数之和|1.0.0|
|SDIFF key [key …]|返回一个集合的全部成员，该集合是所有给定集合之间的差集|O(N)|1.0.0|
|SDIFFSTORE destination key [key …]|这个命令的作用和SDIFF key [key …]类似,但它将结果保存到destination集合|O(N)|1.0.0|
