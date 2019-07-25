# 概览

本文章记录List的命令含义及事件复杂度


|命令|说明|时间复杂度|版本|
|--|--|--|--|
|LPUSH key value [value …]|将一个或多个值 value 插入到列表 key 的表头|O(1)|1.0.0|
|LPUSHX key value|将值value插入到列表key的表头,当且仅当key存在并且是一个列表|O(1)|2.2.0|
|RPUSH key value [value …]|将一个或多个值 value 插入到列表 key 的表尾(最右边)|O(1)|1.0.0|
|RPUSHX key value|将值 value 插入到列表 key 的表尾，当且仅当 key 存在并且是一个列表|O(1)|2.2.0|
|LPOP key|移除并返回列表key的头元素|O(1)|1.0.0|
|RPOP key|移除并返回列表key的尾元素|O(1)|1.0.0|
|RPOPLPUSH source destination|命令 RPOPLPUSH 在一个原子时间内，执行以下两个动作：1. 将列表source中的最后一个元素(尾元素)弹出，并返回给客户端，2.将 source 弹出的元素插入到列表 destination ，作为 destination 列表的的头元素|O(1)|1.2.0|
|LREM key count value|根据参数count的值，移除列表中与参数value相等的元素|O(N)|1.0.0|
|LLEN key|返回列表 key 的长度。如果 key 不存在，则 key 被解释为一个空列表，返回 0|O(1)|2.2.0|
|LLEN key|返回列表 key 的长度。如果 key 不存在，则 key 被解释为一个空列表，返回 0 |O(1)|2.2.0|
|LINDEX key index|返回列表 key 中，下标为 index 的元素|O(N)|1.0.0|
|LINSERT key BEFORE|AFTER pivot value|将值 value 插入到列表 key 当中，位于值 pivot 之前或之后|O(N)|2.2.0|
|LSET key index value|将列表key下标为index的元素的值设置为 value|O(1)1.0.0|
|LRANGE key start stop|返回列表 key 中指定区间内的元素，区间以偏移量 start 和 stop 指定|O(S+N)|1.0.0|
||对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除|O(N)|1.1.0|
|BLPOP key [key …] timeout|BLPOP 是列表的阻塞式(blocking)弹出原语|O(1)|2.2.0|
|BRPOPLPUSH 是 RPOPLPUSH source destination 的阻塞版本|BRPOPLPUSH 是 RPOPLPUSH source destination 的阻塞版本|O(1)|2.2.0|
