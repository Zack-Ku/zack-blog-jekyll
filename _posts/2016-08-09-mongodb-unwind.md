---
layout: post
title:  "MongoDB之$unwind操作符"
date:   2016-08-09
excerpt: "把文档中的数组字段拆散，分离出多个文档"
feature: http://oboi2pfvn.bkt.clouddn.com/mongodb.jpg
tag:
- lessons 
- mongodb
- unwind
comments: true
---

在aggregate中，常常会遇到一些字段属性是数组对象，然后又需要对这些数组对象进行统计。
这时候就需要用到$unwind操作符。这是一个常用的，又容易被忽略的一个操作。

-----------

## 定义

- field 版
    
           { $unwind: <field path> }
           
- document版

            {
              $unwind:
                {
                  path: <field path>,
                  includeArrayIndex: <string>,
                  preserveNullAndEmptyArrays: <boolean>
                }
            }
            
1. \<field path\> 你要打散的字段 
2. includeArrayIndex，分配一个存该数组索引的字段 
3. preserveNullAndEmptyArrays，是否输出空内容。

-----------

## 场景
一个用户表user，其中一个字段是一个数组对象，存的是用户的奖励信息。
这时需要统计用户A所有奖励类型为b的总额。

        {
            user_id:A_id ,
            bonus:[
                { type:a ,amount:1000 },
                { type:b ,amount:2000 },
                { type:b ,amount:3000 }
            ]
        }

**unwind操作：**

        db.user.aggregate([
            {$unwind:bonus}
        ])
        
        //结果
        {user_id : A_id , bonus:{type : a ,amount : 1000}}
        {user_id : A_id , bonus:{type : b ,amount : 2000}}
        {user_id : A_id , bonus:{type : b ,amount : 3000}}
       
**统计：**

        db.user.aggregate([
            {$match: {user_id : A_id} },
            {$unwind:bonus},
            {$match: {'bonus.type' : b} },
            {$group: {_id : '$user_id' , amount : {$sum : {'$bonus.amount'}} }}
        ])
        
        //结果
        {_id:A_id , amount : 5000}
       
        

**参考 :**  


<https://docs.mongodb.com/manual/reference/operator/aggregation/unwind/>  $unwind MongoDB

    
