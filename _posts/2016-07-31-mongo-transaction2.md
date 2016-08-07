---
layout: post
title:  "mongodb拥有事务的解决方案（二）"
date:   2016-07-31
excerpt: "从各种层面的解决方案为mongodb添加事务性的处理。二段提交"
feature: /assets/img/mongodb.jpg
tag:
- lessons 
- mongodb
- 事务
- 二段提交
comments: false
---

[mongodb拥有事务的解决方案（一）](http://zackku.com/mongo-transaction/)  

接上篇博文，本文将探索下mongodb的二段提交方案去解决事务问题。

-----------

##  背景
>在MongoDB中，操作单个文档（document）是原子性的；但是，涉及到多个文档的操作，也就是常说的“多文档事务”，是非原子性的。由于document可以设计的非常复杂，包含多个“内嵌的”文档，因此单个文档的原子性为很多实际场景提供了必要的支持。 
  尽管单文档原子操作很强大，但在很多场景下依然需要多文档事务。当执行一个由几个顺序操作组成的事务时，可能会出现某些问题，例如： 
 
>  - 原子性：如果某个操作失败了，在同一个事务中前面的操作回滚到最初的状态（即，要么全做，要么全部做）。 
>  - 一致性：如果发生了严重故障将事务中断（比如：网络、硬件故障），数据库必须能够恢复到一致的状态。
  
>在需要多文档事务的场景中，你可以实现两阶段提交来完成场景需求。两阶段提交可以保证数据的一致性，如果发生错误，可以恢复到事务开始之前的状态。在事务执行过程中，无论发生什么情况都可以还原到数据和状态的准备阶段。

自官方文档

-----------

## 二段的含义
两段指prepare阶段和commit阶段：      
![二段提交](/assets/img/two-phase-commit.png)
**Prepare**：TM(transaction manager)给每个参与者（resource manager）发送prepare信息。每个参与者要么直接返回失败，要么在本地执行事务（记录日志和rollback的信息），但不commit。         
**Commit**：如果TM收到了任一参与者的失败消息或超时，那么TM会发rollback给其他的参与者，参与者会执行rollback并在最后释放锁资源。否则则发送commit让所有参与者完成事务。
 这样能保证在事务提交前尽可能完成所有能完成的工作，最后的commit是一个耗时很短的操作，错误概率相对很低。相比单一阶段的commit，两段式更加可靠但是会消耗更多时间，所以会提高锁资源的冲突，加大了死锁的发生几率。

-----------

## MongoDB中的两段式提交
以下来自官方文档中的典型例子。从A账户资金转移到B账户。       
MongoDB中的一个accounts集合保存了账户的name和余额balance信息，以及当前事务的数组pendingTransactions。      

1. 首先创建两个账户：          
    
            db.accounts.save({name: "A", balance: 1000, pendingTransactions: []})
            db.accounts.save({name: "B", balance: 1000, pendingTransactions: []})

2. 然后创建**事务**集合transactions，来保存事务的信息——源账户source、目标账户dest、转移金额value、事务状态state。如果通过账户A转移一笔钱到账户B时，将通过下面的方式发起一个事务（state为initial）

           db.transactions.save({source: "A", destination: "B", value: 100, state: "initial"})

3. 找到transactions集合中对应initial状态文档，并改为pending状态
            
           t = db.transactions.findOne({state: "initial"})
           db.transactions.update({_id: t._id, state: "initial"}, {$set: {state: "pending"}})
           
4. 为每一个账户应用事务，并记录事务的_id。

           db.accounts.update(
              { name: t.source, pendingTransactions: { $ne: t._id } },
              { $inc: { balance: -t.value }, $push: { pendingTransactions: t._id } })
           db.accounts.update(
              { name: t.destination, pendingTransactions: { $ne: t._id } },
              { $inc: { balance: t.value }, $push: { pendingTransactions: t._id } })

5. 检查账户状态，是否应用成功
   
6. 将事务文档状态改为commited
            
            db.transactions.update({_id: t._id}, {$set: {state: "committed"}})

7. 然后更新accounts集合，相当于一个释放锁的过程。

            db.accounts.update({name: t.source}, {$pull: {pendingTransactions: t._id}})
            db.accounts.update({name: t.destination}, {$pull: {pendingTransactions: t._id}})
            
8. 将事务状态设置为done。

        db.transactions.update({_id: t._id}, {$set: {state: "done"}})


-----------

## 深入理解
对应二段的含义，该例子的        

- **2.**对应就是二段含义图中的预备。该过程会记录事务发生的对象，及将要发生的变化内容。在这操作当中，可以尽可能把信息存储起来，，以方便事务失败的回滚
- **3.4.5.**对应就是二段图中的就绪。即尝试完成该事务，把结果返回给事务管理器。
- **6.**对应则是提交。确认事务的结果。
- **7.**对应是已提交。告知事务管理器，把事务close掉。

再结合上篇博文讲到`$isolated`操作符，应该把以上的所有update操作的都加上，以避免多请求的并发操作。

-----------

## 事务回滚
当出现一些不可恢复的错误时（例如其中一个文档不存在、文档条件(balance)不满足等），就需要进行回滚操作。有两种可能的回滚方法：
1. 完成设置事务状态为done后，你需要完全提交这个事务，而不能进行回滚操作。可以创建一个新的事务来转换源和目标的文档。
2. 完成设置事务状态为pending和d.将事务状态设置为commited之间，你需要执行下面的步骤：
    
    //设置事务状态为canceling
    db.transactions.update({_id: t._id, state: "pending"}, {$set: {state: "canceling"}})   
    
    //回滚事务：执行一系列的相反的操作。
    db.accounts.update({name: t.source, pendingTransactions: t._id}, {$inc: {balance: t.value}, $pull: {pendingTransactions: t._id}})
    db.accounts.update({name: t.destination, pendingTransactions: t._id}, {$inc: {balance: -t.value}, $pull: {pendingTransactions: t._id}})
    db.accounts.find()
   
    //设置事务状态为canceled.
    db.transactions.update({_id: t._id}, {$set: {state: "canceled"}})



**参考 :**  


<https://docs.mongodb.com/manual/tutorial/perform-two-phase-commits/>  mongodb官方二段提交官方文档

    
