---
layout: post
title:  "mongodb拥有事务的解决方案（一）"
date:   2016-07-31
excerpt: "从各种层面的解决方案为mongodb添加事务性的处理。Redis内存锁、mongodb的原子操作"
feature: http://oboi2pfvn.bkt.clouddn.com/mongodb.jpg
tag:
- lessons 
- mongodb
- 事务
- 原子操作
comments: true
---

[mongodb拥有事务的解决方案（二）](http://zackku.com/mongo-transaction2/)  

最近因为node的并发优点加上mongodb的无事务性被坑了几把，所以找了一些解决方案。  

1. Redis内存锁（不可靠）
2. 用mongodb自带的原子操作 （使用范围局限）
3. 二段提交

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
总结就是，能用原子操作就用原子操作。

1. [db.module.findAndModify()](https://docs.mongodb.com/manual/reference/method/db.collection.findAndModify/) 先查找后更新操作      
    例如一个图书管理系统，查一本书是否可以借阅。那么首先需要找到这本书，然后判断书的借阅状态，若未必借阅，则把状态设置为已借阅，
    把书借阅出去。使用这个函数就可以达到先查找后更新的原子操作。      
        
        db.book.findAndModify({
            query:{ id: bookid, status: 'free'},
            update:{
                $set:{
                    status: 'busy'
                }
            }
        })
        
   [其他操作符](http://www.runoob.com/mongodb/mongodb-atomic-operations.html):    
   `$unset` (删除一个field)     
   `$inc` (对文档的某个值为数字型的field进行增减的操作)    
   `$push` (把value追加到field里面去，field一定要是数组类型才行，如果field不存在，会新增一个数组类型加进去)      
   `$pushAll` (同$push,只是一次可以追加多个值到一个数组字段内)     
   `$pull` (从数组field内删除一个等于value值)     
   `$addToSet` (增加一个值到数组内，而且只有当这个值不在数组内才增加)     
   `$pop` (删除数组的第一个或最后一个元素)     
   `$rename` (修改字段名称)     
   `$bit` (位操作，integer类型)
   
2. [update with upsert](https://docs.mongodb.com/manual/reference/method/db.collection.update/)  更新插入(upsert = update & insert)             
    对应上述场景一，比较适合用这种方法。更新一条记录，若记录不存在则创建。例如
            
            User.update({id : user_id }, param , { upsert : true })

    上面的例子，若查找不到id为user_id的文档，则插入数据为param的文档。这样可避免先查找后插入的并发问题。
    
3. [update/remove with $isolated](https://docs.mongodb.com/manual/reference/operator/update/isolated/) 文档隔离     
    
    >Prevents a write operation that affects multiple documents from yielding to other reads or writes once the first document is written. By using the $isolated option, you can ensure that no client sees the changes until the operation completes or errors out.
     
    >This behavior can significantly affect the concurrency of the system as the operation holds the write lock much longer than normal for storage engines that take a write lock (e.g. MMAPv1), or for document-level locking storage engine that normally do not take a write lock (e.g. WiredTiger), $isolated operator will make WiredTiger single-threaded for the duration of the operation.
    
    总结的说就是在update/remove操作的时候，把选中的文档进行加锁，直到操作完成或错误退出。在这加锁的过程中，不允许其他操作读取和写入这些文档。
    
            db.foo.update(
                { status : "A" , $isolated : 1 },
                { $inc : { count : 1 } },
                { multi: true }
            )

    在上面这个例子当中，如果没有`$isolated`，由于是多multi操作，若有其他操作对这些文档读取写入，会破坏文档的正确数据。
    
