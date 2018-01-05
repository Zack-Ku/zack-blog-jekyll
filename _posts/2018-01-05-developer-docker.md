---
layout: post
title:  "docker-开发人员必须要知道的docker"
date:   2018-01-05
excerpt: "随着微服务的发展，服务容器化已经是开发必须掌握的课题..."
feature: http://oboi2pfvn.bkt.clouddn.com/docker.png
tag:
- docker 
- Dockerfile
- docker-compose
comments: true
---

随着微服务的发展，服务容器化已经是开发必须掌握的课题。      
很多开发以为，项目部署，集群管理与自己无关，那是运维的事，但如今技术百花齐放，不同语言的项目有不同的部署方式，部署java需要懂jvm、tomcat、mvn，部署node项目要懂npm、v8等等。假若一个集群有多种技术栈的项目，那么运维即使不精通，也要基本了解全部的知识，否则服务器出问题，也不知道从哪里入手，再加上各种杂七杂八的项目配置，运维还要知道配置对应的业务内容，到最终只能增加人手去管理真正的运维工作（网络、ECS、负载、集群等等）。      
所以，如何切分开发和运维的工作，是很重要的课题。docker是一个很好的切入点，开发负责编码开发，然后把服务级别的docker给打包好，交付给集群使用，运维管理docker集群。当然docker还有非常多的好处，一次打包，到处运行；配置好一个集群，可以随意开多个独立环境，对于产品复制和测试调试都非常好用。      
下面一个项目中打包成一个docker服务的简单方案。

-----------

