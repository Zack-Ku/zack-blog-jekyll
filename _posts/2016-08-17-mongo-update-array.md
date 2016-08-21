---
layout: post
title:  "MongoDB更新数组中某个对象的元素"
date:   2016-08-20
excerpt: "解决复杂的文档更新数组问题"
feature: http://oboi2pfvn.bkt.clouddn.com/mongodb.jpg
tag:
- lessons 
- mongodb
comments: true
---

经常一些场景当中，需要更新数组中某个对象的元素。用$操作符即可完成这操作。可以把$理解为数组下标

-----------

## 例子
一个包含数组sign表文档

            {
                user_id:'abc'
                signDate:[
                    {
                        dateTime:'2016-8-15'
                        isSign:1
                    },
                    {
                        dateTime:'2016-8-16'
                        isSign:0
                    }
                ]
            }

现在想把8-16那记录的isSign改成1。除了把整个数组拿出来修改再整个update回去外。
就是用$去更新，搭配$elemMatch

          db.sign.update(
              {
                user_id: 'abc' ,
                signDate:{$elemMatch:{dateTime:'2016-8-16'}} 
              },  //查找对应记录，精确到哪些数组
              {$set: {'signDate.$.isSign':1}})   //更新找到的记录,$代表获取的下标
              
这个方法可以更新多条记录，不过最后要加上`{multi:true}`。             

如果数组不是一个对象，可以不用$elemMatch。而是直接当成一个属性去查找。

            {
                user_id: 'aaa'
                score:[90,88]
            }
     
把上面记录score成绩小于90的记录改为100

            db.score.update(
                {user:id:'aaa' , score:{$lt:90}},   //直接查也行, sscore:88
                {$set:{'score.$':100}}
            )
