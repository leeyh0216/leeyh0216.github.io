---
layout: post
title:  "[Hadoop] Yarn Programming(1)"
date:   2017-03-12 01:39:00 +0900
author: leeyh0216
categories: dev hadoop yarn
---

> 이 문서는 Yarn Application 내부 동작에 대해 확인을 위하여 작성하였습니다. 
소스코드는 다음 Git Repository를 참고해주세요 : [leeyh0216:YarnJobExecutor](https://github.com/leeyh0216/YarnJobExecutor)

### Yarn Application 개발

#### build.gradle 설정

기본적으로 해당 프로젝트는 Hadoop 2.7.3 기준이며 Build Tool은 Gradle 2.14를 사용하도록 한다.
build.gradle 내에는 다음 5개의 Dependency를 설정한다.

- compile group: 'org.apache.hadoop', name: 'hadoop-yarn-common', version: '2.7.3'
- compile group: 'org.apache.hadoop', name: 'hadoop-yarn-client', version: '2.7.3'
- compile group: 'org.apache.hadoop', name: 'hadoop-yarn-api', version: '2.7.3'
- compile group: 'org.apache.hadoop', name: 'hadoop-common', version: '2.7.3'
- compile group: 'org.apache.hadoop', name: 'hadoop-hdfs', version: '2.7.3' 

추가적으로 Logging과 Guava Library를 사용하기 위하여 다음 Dependency도 추가한다.
- compile group: 'ch.qos.logback', name: 'logback-classic', version: '1.2.1'
- compile group: 'ch.qos.logback', name: 'logback-core', version: '1.2.1'  
- compile group: 'com.google.guava', name: 'guava', version: '21.0'

#### Yarn Client 초기화

Yarn에 Application 실행을 요청하기 위해서는 YarnClient 객체를 초기화해야 한다.
기존에 설치한 Hadoop의 설정파일이 있는 $HADOOP_HOME/etc/hadoop 경로를 확인하고 다음과 같이 작성하여 YarnClient를 초기화한다.
{% highlight java %}
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.yarn.client.api.YarnClient;
import org.apache.hadoop.yarn.client.api.YarnClientApplication;
import org.apache.hadoop.yarn.conf.YarnConfiguration;

//설치된 Yarn과 통신할 수 있는 YarnClient 객체를 생성한다.
YarnClient yarnClient = YarnClient.createYarnClient();
Configuration configuration = new YarnConfiguration();

//Configuration에 필요한 파일은 core-site.xml, hdfs-site.xml, mapred-site.xml, yarn-site.xml이다.
File hadoopConfDir = new File("/home/leeyh0216/hadoop_conf/");
File[] confFileList = hadoopConfDir.listFiles();

//각 파일들을 Configuration에 추가한다.
for(File f : confFileList)
    configuration.addResource(new Path(f.getAbsolutePath()));

yarnClient.init(configuration);
yarnClient.start();
{% endhighlight %}
위 과정에서 Configuration을 설정 파일로 초기화하지 않으면, Yarn의 ResourceManager 주소를 localhost에서 찾게 된다.
따라서 다른 서버에 Yarn이 설치되어 있는 경우 반드시 Hadoop Configuration File을 사용해서 Configuration 객체를 설정해야 한다.

#### Application Master를 생성

위의 YarnClient 초기화 과정에서 설치된 Yarn과 통신하기 위한 과정은 끝났다.
Yarn에서 동작하는 Application은 Application Master와 Container로 이루어져 있다.
Application Master는 Application을 실행하기 위한 정보를 Resource Manager에게 할당받고, 할당받은 자원을 통해 실제 작업을 수행하는 Container를 실행하는 주체이다.
따라서 YarnApplication의 실행을 위해서는 Application Master의 실행이 필요하다.
{% highlight java %}

import org.apache.hadoop.yarn.client.api.YarnClientApplication;
import org.apache.hadoop.yarn.api.protocolrecords.GetNewApplicationResponse;
import org.apache.hadoop.yarn.api.records.Resource;
import org.apache.hadoop.yarn.util.Records;

//Yarn에 새로운 Application의 생성을 요청하고, 반환된 GetNewApplicationResponse를 이용하여 Application 생성을 위한 정보(Application ID, 생성 가능한 최대 메모리, 코어 수 등)를 반환받는다.
// 이 과정에서는 아직 Application이 생성되지는 않은 상태이며, Yarn이 Client로부터 새로운 Application 생성을 요청받을 것이라는 것만 아는 상태이다.
YarnClientApplication app = yarnClient.createApplication();
GetNewApplicationResponse response = app.getNewApplicationResponse();

//Yarn에 설정된 정보들을 가져온다.
int yarnAMMaxMem = appResponse.getMaximumResourceCapability().getMemory();
int yarnAMMaxCore = appResponse.getMaximumResourceCapability().getVirtualCores();

//Application Master 실행 요청을 위한 Context를 생성한다.
ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
appContext.setApplicationName("Test Application");

//Application Master를 실행하기 위한 정보를 설정한다.
Resource appRes = Records.newRecord(Resource.class);
appRes.setMemory(50);
appRes.setVirtualCores(1);
appContext.setResource(appRes);
{% endhighlight %}
위의 ApplicationSubmissionConext는 Application Master의 실행 요청 위한 객체이다.
해당 객체에 Application Master를 실행하기 위한 정보를 넣어 YarnClient의 submit을 통하여 Yarn에 요청하게 되면, Yarn은 해당 정보를 기반으로 리소스를 할당한 후 Application Master를 실행시킨다. 하지만 현재는 Application Master의 Jar 파일, Application Master를 실행하기 위한 명령어 등이 설정되지 않은 상태이므로, 아래 과정을 통해 세부 내용들을 설정한다.
{% highlight java %}
import org.apache.hadoop.yarn.api.records.ContainerLaunchContext;

ContainerLaunchContext amContainer = Records.newRecord(ContainerLaunchContext.class);
Map<String, LocalResource> resMap = Maps.newHashMap();

//Application Master Jar File을 HDFS에 업로드한다.
File amJarFile = new File(AMJarPath);
if(!amJarFile.exists())
	throw new FileNotFoundException(String.format("Application Master Jar File을 찾을 수 없습니다 : %s",AMJarPath));
String jarFileName = amJarFile.getName();
LocalResource jarRes = uploadLocalResourceToHDFS(AMJarPath, appContext.getApplicationName(), appContext.getApplicationId().getId(), jarFileName);

//업로드 한 Jar File 정보를 Resource Map에 저장한 뒤 amContainer의 setLocalResources을 호출하여 세팅한다.
resMap.put(jarFileName, jarRes);
amContainer.setLocalResources(resMap);

//Application Master를 실행하기 위한 환경 변수를 설정한다.
//이 부분에서는 Application Master를 실행하기 위한 CLASSPATH를 설정해준다.
Map<String,String> envMap = Maps.newHashMap();
StringBuilder classPathEnv = new StringBuilder(Environment.CLASSPATH.$$()).append(ApplicationConstants.CLASS_PATH_SEPARATOR).append("./*");
for (String c : configuration.getStrings(YarnConfiguration.YARN_APPLICATION_CLASSPATH, YarnConfiguration.DEFAULT_YARN_CROSS_PLATFORM_APPLICATION_CLASSPATH)) {
	classPathEnv.append(ApplicationConstants.CLASS_PATH_SEPARATOR);
        classPathEnv.append(c.trim());
}
envMap.put("CLASSPATH", classPathEnv.toString());
amContainer.setEnvironment(envMap);

//Application Master를 실행할 Command를 생성한다.
List<String> cmdList = Lists.newArrayList();
cmdList.add(String.format("%s/bin/java", Environment.JAVA_HOME.$$()));
cmdList.add(String.format("-Xmx%dm", appContext.getResource().getMemory()));
cmdList.add(amMainClass);
cmdList.add(String.format("%s %d", Constants.CONTAINER_CORE_OPT,containerExecConfig.getContainerCores()));
cmdList.add(String.format("%s %d", Constants.CONTAINER_MEM_OPT,containerExecConfig.getContainerMem()));
cmdList.add(String.format("%s %d", Constants.CONTAINER_NUM_OPT,containerExecConfig.getNumContainer()));
cmdList.add(String.format("%s %d", Constants.CONTAINER_PRIORITY_OPT,containerExecConfig.getContainerPriority()));
cmdList.add(String.format("1> %s/AppMaster.stdout", ApplicationConstants.LOG_DIR_EXPANSION_VAR));
cmdList.add(String.format("2> %s/AppMaster.stderr", ApplicationConstants.LOG_DIR_EXPANSION_VAR));
amContainer.setCommands(cmdList);

//설정이 완료된 amContainer 객체를 ApplicationSubmissionContext에 세팅한다.
appContext.setAMContainerSpec(amContainer);

//Yarn에 Application Master 실행을 요청한다.
yarnClient.submitApplication(appContext);
{% endhighlight %}
상세 설정이 굉장히 많지만, 차근차근 살펴보면 어렵지 않았다.
일단, Application Master는 현재 Application 실행을 요청하는 머신이 아닌 Yarn Cluster 내의 특정 머신에서 실행되게 되므로, HDFS에 업로드 하여 해당 머신이 Application Master Jar를 다운받아 실행할 수 있도록 한다. 이 부분이 uploadLocalResourceToHDFS() 메서드에서 이루어지게 된다. 만일 다른 Dependency파일들도 실행시에 필요하다면 동일한 방식으로 HDFS에 업로드하면 된다.

Application Master를 실행하는 머신은 HDFS의 어떤 위치에 필요한 Resource가 존재하는지 알 수 없다. 또한 실행하는 머신에서도 어떤 머신에서 Application Master가 실행될지, 어떤 경로에서 실행될 지 알 수 없다. 따라서 Map<String, LocalResource> 형태의 객체를 NodeManager에게 전달하는 amContainer 객체의 setLocalResource 메소드를 이용하여 세팅한다.
여기서 Map의 Key는 HDFS에 업로드 된 정보를 다운로드 받을 때 저장할 파일의 이름(또는 경로), Value는 다운로드 받을 파일의 정보(HDFS에 업로드 되어 있는)이다.

그 후에는 Application Master 실행 환경을 설정하는 setEnvironment() 메소드를 호출하게 된다. 이 예제에서는 CLASSPATH를 구성하여 세팅해주는데, ./*으로 설정하게 되면, 실행 위치에 있는 모든 파일들을 Classpath에 넣게 된다. 위에서 setLocalResource의 key를 폴더/파일 형식이 아닌 파일명 형식으로만 지정했다면, 해당 파일들이 모두 Classpath에 들어가게 된다.

아래는 우리가 실행시킬 Application Master의 main() 함수이다. 물론 Yarn API를 이용하여 구현하지 않았기 때문에 실패로 동작하지만, 해당 Application이 실행되는 위치에 우리가 업로드 한 파일이 다운로드 되었는지, 실행은 어떤 방식으로 이루어지는지를 알 수 있다.

{% highlight java %}
package com.leeyh0216.test;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;

public class Test {

	public static void main(String[] args) throws Exception{
		System.out.println("Max Mem : "+Runtime.getRuntime().maxMemory());
		for(int i = 0;i<args.length;i++){
			System.out.println(String.format("Arg : %s", args[i]));
		}
		
		File root = new File(".");
		System.out.println(root.getAbsolutePath());
		File[] listFiles = root.listFiles();
		for(File f : listFiles ){
			System.out.println(f.getAbsolutePath());
			if(f.getName().contains("launch")||f.getName().contains("default")){
				BufferedReader bufferedReader = new BufferedReader(new FileReader(f));
				String str = "";
				while((str=bufferedReader.readLine())!=null){
					System.out.println(str);
				}
			}
		}
	}
}
{% endhighlight %}

위 파일을 Jar 형태로 생성하여 YarnJobLauncher를 실행시키면 다음과 같은 결과가 Container Output으로 설정된다.
{% highlight bash %}
Max Mem : 50331648
Arg : -cc
Arg : 1
Arg : -cm
Arg : 50
Arg : -cn
Arg : 1
Arg : -p
Arg : 5
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/.
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./core-site.xmlttt
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./SampleJavaProject-1.0.jar
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./.default_container_executor.sh.crc
crc?? ?Q?R
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./container_tokens
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./.default_container_executor_session.sh.crc
crc??U

echo $$ > /data/hadoop/yarn-nm-local/nmPrivate/application_1489894768082_0022/container_1489894768082_0022_02_000001/container_1489894768082_0022_02_000001.pid.tmp
/bin/mv -f /data/hadoop/yarn-nm-local/nmPrivate/application_1489894768082_0022/container_1489894768082_0022_02_000001/container_1489894768082_0022_02_000001.pid.tmp /data/hadoop/yarn-nm-local/nmPrivate/application_1489894768082_0022/container_1489894768082_0022_02_000001/container_1489894768082_0022_02_000001.pid
exec setsid /bin/bash "/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/launch_container.sh"
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./.container_tokens.crc
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./launch_container.sh

export LOCAL_DIRS="/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022"
export APPLICATION_WEB_PROXY_BASE="/proxy/application_1489894768082_0022"
export HADOOP_CONF_DIR="/home/hadoop/hadoop/etc/hadoop"
export MAX_APP_ATTEMPTS="2"
export NM_HTTP_PORT="8042"
export JAVA_HOME="/usr/lib/java"
export LOG_DIRS="/home/hadoop/hadoop/logs/userlogs/application_1489894768082_0022/container_1489894768082_0022_02_000001"
export NM_AUX_SERVICE_mapreduce_shuffle="AAA0+gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
"
export NM_PORT="33087"
export USER="leeyh0216"
export HADOOP_YARN_HOME="/home/hadoop/hadoop"
export CLASSPATH="$CLASSPATH:./*:$HADOOP_CONF_DIR:$HADOOP_COMMON_HOME/share/hadoop/common/*:$HADOOP_COMMON_HOME/share/hadoop/common/lib/*:$HADOOP_HDFS_HOME/share/hadoop/hdfs/*:$HADOOP_HDFS_HOME/share/hadoop/hdfs/lib/*:$HADOOP_YARN_HOME/share/hadoop/yarn/*:$HADOOP_YARN_HOME/share/hadoop/yarn/lib/*"
export APP_SUBMIT_TIME_ENV="1489903145986"
export NM_HOST="hadoop02"
export HADOOP_TOKEN_FILE_LOCATION="/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/container_tokens"
export HADOOP_HDFS_HOME="/home/hadoop/hadoop"
export LOGNAME="leeyh0216"
export JVM_PID="$$"
export PWD="/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001"
export HADOOP_COMMON_HOME="/home/hadoop/hadoop"
export HOME="/home/"
export CONTAINER_ID="container_1489894768082_0022_02_000001"
export MALLOC_ARENA_MAX="4"
ln -sf "/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/filecache/11/yarn-site.xml" "yarn-site.xmlttt"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi
ln -sf "/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/filecache/14/mapred-site.xml" "mapred-site.xmlttt"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi
ln -sf "/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/filecache/10/SampleJavaProject-1.0.jar" "SampleJavaProject-1.0.jar"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi
ln -sf "/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/filecache/12/core-site.xml" "core-site.xmlttt"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi
ln -sf "/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/filecache/13/hdfs-site.xml" "hdfs-site.xmlttt"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi
exec /bin/bash -c "$JAVA_HOME/bin/java -Xmx50m com.leeyh0216.test.Test -cc 1 -cm 50 -cn 1 -p 5 1> /home/hadoop/hadoop/logs/userlogs/application_1489894768082_0022/container_1489894768082_0022_02_000001/AppMaster.stdout 2> /home/hadoop/hadoop/logs/userlogs/application_1489894768082_0022/container_1489894768082_0022_02_000001/AppMaster.stderr"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./.launch_container.sh.crc
?)???1wB??-????r???	?
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./yarn-site.xmlttt
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./default_container_executor.sh
#!/bin/bash
/bin/bash "/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/default_container_executor_session.sh"
rc=$?
echo $rc > "/data/hadoop/yarn-nm-local/nmPrivate/application_1489894768082_0022/container_1489894768082_0022_02_000001/container_1489894768082_0022_02_000001.pid.exitcode.tmp"
/bin/mv -f "/data/hadoop/yarn-nm-local/nmPrivate/application_1489894768082_0022/container_1489894768082_0022_02_000001/container_1489894768082_0022_02_000001.pid.exitcode.tmp" "/data/hadoop/yarn-nm-local/nmPrivate/application_1489894768082_0022/container_1489894768082_0022_02_000001/container_1489894768082_0022_02_000001.pid.exitcode"
exit $rc
/data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001/./mapred-site.xmlttt
{% endhighlight %}

위 결과를 확인해보면, /data/hadoop/yarn-nm-local/usercache/leeyh0216/appcache/application_1489894768082_0022/container_1489894768082_0022_02_000001 디렉토리에서 Application Master Jar이 실행되는 것을 확인할 수 있다. 또한 우리가 다운로드 한 HDFS의 파일들은 해당 디렉토리의 filecache/{숫자}/파일명 형식으로 저장되는데 이를 ln -sf 명령어를 통해서 실행 디렉토리로 옮기고, 이때 사용되는 것이 우리가 setLocalResource의 인자로 지정한 Map의 Key값임을 확인할 수 있다.

다음 글에서는 Yarn API를 이용해서 실제로 동작하는 Application Master를 구현해 볼 것이다.