## docker基本操作
首先肯定要熟悉docker的基本操作和原理。对于这部分内容，网上有非常多教程，在此不再赘述。    
提供个参考资料
[Docker — 从入门到实践](https://www.gitbook.com/book/yeasy/docker_practice/details)

## Dockerfile
Dockerfile是用于构建镜像用的，指定某个镜像，执行一系列的操作，然后形成一个新的镜像。例如制作一个带node的centos镜像

	FROM centos:centos7
	MAINTAINER zack@xxxx.com

	RUN yum -y update; yum clean all
	RUN yum -y install epel-release; yum clean all
	RUN yum -y install nodejs npm; yum clean all

具体操作，参考上一节中的资料。


## 基础运行环境镜像构建
docker的内部是一种层级结构，上层是依赖下层构建的。例如一个安装了node的ubuntu，上层是node，下层是ubuntu，整体加起来就是带着node环境的ubuntu。在镜像制作后上传中，如果在仓库里面，底层已经存在的话，只会上传上层，但如果中间一层更改了，即上层依赖的层变更了，中间那层开始都要重新制作上传。

	b56f2cfbd344: Pushed 
	2c3396684cbe: Pushing[================>   ]  20.58MB/63.15MB
	58eb630a9b0e: Layer already exists 
	69b918931b8c: Layer already exists 
	8a61997aeb56: Layer already exists 
	1a4a50f5502f: Layer already exists 
	3f5bd7889e12: Layer already exists 
	25baa3ba1903: Layer already exists 
	5b1e27e74327: Layer already exists 
	04a094fe844e: Layer already exists 

所以，构建镜像也是门学问，要减少每次构建的执行时间，高效完成。一个原则就是尽量保证底层是不变动的。实践来看，很容易就能分出两层，运行环境为底层、项目服务。     
**运行环境**，项目所依赖的基础运行环境，例如jdk、tomcat、node、npm等等。这层一般不需要变动，构建一次即可，并且可以供同类项目使用。    
**项目服务**，业务层面可对外提供服务的层级。一般把打包好的war包、可执行的项目放这里，启动该镜像即可提供服务。     

基础镜像一般不以ubunut或centos为基础构建，这些版本docker镜像至少要几百M，构建出来体积太大，而且很多功能不需要，而且很占空间和带宽。所以大多基础镜像的构建是基于alpine构建的，一般才几十M。apline最linux可运行最简单的系统，或后续需要一些系统工具例如curl、wget等都需要额外安装。

简单的镜像可以在
[docker-library](https://github.com/docker-library) 找到对应的官方Dockerfile，然后再此基础上修改。   
下面以构建java web项目为例，构建一个jdk、tomcat的基础环境镜像。

	FROM openjdk:8-jre-alpine

	ENV CATALINA_HOME /usr/local/tomcat
	ENV PATH $CATALINA_HOME/bin:$PATH
	RUN mkdir -p "$CATALINA_HOME"
	WORKDIR $CATALINA_HOME

	# let "Tomcat Native" live somewhere isolated
	ENV TOMCAT_NATIVE_LIBDIR $CATALINA_HOME/native-jni-lib
	ENV LD_LIBRARY_PATH ${LD_LIBRARY_PATH:+$LD_LIBRARY_PATH:}$TOMCAT_NATIVE_LIBDIR

	RUN apk add --no-cache gnupg

	# see https://www.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/KEYS
	# see also "update.sh" (https://github.com/docker-library/tomcat/blob/master/update.sh)
	ENV GPG_KEYS 05AB33110949707C93A279E3D3EFE6B686867BA6 07E48665A34DCAFAE522E5E6266191C37C037D42 47309207D818FFD8DCD3F83F1931D684307A10A5 541FBE7D8F78B25E055DDEE13C370389288584E7 61B832AC2F1C5A90F0F9B00A1C506407564C17A3 713DA88BE50911535FE716F5208B0AB1D63011C7 79F7026C690BAA50B92CD8B66A3AD3F4F22C4FED 9BA44C2621385CB966EBA586F72C284D731FABEE A27677289986DB50844682F8ACB77FC2E86E29AC A9C5DF4D22E99998D9875A5110C01C5A2F6059E7 DCFD35E0BF8CA7344752DE8B6FB21E8933C60243 F3A04C595DB5B6A5F1ECA43E3B7BBB100D811BBE F7DA48BB64BCB84ECBA7EE6935CD23C10D498E23
	RUN set -ex; \
		for key in $GPG_KEYS; do \
			gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
		done

	ENV TOMCAT_MAJOR 8
	ENV TOMCAT_VERSION 8.0.48
	ENV TOMCAT_SHA1 d2446c127c9b11f88def11e542af98998071d91d

	ENV TOMCAT_TGZ_URLS \
	# https://issues.apache.org/jira/browse/INFRA-8753?focusedCommentId=14735394#comment-14735394
	https://www.apache.org/dyn/closer.cgi?action=download&filename=tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz \
	# if the version is outdated, we might have to pull from the dist/archive :/
	https://www-us.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz \
	https://www.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz \
	https://archive.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz

	ENV TOMCAT_ASC_URLS \
	https://www.apache.org/dyn/closer.cgi?action=download&filename=tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz.asc \
	# not all the mirrors actually carry the .asc files :'(
	https://www-us.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz.asc \
	https://www.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz.asc \
	https://archive.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz.asc

	RUN set -eux; \
		\
		apk add --no-cache --virtual .fetch-deps \
			ca-certificates \
			openssl \
		; \
		\
		success=; \
		for url in $TOMCAT_TGZ_URLS; do \
			if wget -O tomcat.tar.gz "$url"; then \
				success=1; \
				break; \
			fi; \
		done; \
		[ -n "$success" ]; \
		\
		echo "$TOMCAT_SHA1 *tomcat.tar.gz" | sha1sum -c -; \
		\
		success=; \
		for url in $TOMCAT_ASC_URLS; do \
			if wget -O tomcat.tar.gz.asc "$url"; then \
				success=1; \
				break; \
			fi; \
		done; \
		[ -n "$success" ]; \
		\
	gpg --batch --verify tomcat.tar.gz.asc tomcat.tar.gz; \
	tar -xvf tomcat.tar.gz --strip-components=1; \
	rm bin/*.bat; \
	rm tomcat.tar.gz*; \
	\
	nativeBuildDir="$(mktemp -d)"; \
	tar -xvf bin/tomcat-native.tar.gz -C "$nativeBuildDir" --strip-components=1; \
	apk add --no-cache --virtual .native-build-deps \
		apr-dev \
		coreutils \
		dpkg-dev dpkg \
		gcc \
		libc-dev \
		make \
		"openjdk${JAVA_VERSION%%[-~bu]*}"="$JAVA_ALPINE_VERSION" \
		openssl-dev \
	; \
	( \
		export CATALINA_HOME="$PWD"; \
		cd "$nativeBuildDir/native"; \
		gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
		./configure \
			--build="$gnuArch" \
			--libdir="$TOMCAT_NATIVE_LIBDIR" \
			--prefix="$CATALINA_HOME" \
			--with-apr="$(which apr-1-config)" \
			--with-java-home="$(docker-java-home)" \
			--with-ssl=yes; \
		make -j "$(nproc)"; \
		make install; \
	); \
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive "$TOMCAT_NATIVE_LIBDIR" \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --virtual .tomcat-native-rundeps $runDeps; \
	apk del .fetch-deps .native-build-deps; \
	rm -rf "$nativeBuildDir"; \
	rm bin/tomcat-native.tar.gz; \
	\
	# sh removes env vars it doesn't support (ones with periods)
	# https://github.com/docker-library/tomcat/issues/77
	apk add --no-cache bash; \
	find ./bin/ -name '*.sh' -exec sed -ri 's|^#!/bin/sh$|#!/usr/bin/env bash|' '{}' +

	# verify Tomcat Native is working properly
	RUN set -e \
	&& nativeLines="$(catalina.sh configtest 2>&1)" \
	&& nativeLines="$(echo "$nativeLines" | grep 'Apache Tomcat Native')" \
	&& nativeLines="$(echo "$nativeLines" | sort -u)" \
	&& if ! echo "$nativeLines" | grep 'INFO: Loaded APR based Apache Tomcat Native library' >&2; then \
		echo >&2 "$nativeLines"; \
		exit 1; \
	fi

	# install maven
	#RUN wget http://apache-mirror.rbc.ru/pub/apache/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz
	#RUN tar xzvf apache-maven-3.3.9-bin.tar.gz
	#RUN cp -R apache-maven-3.3.9 /usr/local/bin
	#RUN export PATH=apache-maven-3.3.9/bin:$PATH
	#RUN export PATH=/usr/local/bin/apache-maven-3.3.9/bin:$PATH
	#RUN ln -s /usr/local/bin/apache-maven-3.3.9/bin/mvn /usr/local/bin/mvn
	#RUN ls -l /usr/local/bin
	#RUN echo $PATH

	#EXPOSE 8080
	#CMD ["catalina.sh", "run"]

这里面不构建maven环境。因为maven是用于项目的构建，而不是用于执行的。项目的构建应该是构建的机器上做，以减少docker的打包成本。同时也不需要在基础镜像中指定暴露端口和启动命令，这部分应在项目镜像构建里面做。
## 项目执行镜像构建
有了基础的镜像，在此层做的东西就非常简单了，只要把项目运行的东西往docker里面塞，让docker里面的服务跑起来就行了。例如java web项目，需要将war包放到docker里面的的tomcat目录，暴露对应端口，指定镜像启动命令。

	FROM registry.xxxx/xxxx:latest

	ENV CATALINA_HOME /usr/local/tomcat
	ENV MODEL xxx
	#ENV APP /app

	WORKDIR $CATALINA_HOME/webapps

	COPY ${MODEL}/target/*war .

	EXPOSE 8080
	CMD ["catalina.sh", "run"]

到此镜像可执行的镜像已经构建完了。细心的同学可能发现了，到目前为止都没有配置过项目的启动参数。是的，镜像本身不应该包含项目的环境变量参数等，它应该在容器启动时候指定输入。否则每次变更配置需要重新构建，重新打包上传，不利于镜像扩展使用。在后面会介绍在启动时添加环境变量的方案。

## Make批处理
构建一个镜像，经常会包含多个命令，并且很多时候需要切换执行目录或复制移动文件。所以日常开发用make来配合镜像的构建上传下载打tag是非常有必要的，特别是要制作不同产品，不同环境的镜像。
	
	base:
		echo building ${NAME}-base:${TAG}
		cp ${model}/docker/base/Dockerfile .
		docker build -t ${REGISTRY}/${NAME}-base:${TAG} .
		docker tag ${REGISTRY}/${NAME}-base:${TAG} ${REGISTRY}/${NAME}-base:latest
		rm Dockerfile
		docker push ${REGISTRY}/${NAME}-base:${TAG}
		docker push ${REGISTRY}/${NAME}-base:latest
		
	war:
   		# TODO 加MD5校验，代码没改，不用执行
		echo package war ${model}
		mvn -pl ${model} -am -Dmaven.test.skip=true -Denv=xxx install

	local:war
		echo building ${NAME}:${TAG}
		cp ${model}/docker/local/Dockerfile .
		docker build -t ${REGISTRY}/${NAME}-${model}:latest .
		#docker build -t ${REGISTRY}/${NAME}-${model}:${TAG} .
		#docker tag ${REGISTRY}/${NAME}-${model}:${TAG} ${REGISTRY}/${NAME}-${model}:${FIXTAG}
		rm Dockerfile
		#docker push ${REGISTRY}/${NAME}-${model}:${TAG}
		#docker push ${REGISTRY}/${NAME}-${model}:${FIXTAG}
		docker push ${REGISTRY}/${NAME}-${model}:latest

## 一键拉起一套栈
很多时候，一个web项目需要很多依赖的服务，例如db，redis，zookeeper等等。如果想一套拉起多个容器服务，可以用docker-compose。很多人搞不清Dockerfile和docker-compose的区别，前者是构建镜像用的，一个是创建容器，即就是镜像与容器的关系，所以到底什么操作在Dockerfile里面做，什么操作在compose里面做，搞清楚这个关系就很容易区分了。     
很多集群管理工具，像rancher、k8s都支持用docker-compose导出一套栈进来管理。

以下是一个mysql、zookeeper、memcached及web服务自身的例子。执行docker-compose up即可一套拉起。
	
	version: "2"
	services:
		mysql:
			image:mysql
		zookeeper:
			image: zookeeper
		memcached:
			image: memcached
		app:
			image: registry.xxxx/xxxxx:latest
			environment:
				- JAVA_OPTS=-server -Xms512m -Xmx512m -Xss256K -Duser.timezone=GMT+08 
			ports:
				- "8080:8080"
			depends_on:
				- memcached
				- zookeeper
				- mysql
			links:
				- memcached
				- zookeeper
				- mysql
    		
  
