# 概览

本文讲解String的命令

|命令|说明|时间复杂度|版本|
|--|--|--|--|
|SET key value |将key和value建立关联|O(1)|1.0.0|
|SETNX key value|只在键key不存在的情况下才能键key的值设置为 value|O(1)|1.0.0|
|SETEX key seconds value|key值设置为value，并设置生存时间为 seconds秒|O(1)|2.0.0|
|PSETEX key milliseconds value|同SETEX，过期时间为毫秒| O(1)|2.6.0|
|GET key|返回与键key相关联的字符串值|O(1)|1.0.0|
|GETSET key value|将键 key 的值设为 value ， 并返回键 key 在被设置之前的旧值|O(1)|1.0.0|
|STRLEN key|返回键 key 储存的字符串值的长度|O(1)|2.2.0|
|APPEND key value|如果key存在并且值是一个字符串,把value追加到现有值的末尾,如果key不存在,就像执行SET key value 一样|平摊O(1)|2.0.0|
|SETRANGE key offset value|从偏移量offset开始,用value参数覆写键key储存的字符串值。不存在的键key当作空白字符串处理|O(1)或O(M)|2.2.0|
|GETRANGE key start end|返回键key储存的字符串值的指定部分,字符串的截取范围由start和end两个偏移量决定|O(N)|2.4.0|
|INCR key|为键 key 储存的数字值加上一|O(1)|1.0.0|
|INCRBY key increment|为键key储存的数字值加上增量increment|O(1)|1.0.0|
|INCRBYFLOAT key increment|为键key储存的值加上浮点数增量increment|O(1)|2.6.0|
|DECR key|为键key储存的数字值减去一|O(1)|1.0.0|
|DECRBY key decrement|将键key储存的整数值减去减量decrement|O(1)|1.0.0|
|MSET key value [key value …]|同时为多个键设置值|O(N)|1.0.1|
|MSETNX key value [key value …]|当且仅当所有给定键都不存在时， 为所有给定键设置值。|O(N)|1.0.1|
|MGET key [key …]|返回给定的一个或多个字符串键的值|O(N)|1.0.0|
