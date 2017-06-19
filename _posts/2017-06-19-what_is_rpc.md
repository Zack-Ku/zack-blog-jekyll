---
layout: post
title:  "到底RPC是什么鬼"
date:   2017-06-19
excerpt: "RPC到底是什么东西呢？和HTTP有什么关系？"
published: false
<!-- feature: http://oboi2pfvn.bkt.clouddn.com/es6.jpg -->
tag:
- rpc 
- lessons
comments: true
---

当用aggregate处理大量数据超出大小限制时，可以用一下方法解决。

-----------

## Mongo的限制

1. 结果大小限制
    对于聚合操作的结果返回，MongoDB是对其有大小限制的。从2.6版本开始，MongoDB对大小的限制为16M。
    
2. 内存大小限制
    对于聚合操作过程，MongoDB的对内存限制是100M。一般来说，超过这个限制就会产生一个错误。但如果想超过这个限制则需要在options参数内加上
    
        allowDiskUse: true
        
## 解决方法

针对上面 **2.** 的解决方法以给出。   
对于 **1.**  有以下方法：

a. match完先project，去掉不需要的字段，减少返回值大小    
b. 用 MongoDB 提供的GridFS API 存储数据的大小

**参考 :**  


<https://docs.mongodb.com/manual/core/aggregation-pipeline-limits/>  aggregation-pipeline-limits MongoDB
<https://docs.mongodb.com/manual/reference/limits/#BSON-Document-Size>  BSON-Document-Size MongoDB


