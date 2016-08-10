---
layout: post
title:  "基于jekyll和github pages博客搭建全过程(一)"
date:   2016-07-09
excerpt: "从0开始，搭建一个拥有私人域名的专属geek博客。博客的初步搭建。"
feature: http://oboi2pfvn.bkt.clouddn.com/blz.jpg
tag:
- lessons 
- blog
- jekyll
- github pages
comments: true
---

[基于jekyll和github pages博客搭建全过程(二)](http://zackku.com/built-blog-2)  
 
 
[基于jekyll和github pages博客搭建全过程(三)](http://zackku.com/built-blog-3)

#### 为什么要搭建这样的一个博客
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一直以来，所有信息的记录几乎都是用印象笔记的，包括工作的note、日常随手、一些学习记录。
这样导致一个问题，没有一个很好的地方让自己去写一些总结，自己的心得。不是说印象不可以做，
只是那里没有所谓的组织感，大部分都是一些摘录，copy。写blog，维护blog让自己更容易有成就感，
因为它是公开的，别人可以通过blog了解你更多的信息，而因为如此你更加有动力去更新更好的文章，
弄更好看的界面。其次，写博客是一种很好的消遣，让你学习或工作累了，写写文章，
对之前某些事或知识点做回顾和总结。这样比浪费时间好太多。长期下去，积累了许多总结过的精华，
当你回顾的时候，你会发现,这，是你一生的财产。

-------------

## 搭建条件
像搭建这样的一个基于github pages和jekyll的博客，并不是任何人都可以轻松完成的。  
至少具备以下条件(可以通过简单学习达到):  

- git的基本使用，用于代码的上传下载
- github的基本使用，用于部署静态页面的blog
- Markdown的基本语法，用于编写格式可控的博客

如果想要更加个性高效的维护开发，需要以下条件(可选):  

- 简单前端知识，html，css等
- jekyll的本地安装，用于本地运行博客
- 域名购买
- DNS配置

----------

## 主要原理
github有个github pages功能，把代码的静态页面发布到`username.github.io`上,
username是你github的用户名，像本博客之前的域名就是`zack-ku.github.io`。
其实github pages就是一个jekyll的服务，我们的代码要基于jekyll，才能成功发布。

----------

## 搭建jekyll博客
这个环节对于刚接触的人来说感觉好像很难，其实我们并不用真正完整去学习jekyll才能搭建blog，
我们主要的目的是写blog。  

#### 最简单的方法
[github pages的主页](https://pages.github.com) 里面就介绍了最简单最快速，
小白都会的搭建个人博客的方法(里面连git的命令都一步步告诉你)，
就是用github项目设置中的Automatic page generator。它会自动把你的代码清空掉，
重构成基于jekyll的项目，并且你可以自由选择博客的主题风格，连一点代码都不需要写。

![github setting](http://oboi2pfvn.bkt.clouddn.com/githubpages.png)

#### 定制的方法
<http://jekyllthemes.io>   
  
<http://jekyllthemes.org>     
上面有直接下载的jekyll模板，不过每个模板的功能都不一样，你要想很好的使用它，
你需要一点代码的功底和前端的知识才能比较轻松的定制出自己的blog。
另外你可以上github去搜索多人关注的jekyll项目，fork一个。

------------------------------------

## 博客编写
一般jekyll的文件目录结构如下
![folder of jekyll](http://oboi2pfvn.bkt.clouddn.com/folder-of-jekyll.png)
虽然不同模板可能不尽相同，但是一定会有 **_posts** 这个文件夹，
而这个文件夹正是编写博客的地方，如果不关注网站的其他方面，那只管在这文件夹里面写博客就好了。
在这个文件夹里面新增一个.md格式的文件，文件名格式一般需要以日期开头，标题结尾，如  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`2016-07-09-how-to-build-a-blog.md`  
注意文件名不能有空格，否则在用url读取时会出现问题。
至于博客具体编写是需要用到**Markdown**的。这语言非常容易学，不用5分钟就可以学会使用了。
具体的基础语法可见本博客 [Markdown基础语法](zackku.com/Markdown/ "点击打开")

------------------------------------
到这为止，你已经拥有一个专属的个人博客，并且可以在上面自由的撰写美文了。  
具体的关于jekyll的深入，和域名、DNS配置，请留意后续博文。  

**参考 :**  


<https://pages.github.com>  github pages主页

<http://jekyllcn.com/>  jekyll中文主页
