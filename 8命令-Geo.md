# 概览

本文章记录Geo的命令含义及事件复杂度

|命令|说明|时间复杂度|版本|
|--|--|--|--|
|GEOADD key longitude latitude member [longitude latitude member …]|将给定的空间元素（纬度、经度、名字）添加到指定的键里面。 这些数据会以有序集合的形式被储存在键里面|O(Log(N))|3.2.0|
|GEOPOS key member [member …]|从键里面返回所有给定位置元素的位置|O(Log(N))|3.2.0|
|GEODIST key member1 member2 [unit]|返回两个给定位置之间的距离|O(Log(N))|3.2.0|
|GEORADIUS key longitude latitude radius m km ft mi [WITHCOORD] [WITHDIST] [WITHHASH] [ASC DESC] [COUNT count]|以给定的经纬度为中心， 返回键包含的位置元素当中， 与中心的距离不超过给定最大距离的所有位置元素。|O(N+log(M))|3.2.0|
|GEORADIUSBYMEMBER key member radius m\|km\|ft\|mi [WITHCOORD] [WITHDIST] [WITHHASH] [ASC\|DESC] [COUNT count]|这个命令和 GEORADIUS 命令一样， 都可以找出位于指定范围内的元素， 但是 GEORADIUSBYMEMBER 的中心点是由给定的位置元素决定的， 而不是像 GEORADIUS 那样， 使用输入的经度和纬度来决定中心点。|O(log(N)+M)|3.2.0|
|GEOHASH key member [member …]|返回一个或多个位置元素的 Geohash 表示|O(log(N)) |3.2.0|
