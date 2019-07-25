# 概览

本文章记录Hash的命令含义及事件复杂度

|命令|说明|时间复杂度|版本|
|--|--|--|--|--|
|HSET hash field value|将哈希表hash中域field的值设置为value,如果给定的哈希表并不存在，那么一个新的哈希表将被创建并执行HSET操作。如果域field已经存在于哈希表中， 那么它的旧值将被新值value覆盖|O(1)|2.0.0|
|HSETNX hash field value|当且仅当域field尚未存在于哈希表的情况下，将它的值设置为value。如果给定域已经存在于哈希表当中， 那么命令将放弃执行设置操作|O(1)|2.0.0|
|HGET hash field|返回哈希表中给定域的值|O(1)|2.0.0|
|HEXISTS hash field|检查给定域 field 是否存在于哈希表 hash 当中|O(1)|2.0.0|
|HDEL key field [field …]|删除哈希表 key 中的一个或多个指定域，不存在的域将被忽略|O(N)|2.0.0|
|HLEN key|返回哈希表 key 中域的数量|O(1)|1.0.0|
|HSTRLEN key field|返回哈希表key中,与给定域field相关联的值的字符串长度|O(1)|3.2.0|
|HINCRBY key field increment|为哈希表key中的域field的值加上增量increment|O(1)|2.0.0|
|HINCRBYFLOAT key field increment|为哈希表key中的域field加上浮点数增量increment|O(1)|2.6.0|
|HMSET key field value [field value …]|同时将多个field-value(域-值)对设置到哈希表key中。此命令会覆盖哈希表中已存在的域|O(N)|2.0.0|
|HMGET key field [field …]|返回哈希表 key 中，一个或多个给定域的值|O(N)|2.0.0|
|HKEYS key|返回哈希表 key 中的所有域|O(N)|2.0.0|
|HVALS key|返回哈希表 key 中所有域的值|O(N)|2.0.0|
|HGETALL key|返回哈希表 key 中，所有的域和值|O(N)|2.0.0|
|O(N)|HSCAN 命令和 ZSCAN 命令都用于增量地迭代|O(N)|2.8.0|
