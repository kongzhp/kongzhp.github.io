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

	我们使用Jenkins跑test case通常使用maven,因此我们一般只需要一个装有jdk和maven的linux image即可。Docker Hub上已经有装有maven的image,然而，由于万恶的GFW，下载是超级无敌慢，往往下一个ubuntu或centos的image，再下载jdk/maven，使用Dockerfile安装会比下一个完整带有maven的image快得多。

	Dockerfile

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

	把上述Dockerfile与jdk-7u79-linux-x64.rpm/apache-maven-3.3.3-bin.tar.gz放在同一新建的空目录下(例如：~/ciimage)，执行以下命令：

		cd ~/ciimage
		docker build -t centos/java7_maven .

2. CI 脚本

	CI脚本参考了https://github.com/kabisa/jenkins-docker 上的run-containerized,它要求Job定义Dockerfile文件，此文件可作为扩展，使得不同的项目可以先运行指定脚本构建一个新的Image，再在此image执行maven命令跑test case。然而如果不需要预先在跑case前在容器中执行某些命令，则没必要重新build一个image。另外，假如有case fail，脚本返回非0值使得Job变红，而我们只需要Job变黄，因此本文对run-centainerized脚本进行了一些修改和简化。

	run-containerized:

		#!/usr/bin/env bash
		#
		# Usage:
		# run-containerized [CI command]
		#
		# Examples:
		# run-containerized 'bin/ci'
		# run-containerized 'mvn clean install'
		set -o pipefail
		trap cleanupContainer EXIT

		function cleanupContainer() {
		  if [ -n "$CID" ]; then
		    echo "Cleaning up docker container"
		    docker kill $CID > /dev/null
		  fi
		}

		function abort() {
		  echo "$(tput setaf 1)$1$(tput sgr 0)"
		  exit 1
		}

		# Where to store cachable artifacts. Docker containers ran by this script get
		# a /cache directory that's unique to the job and can be used to store any build
		# artifacts that should persist across builds. Think about Ruby gems, the local
		# Java Maven repository etc.
		CACHE_DIR=${CACHE_DIR:-/ci/cache}
		BUILD_CACHE_DIR="$CACHE_DIR/$JOB_NAME"

		# Deduce a tag name for the container we're going to build based on the Jenkins job name
		DOCKER_TAG_NAME=$(echo $JOB_NAME | tr '[:upper:]' '[:lower:]' | sed 's/[^-a-z0-9_.]//g')

		DOCKER_IMG=${DOCKER_IMG:-centos/java7_maven}

		# Now actually run the Docker container.
		# We mount the jobs workspace under /workspace and a directory to store build
		# caches/artifacts under /cache.
		#
		# Additional Docker CLI options can be passed in by setting a DOCKER_OPTS environment
		# variable.
		#
		# If the container contains a /usr/local/bin/init-container script we execute this first,
		# before executing the actual CI command. Use this to start services like Postgres, Redis etc,
		# or to run other dynamic configuration of the build environment.
		#
		# After the CI build has finished we chown the workspace to user executing this script.
		# In practise this means the workspace is properly owned by the user running Jenkins.
		#
		# Note that containers are run detached because due to how the Docker CLI works it's
		# not trivial to cleanup the Docker container when the Jenkins job get's aborted if
		# containers are run in the foreground.
		set -x
		CID=$(docker run -it -d -v $WORKSPACE:/workspace \
		                        -v $BUILD_CACHE_DIR:/root/.m2 \
		                        -w /workspace \
		                        -e 'CI=true' \
		                        -e 'DISPLAY=:0' \
		                        $DOCKER_OPTS $DOCKER_IMG \
		                        $@ ;\
		                        [ \$? -eq 1 ] && echo '[Error] There are test case failed' ;\
								chown -R $(id -u `whoami`) /workspace ;\
		                        (exit 0)")
		set +x

		# This is used for debugging. If you run this script locally while setting up or
		# debugging your CI build environment you could pass 'bash' as CI command and get
		# an interactive environment.
		[[ "$@" = "bash" ]] && docker attach $CID

		# Tail the logs of the container so we can see the build output
		docker logs -f $CID &

		# Wait for the container to exit when the CI command is done.
		exit $(docker wait $CID)

3. 新建Jenkins Job
	
	3.1 新建一个"自由风格的软件项目"
	3.2 配置好CMS
	3.3 新增构建步骤：Execute shell,调用run-containerized执行进行maven test
	![run-containerized]({{ site.baseurl }}/img/docker_jenkins/build_shell.png)
	3.4 新增构建后操作：Publish JUnit test result report, 从而把失败的test case归档。
	![JUnit_report]({{ site.baseurl }}/img/docker_jenkins/junit_report.png)
