---
layout: post
title:  "ES6-Number.EPSILON 用法"
date:   2016-08-20
excerpt: "处理js浮点数问题"
feature: http://oboi2pfvn.bkt.clouddn.com/es6.jpg
tag:
- ES6 
- EPSILON
comments: true
---

用 ES6 中，Number的常量扩展 ，以校验javascript的浮点数运算问题

-----------

ES6在Number对象上面，新增一个极小的常量 **Number.EPSILON**。

            Number.EPSILON
            // 2.220446049250313e-16
            Number.EPSILON.toFixed(20)
            // '0.00000000000000022204'

引入一个这么小的量的目的，在于为浮点数计算，设置一个误差范围。我们知道浮点数计算是不精确的。

          0.1 + 0.2
          // 0.30000000000000004
          
          0.1 + 0.2 - 0.3
          // 5.551115123125783e-17
          
          5.551115123125783e-17.toFixed(20)
          // '0.00000000000000005551'
              
但是如果这个误差能够小于 **Number.EPSILON** ，我们就可以认为得到了正确结果。            

            5.551115123125783e-17 < Number.EPSILON
            // true
     
因此，Number.EPSILON的实质是一个可以接受的误差范围。

           function withinErrorMargin (left, right) {
             return Math.abs(left - right) < Number.EPSILON;
           }
           withinErrorMargin(0.1 + 0.2, 0.3)
           // true
           withinErrorMargin(0.2 + 0.2, 0.3)
           // false
           
上面的代码为浮点数运算，部署了一个误差检查函数。
