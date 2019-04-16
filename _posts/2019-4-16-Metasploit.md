---
layout: post
title:  "Metasploit"
date:   2019-4-16
categories: 
excerpt: 
---

* content
{:toc}


# **Metasploit使用**

## **简介**

以前听说过Metasploit，一直以为是一款渗透测试框架的工具，知道看了泉哥的《漏洞战争》才知道，可以通过Metasploit生成可触发漏洞的PoC样本，特别是对于一些经典漏洞。还可以用来自己编写Exp。

## **安装**

官网：*https://www.metasploit.com/*

安装：*https://www.fujieace.com/metasploit/windows.html*

需要配置用户环境

## **生成PoC样本**

1.  利用search搜索需要的漏洞相关利用代码

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g24exufmxfj211a0qujth.jpg>)

1.  利用use命令指定exploit，并用info命令生成关于此exploit的相关信息，包括参数值、漏洞描述、参考资料等信息。

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g24eznps2fj20qb0qk0vn.jpg>)

5.  在这里面选择想要的样本，如果是分析漏洞成因，只要选可触发崩溃的PoC岩本，直接通过set命令即可设置target参数值

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g24f2y3826j20uw0qkjv4.jpg>)

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g24f3jl1jyj20m001b3yf.jpg>)

6.  最后直接用exploit命令直接生成测试样本

![](<http://ww1.sinaimg.cn/large/7fb67c86ly1g24f4iw7bvj20kc02qq32.jpg>)

## **编写Exploit样本**

以后补充
