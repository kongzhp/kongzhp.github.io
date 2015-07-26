---
layout:     post
title:      "Use Jenkins + Docker for Automatically test"
subtitle:   "Run multi jobs in a slave at the same time safely."
date:       2015-07-26 19:35:00
author:     "Pete Kong"
header-img: "img/docker-logo.jpg"
---

# Before

在使用docker之前，我们一般使用jenkins跑java项目的Unit Test(UT)、Component Test(CT)，为充分利用一台slave的资源，往往同一个slave会同时跑几个Job,那么如果跑UT/CT过程中，不同Job（project）的测试代码同时使用了同一个端口，那么就会因为端口冲突导致test case不稳定。

# Now

由于Docker容器是一个轻量级虚拟机，在容器内运行的应用就可以随心所欲地使用端口，而不影响其他容器中运行的应用。

# HowTo

1. 准备docker image

	我们使用Jenkins跑test case通常使用maven,因此我们一般只需要一个装有jdk和maven的linux image即可。Docker Hub上已经有装有maven的image,然而，由于万恶的GFW，下载是超级无敌慢，往往下一个ubuntu或centos的image，再下载jdk/maven，使用DOCKERFILE安装会比下一个完整带有maven的image快得多。

~~~
FROM centos:6

#install java
COPY jdk-7u79-linux-x64.rpm /root/
RUN rpm -ivh /root/jdk-7u79-linux-x64.rpm
RUN rm -f /root/jdk-7u79-linux-x64.rpm 

#install maven
COPY apache-maven-3.3.3-bin.tar.gz /usr/local/
RUN cd /usr/local ; tar xzvf apache-maven-3.3.3-bin.tar.gz; rm apache-maven-3.3.3-bin.tar.gz 
ENV M2_HOME=/usr/local/apache-maven-3.3.3
ENV MAVEN_OPTS="-Xms256m -Xmx512m"
ENV PATH=${M2_HOME}/bin:$PATH
ONTAINER_INIT /usr/local/bin/init-container
RUN echo '#!/usr/bin/env bash' > $CONTAINER_INIT ; chmod +x $CONTAINER_INIT
RUN echo 'mkdir -p /cache/m2 && ln -sf /cache/m2 ~/.m2' >> $CONTAINER_INIT

~~~

2. CI 脚本

3. 新建Jenkins Job
