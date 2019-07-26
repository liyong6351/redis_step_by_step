# 概览

本文章记录ZSet的命令含义及事件复杂度

|命令|说明|时间复杂度|版本|
|--|--|--|--|
|ZADD key score member [[score member] [score member] …]|将一个或多个member元素及其score值加入到有序集key当中|O(M*log(N))|1.2.0|
|ZSCORE key member|返回有序集key中,成员member的score值|O(1)|1.2.0|
|ZINCRBY key increment member|为有序集key的成员member的score值加上增量increment|O(log(N))|1.2.0|
|ZCARD key|返回有序集key的成员个数|O(1)|1.2.0|
|ZCOUNT key min max|返回有序集key中,score值在min和max之间(默认包括score值等于min或max)的成员的数量|O(log(N))|2.0.0|
|ZRANGE key start stop [WITHSCORES]|返回有序集 key 中，指定区间内的成员|O(log(N)+M),N为有序集的基数,而M为结果集的基数|1.2.0|
|ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]|返回有序集key中，所有score值介于min和max之间(包括等于min或max)的成员|O(log(N)+M),N为有序集的基数,M为被结果集的基数|1.0.5|
|ZRANK key member|返回有序集key中成员member的排名|O(log(N))|2.0.0|
|ZREVRANGE key start stop [WITHSCORES]|返回有序集key中，指定区间内的成员|O(log(N)+M)|1.2.0|
|ZREVRANK key member|返回有序集 key 中成员 member 的排名|O(log(N))|2.0.0|
|ZREM key member [member …]|移除有序集key中的一个或多个成员,不存在的成员将被忽略|O(M*log(N)),N为有序集的基数,M为被成功移除的成员的数量|1.2.0|
|ZREMRANGEBYRANK key start stop|移除有序集 key 中，指定排名(rank)区间内的所有成员|O(log(N)+M)|2.0.0|
|ZREMRANGEBYSCORE key min max|移除有序集key中,所有score值介于min和max之间(包括等于min或max)的成员|O(log(N)+M),N为有序集的基数,而M为被移除成员的数量|1.2.0|
|ZRANGEBYLEX key min max [LIMIT offset count]||O(log(N)+M)|2.8.9|
|ZLEXCOUNT key min max|对于一个所有成员的分值都相同的有序集合键 key 来说， 这个命令会返回该集合中， 成员介于 min 和 max 范围内的元素数量|O(log(N))|2.8.9|
|ZREMRANGEBYLEX key min max|对于一个所有成员的分值都相同的有序集合键 key 来说， 这个命令会移除该集合中， 成员介于 min 和 max 范围内的所有元素|O(log(N)+M)|2.8.9|
|ZSCAN|参见SCAN|O(1)或者O(N)|2.8.0|
|ZUNIONSTORE destination numkeys key [key …] [WEIGHTS weight [weight …]] [AGGREGATE SUM MIN MAX]|计算给定的一个或多个有序集的并集，其中给定key的数量必须以numkeys参数指定，并将该并集(结果集)储存到 destination|O(1)|1.2.0|
|ZINTERSTORE destination numkeys key [key …] [WEIGHTS weight [weight …]] [AGGREGATE SUM MIN MAX]||O(N*K)+O(M*log(M))， N 为给定 key 中基数最小的有序集， K 为给定有序集的数量， M 为结果集的基数|2.0.0|
