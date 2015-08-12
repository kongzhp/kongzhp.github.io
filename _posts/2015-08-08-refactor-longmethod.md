---
layout:     post
title:      "《重构-改善既有代码的设计》读书笔记（2）"
subtitle:   "Long methods "
date:       2015-08-08 17:35:00
author:     "Pete Kong"
header-img: "img/post-bg-02.jpg"
---

# Introduction

Long Methods过长函数是典型的一种代码的“坏味道”，因为函数越长越难理解，而小函数则带来很多便利：

* 解释能力: 良好的函数命名可以很好解释代码的意图，而不需要添加过多的注释说明。

* 共享能力: 一个良好封装的子函数可被多个不同地点调用，从而减少重复代码

* 选择能力：抽出来的小函数可接收多态消息，从而封装条件逻辑，通过把条件逻辑转化成消息形式，即函数参数，降低代码的重复，增加清晰度和提高弹性。



# How To

要把函数变小，大多数情况使用Extract Mehthod即可以。Extract Method，从长函数中提取小粒度的函数十分简单，只要把小函数取一个清晰有意义的名称即可，这里不再赘述。但是抽取的函数可能带来很多的参数和临时变量作为参数，传递给新增的函数，这样可读性几乎没有任何提升。这时可以使用Replace Temp with Query来消除临时元素。还有Introduce Parameter Object和Preserve Whole Object来消除过长的参数列。

## Replace Temp with Query

临时变量的缺点在于他们是暂时的，只在函数体里可见，为了使用这个变量，则驱使你写更长的函数。所以如果在一个长函数中，有一个临时变量在很多地方使用，则把临时变量替换为一个查询，那么同一个类中的所有方法都能获得这份信息。


## Introduce Parameter Object