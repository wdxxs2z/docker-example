# mesos-apps
这个不算是个项目，算是一个对之前docker使用的总结：只不过这次加了Jenkis进去

### 目标是基于mesos+marathon之上加入Jenkis使代码能够通过marathon的API直接部署到docker中。

* 安装mesos,marathon,zookeeper,docker组件
* 安装Jenkis
* 写一个Dockerfile
* 写一个部署脚本
* 执行Jenkis部署任务

#### 安装mesos等，这里只说启动
1.Zookeeper,不说了。</br>

2.Mesos Master启动:</br>
mesos-master --work_dir=/var/lib/mesos --zk=zk://192.168.172.150:2181/mesos --quorum=1</br>

3.Mesos Slaver启动:</br>
mesos-slave --master=zk://192.168.172.150:2181/mesos --containerizers=docker,mesos --executor_registration_timeout=5mins</br>

4.Marathon启动:</br>
/home/jojo/marathon-0.9.0/bin/start --master zk://192.168.172.150:2181/mesos --zk zk://192.168.172.150:2181/marathon</br>

#### 安装Jenkis
1.这里只说一下插件：SCM Sync Configuration Plugin ，GitHub plugin ，GIT plugin ，GIT client plugin</br>
如果想尝鲜docker,目前也有很多直接部署docker的插件，但这次我用不到

#### 写一个Dockerfile
https://github.com/wdxxs2z/selfdocker-store/blob/master/Dockerfile</br>
很简单，就是在tomcat的webapps里增加我们的war包

		#deploy our war
		ADD java-hello-world.war /tomcat/webapps/hello.war
		
写完后本地测试，这里要说明的是，在docker run -d wdxxs2z/helloworld:1.0后，最好是有自己的私有registory,否则后面发布到marathon上会有问题。</br>

		docker login
		docker commit containerID wdxxs2z/helloworld:1.0
		docker push

#### 写一个部署脚本

		#!/bin/sh
		set +e

		#这句仅仅是为了测试
		/usr/bin/docker build -t wdxxs2z/helloworld:1.0 /home/jojo/.jenkins/workspace/helloworld | tee /home/jojo/.jenkins/workspace/helloworld/Docker_build_result.log
		RESULT=$(cat /home/jojo/.jenkins/workspace/helloworld/Docker_build_result.log | tail -n 1)
		  
		echo 'push our app to marathon......'
		
		curl -l -H "Content-type: application/json" -X POST -d '{
		"id": "tomcat-helloworld",
		"cpus": 0.5,
		"instances": 1,
		"mem": 400,
		"ports": [8080],
		"container":
		 {
			"type": "DOCKER",
			"docker":{
				"image": "wdxxs2z/helloworld:1.0",
				"network": "BRIDGE",
				"privileged": false,
				"forcePullImage": true,
				"portMappings": [
					{
						 "containerPort": 8080,
						 "hostPort": 0,
						 "servicePort": 8080,
						 "protocol": "tcp"
					}
				]
			 }
		  },
		 "healthChecks": [
			 {
					"protocol": "TCP",
					"gracePeriodSeconds": 3,
					"intervalSeconds": 5,
					"portIndex": 0,
					"timeoutSeconds": 5,
					"maxConsecutiveFailures": 3
				}
		  ]
		}' http://192.168.172.150:8080/v2/apps
		
#### 执行Jenkis部署任务
上面的脚本可以健壮化，通过api获取相关的app，具体怎么玩就由需求决定了。

		Sending build context to Docker daemon 
		Step 0 : FROM tifayuki/java:7
		 ---> 2128079f9b8f
		Step 1 : MAINTAINER Feng Honglin <hfeng@tutum.co>
		 ---> Using cache
		 ---> 7db202c95b00
		Step 2 : RUN apt-get update &&     apt-get install -yq --no-install-recommends wget pwgen ca-certificates &&     apt-get clean &&     rm -rf /var/lib/apt/lists/*
		 ---> Using cache
		 ---> 610fcdedec5a
		Step 3 : ENV TOMCAT_MAJOR_VERSION 7
		 ---> Using cache
		 ---> 7dc387eb1316
		Step 4 : ENV TOMCAT_MINOR_VERSION 7.0.55
		 ---> Using cache
		 ---> f60cee415e16
		Step 5 : ENV CATALINA_HOME /tomcat
		 ---> Using cache
		 ---> 8f445bd4fd2b
		Step 6 : RUN wget -q https://archive.apache.org/dist/tomcat/tomcat-${TOMCAT_MAJOR_VERSION}/v${TOMCAT_MINOR_VERSION}/bin/apache-tomcat-${TOMCAT_MINOR_VERSION}.tar.gz &&     wget -qO- https://archive.apache.org/dist/tomcat/tomcat-${TOMCAT_MAJOR_VERSION}/v${TOMCAT_MINOR_VERSION}/bin/apache-tomcat-${TOMCAT_MINOR_VERSION}.tar.gz.md5 | md5sum -c - &&     tar zxf apache-tomcat-*.tar.gz &&     rm apache-tomcat-*.tar.gz &&     mv apache-tomcat* tomcat
		 ---> Using cache
		 ---> 0f8362de35a3
		Step 7 : ADD create_tomcat_admin_user.sh /create_tomcat_admin_user.sh
		 ---> Using cache
		 ---> 502310473fd5
		Step 8 : ADD run.sh /run.sh
		 ---> Using cache
		 ---> 93ec5d720dcb
		Step 9 : RUN chmod +x /*.sh
		 ---> Using cache
		 ---> b114a536832b
		Step 10 : ADD java-hello-world.war /tomcat/webapps/hello.war
		 ---> Using cache
		 ---> a0e519478022
		Step 11 : EXPOSE 8080
		 ---> Using cache
		 ---> 2f45a367a480
		Step 12 : CMD /run.sh
		 ---> Using cache
		 ---> 092f926c10d0
		Successfully built 092f926c10d0
		
上面是个docker app例子只做构建，不启动，然后只在marathon中启动运行，后面可以加如路由发现功能，这样就灵活了。
