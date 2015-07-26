---
layout:     post
title:      "Use Jenkins + Docker for Automatically test"
subtitle:   "Run multi jobs in a slave at the same time safely."
date:       2015-07-26 19:35:00
author:     "Pete Kong"
header-img: "img/docker-logo.jpg"
---

# Before

在使用docker之前，我们一般使用jenkins跑UT、CT，为充分利用一台slave的资源，往往同一个slave会同时跑几个Job,那么如果跑UT/CT过程中，不同Job（roject）的测试代码同时使用了同一个端口，那么就会因为端口冲突导致test case不稳定。

# Now

由于Docker容器是一个轻量级虚拟机，在容器内运行的应用就可以随心所欲地使用端口，而不影响其他容器中运行的应用。

# HowTo

1. 准备docker image

2. CI 脚本

3. 新建Jenkins Job