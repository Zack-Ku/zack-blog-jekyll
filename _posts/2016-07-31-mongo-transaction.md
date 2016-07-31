---
layout: post
title:  "mongodb拥有事务的解决方案"
date:   2016-07-31
excerpt: "从各种层面的解决方案为mongodb添加事务性的处理"
feature: /assets/img/mongodb.jpg
tag:
- lessons 
- mongodb
- 事务
comments: false
---

最近因为node的并发优点加上mongodb的无事务性被坑了几把，所以找了一些解决方案。  

1. Redis内存锁（不可靠）
2. 用mongodb自带的原子操作 （使用范围局限）
3. 作业队列
4. 二段提交

-----------

##  发生场景

1. 想增加一条记录，当这记录不存在的时候。在代码层面操作，先find查找判断是否存在记录,没有则insert一条新记录。
   但并发同时对一个记录操作时，如find操作读取并发了，由于还没有进行insert操作，导致find都没有找到记录。
   最终导致插入了两条记录。
2. 并发查询更新同一条记录。并发查询获取信息操作，导致更新操作数据不正确。例如某账户余额是10元，在并发查询时，
    同时获得10元信息。然后各提取2元，但是由于查询到的余额是10，所以更新操作无论是并发多少，结果都是8。

-----------

## Redis内存锁
这是当时一种临时解决方案。对于上述场景1、2，只要用记录的唯一标识在redis中增加一个内存锁即可。          
例如我想要为A用户注册新增一条用户记录，当且仅当他之前没有注册过（数据库中没有记录），我们可以用用户的身份证号（唯一），
作为键先写入redis当中加锁（node中有redlock等npm包），然后再插入操作，操作完成后解锁即可。

        Thenjs(function(cont){
                 redlock.lock(key,10,(err,lock) =>{
                    lockobj = lock
                    cont(err)
                 })
               })
        .then(function(cont){
                 update/insert.....
              })
        .then(function(cont){
                redlock.unlock(lockboj,(err,result)=>{
                    cont()
                })  
              })
        .fail(function（cont){
                console.log('err')
              })
             
但是这种方法没有根本解决问题，因为这组合的操作也不是原子性的。假如第一个进来的锁住，然后insert操作因为数据库问题卡住，
一直没有回调，时长超过了redis锁设置的时间，同样会让第二请求进入insert操作。如果说把redis缓存时间设置一个超长时间，例如
1天，那么当在update或insert操作时，程序挂了。锁就解不了了，导致该用户在缓存时间内无法注册，除非手工清理。
所以redis锁只能减轻mongodb + node并发无事务的状况，而不能根本解决问题。

-------

## mongodb自带的原子操作
虽然mongodb没有事务处理，但是它自带一些原子性的复合操作。在某些特定场景下使用能达到事务的效果，当然缺陷就是只能是特定的某些场景。

1. `db.findAndModify.find()` 先查找后更新操作
    
2. `update with upsert`  先查找后插入操作    

-----
更新中....
