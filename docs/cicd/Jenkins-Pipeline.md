
## 1 常識

### 1.1 版本支持

[https://pkg.origin.jenkins.io/redhat-stable/](https://pkg.origin.jenkins.io/redhat-stable/)


## 2 使用

### 2.1 指定節點-1

```groovy
pipeline {
    agent any
    // agent{
    //     node {
    //       label 'slave01'
    //     }
    // }
//     agent {
//         docker { image 'harbor.dockerregistry.com/kubesphere/builder-nodejs:v3.2.0-1' }
//     }
...
```

### 2.1 指定節點-2

```groovy
node("slave01"){
  ...
}
```

### 2.2 定時構建

>JAVA 定時語法

[https://www.cnblogs.com/zhongyehai/p/10577168.html](https://www.cnblogs.com/zhongyehai/p/10577168.html)

```shell
## 語法：
* * * * *
# 第一个* 表示分钟，取值0~59
# 第二个* 表示小时，取值0~23  
# 第三个* 表示一个月的第几天，取值1~31
# 第四个* 表示第几月，取值1~12  
# 第五个* 表示一周中的第几天，取值0~7，其中0和7代表的都是周日


## 常用
# 每隔5分钟构建一次
H/5 * * * *
# 每两小时构建一次
H H/2 * * *
# 每天中午下班前定时构建一次
H 12 * * *
# 每天下午下班前定时构建一次
H 18 * * *
# 每30分钟构建一次
H/30 * * * *
# 每2个小时构建一次
H H/2 * * *
# 每天早上8点构建一次
H 8 * * *
# 每天的8点，12点，22点，一天构建3次
H 8,12,22 * * *
```

### 2.3 指定k8s 環境

```groovy
stage('Deploy') {
        echo "5. Deploy Stage"
        def userInput = input(
            id: 'userInput',
            message: 'Choose a deploy environment',
            parameters: [
                [
                    $class: 'ChoiceParameterDefinition',
                    choices: "Dev\nQA\nProd",
                    name: 'Env'
                ]
            ]
        )
        echo "This is a deploy step to ${userInput}"
        sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s.yaml"
        if (userInput == "Dev") {
            // deploy dev stuff
        } else if (userInput == "QA"){
            // deploy qa stuff
        } else {
            // deploy prod stuff
        }
        sh "kubectl apply -f k8s.yaml"
    }
```


### 2.4 Git 子模塊認證

[https://segmentfault.com/a/1190000005018530](https://segmentfault.com/a/1190000005018530)

### 2.5 git push 觸發構建

[https://blog.csdn.net/qq_31519989/article/details/108143299](https://blog.csdn.net/qq_31519989/article/details/108143299)


### 2.6 代碼掃描

```groovy
   stage('代码扫描') {
       sh "mvn -f portal-biz/${project_name} -Dsonar.projectKey=${project_name} -Dsonar.sources=./src/ sonar:sonar -Dsonar.host.url=http://10.0.1.185:8000 -Dsonar.login=45f767ffa483beeb1bda9f7f177f7e07702bea2e"
       sh "echo 代码扫描结束"
   }
```


### 2.7 通過脚本批量拷貝工程

>`Script Console`

```groovy
// 拷贝以“A-基础运维”视图中的“blade”开头的流水线到新视图的“F-基础信息平台测试环境”中，然后重命名为以“test-blade”开头的视图
import hudson.model.*
        //源view
        def str_view = "A-基础运维"
        //目标view
        def str_new_view = "F-基础信息平台测试环境"
        //源job名称(模糊匹配)
        def str_search = "blade"
        //目标job名称(模糊匹配后替换)
        def str_replace = "test-blade"
        def view = Hudson.instance.getView(str_view)
        //copy all projects of a view
        for(item in view.getItems())
        {
          //create the new project name
          newName = item.getName().replace(str_search, str_replace)
          // copy the job, disable and save it
          def job
          try {
                //因为第一次导入后报错，所以添加了try-catch 跳过已存在的job
                job = Hudson.instance.copy(item, newName)
          } catch(IllegalArgumentException e) {
             println(e.toString())
             println("$newName job is exists")
             continue
          } catch(Exception e) {
            println(e.toString())
            continue
          }
      //是否禁用任务，false不禁用，true禁用
          job.disabled = false
          job.save() 
          Hudson.instance.getView(str_new_view).add(job)
          println(" $item.name copied as $newName")
        }
```


### 2.8 合并分支

```groovy
node {
   
   stage('合并分支') {
       sshagent (credentials: ['cicd']) {
         sh """
           git clone ${git_URL}
           cd gd-portal-agg 
           git checkout ${branch}
           git pull 
           git config --global merge.ours.driver true
           echo 'LauncherConstant.java  merge=ours' >> .gitattributes
           echo 'Dockerfile  merge=ours' >> .gitattributes
           echo 'pom.xml merge=ours' >> .gitattributes
           git add .gitattributes
           git commit -m '合并时忽略'
           git push
           git merge origin/${branch_target}
           git push
         """
       }
   }
```


### 2.9 使用流水綫

>新建視圖->列表視圖->新建工程->流水綫


### 2.10 觸發構建環境搭建

>插件列表：

[Git plugin](https://plugins.jenkins.io/git)
[GitLab Plugin](https://plugins.jenkins.io/gitlab-plugin)
[Gitlab Hook Plugin](https://plugins.jenkins.io/gitlab-hook)
[Build Authorization Token Root Plugin](https://plugins.jenkins.io/build-token-root)

> [关闭跨站请求伪造保护（CSRF）](https://www.cnblogs.com/kazihuo/p/12937071.html)

```shell
vim /etc/sysconfig/jenkins
JENKINS_JAVA_OPTIONS="-server -Xms512m -Xmx1024m -XX:PermSize=256M -XX:MaxPermSize=512m -Djava.awt.headless=true -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true"
```

>创建token,給工程做觸發使用

[![excell.png](https://i.postimg.cc/YSQqSkf0/excell.png)](https://postimg.cc/PLrk6g7n)

>创建凭证，給Jenkins鏈接GITLAB

[![excell.png](https://i.postimg.cc/dQfMgprx/excell.png)](https://postimg.cc/F10nLPPy)

>修改系统配置,添加Gitlab配置信息，添加上面的憑證

[![excell.png](https://i.postimg.cc/W4dYmbfp/excell.png)](https://postimg.cc/PLkMThj0)

>獲取Jenkins的webhook地址，配置到GITLAB

[![excell.png](https://i.postimg.cc/6qchJD11/excell.png)](https://postimg.cc/8fFvLnGh)

>將Jenkins 服務器的公鑰添加到GITLAB的項目或用戶配置的SSH KEYS  中

>在Gitlab項目中添加webhook

[![excell.png](https://i.postimg.cc/FsjYRyrN/excell.png)](https://postimg.cc/Tp3dNW54)


### 2.11 權限下放

1. 安裝所需插件：`role-strategy`
2. 修改全局安全配置-->授權策略：`Role-Based Strategy`
3. 新建用户, 依据需要新建即可





## 3 維護操作

### 3.1 Java 環境
```
Failed to start LSB: Jenkins Automation Server. jenkins
```

```shell
vi /etc/rc.d/init.d/jenkins

...
candidates="
/etc/alternatives/java
/usr/lib/jvm/java-1.8.0/bin/java
/usr/lib/jvm/jre-1.8.0/bin/java
/usr/lib/jvm/java-1.7.0/bin/java
/usr/lib/jvm/jre-1.7.0/bin/java
/usr/lib/jvm/java-11.0/bin/java
/usr/lib/jvm/jre-11.0/bin/java
/usr/lib/jvm/java-11-openjdk-amd64
/usr/bin/java
#/alldev/jdk1.8.0_261/bin/java  注释这里
```


### 3.2 更換源

> 插件管理

```shell
# 替换旧配置
# https://updates.jenkins.io/update-center.json 
https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
```


### 3.3 忘記密碼
[https://jenkins-zh.cn/tutorial/management/auth/lost-password/](https://jenkins-zh.cn/tutorial/management/auth/lost-password/)


>找回密码

如果忘记了 Jenkins 的管理员密码的话，也不用担心，只要你有权限访问 Jenkins 的根目录，就可以轻松地重置密码。

>通常情况

Jenkins 的根目录，默认是在当前用户（启动 Jenkins 的操作系统用户）目录的 `.jenkins` 中。对于 unix 系统而言，也就是 `~/.jenkins`。
我们打开配置文件 `vim ~/.jenkins/config.xml` 后，把字段 `useSecurity` 的值设置为 `false`，然后重新启动 Jenkins。你就会发现不需要登陆也能进入管理页面。
进入页面**系统管理->全局安全配置**中，选择**启用安全**，然后在**授权策略**中勾选**任何用户可以做任何事(没有任何限制)**。保存后，找到你希望重置密码的用户，输入新的密码即可。

>其他情況

如果 Jenkins 的根目录在当前用户目录下话，可以通过查找对应进程的方式来确认，下面以 Linux 系统为例说明：
```shell
$ ps -ef | grep jenkins
root     26386 26359  0 Jul23 pts/0    00:24:04 java -Duser.home=/var/jenkins_home -Djenkins.model.Jenkins.slaveAgentPort=50000 -jar /usr/share/jenkins/jenkins.war

# 分析上面的输出，可以得出，该 Jenkins 的根目录为：`/var/jenkins_home`
```

>`JENKINS_HOME`

如果你的环境变量里设置了 `JENKINS_HOME` 的话，那么， Jenkins 的根目录就可能是 `JENKINS_HOME` 指向的目录。

### 3.4 通過插件備份工程

http://www.mydlq.club/article/60/http://www.mydlq.club/article/60/


>`ThinBackup`

[![excell.png](https://i.postimg.cc/P5k7jjx2/excell.png)](https://postimg.cc/VdDgq2XC)

### 3.5 服務管理

```shell
# 主配置文件
/etc/sysconfig/jenkins

# 系统自启动服务
/etc/rc.d/init.d/jenkins

# 日志/war路径 配置
/etc/init.d/jenkins
PARAMS="--logfile=/data/jenkins/var/log/jenkins/jenkins.log --webroot=/data/jenkins/var/cache/jenkins/war --daemon"

# 修改配置
# 停服,转移数据,重启服务
systemctl stop jenkins
rsync -av /var/cache/jenkins/war/* /data/jenkins/var/cache/jenkins/war/
rsync -av /var/log/jenkins/jenkins.log  /data/jenkins/var/log/jenkins/
systemctl daemon-reload
systemctl start jenkins

# 开机自启动
/sbin/chkconfig jenkins on
```


### 3.6 在服務器離綫備份工程文件

>这种方法适合跨Jenkins复制，Jenkins的job都在$JENKINS_HOME/jobs目录（一般是/var/lib/jenkins/jobs）下，每个job一个目录

```shell
# 复制全部job
cd /var/lib/jenkins
# 在源Jenkins上压缩jobs目录
tar -czvf jobs.tar.gz jobs
# 在目标Jenkins上解压jobs目录
tar -zxvf jobs.tar.gz

# 复制某个job
cd /var/lib/jenkins/jobs
# 在源Jenkins上压缩指定的job目录
tar -czvf myjob.tar.gz myjob
# 在目标Jenkins上解压指定的job目录
tar -zxvf myjob.tar.gz

## 在目标Jenkins上，打开Manage Jenkins，选择Reload Configuration from Disk,不需要重启目标Jenkins。
```



### 3.7 配置優化

>内存限制

这里的几个 JVM 参数含义如下：
- -Xms: 使用的最小堆内存大小
- -Xmx: 使用的最大堆内存大小
- -XX:PermSize: 内存的永久保存区域大小
- -XX:MaxPermSize: 最大内存的永久保存区域大小

这几个参数也不是配置越大越好，具体要根据所在机器实际内存和使用大小配置。

```shell
$ vim /etc/sysconfig/jenkins
JENKINS_JAVA_OPTIONS="-server -Xms512m -Xmx1024m -XX:PermSize=256M -XX:MaxPermSize=512m -Djava.awt.headless=true"

systemctl restart jenkins.service
```

>[高版本Jenkins关闭跨站请求伪造保护（CSRF）](https://www.cnblogs.com/kazihuo/p/12937071.html)

```shell
$ vim /etc/sysconfig/jenkins
JENKINS_JAVA_OPTIONS="-server -Xms512m -Xmx1024m -XX:PermSize=256M -XX:MaxPermSize=512m -Djava.awt.headless=true -Dhudson.security.csrf.GlobalCrumbIssuerConfiguration.DISABLE_CSRF_PROTECTION=true"
```

### 3.8 數據遷移

[https://zhuanlan.zhihu.com/p/164827268](https://zhuanlan.zhihu.com/p/164827268)

```shell
### 1、找到主目录
/var/lib/jenkins ## 默認是這個

### 2、迁移关键的四个文件或文件夹
config.xml文件、jobs文件夹、users文件夹和plugins

jobs       //存放的job信息
config.xml //权限，分组，项目，结构等配置信息
plugins    //插件文件
users      //用户文件


## 3 數據備份
lsblk
df -Th /data/
cd /data/
mkdir -p jenkins jenkins.old
cd jenkins.old
# 关键数据全量备份
tar  -czvf jenkins-old.data-`date +%Y%m%d`.tar.gz  /var/lib/jenkins/users  /var/lib/jenkins/jobs /var/lib/jenkins/plugins /var/lib/jenkins/config.xml
# 数据目录全量备份
tar  -czvf jenkins-old.`date +%Y%m%d`.tar.gz /var/lib/jenkins

## 4 修改配置
ll /data/jenkins
vi +10 /etc/sysconfig/jenkins
# 改成新路径
JENKINS_HOME="/data/jenkins"
# 改成root,之前是Jenkins
JENKINS_USER="root"

## 5 數據遷移，目錄權限修改
mv var/lib/jenkins/* /data/jenkins/
systemctl restart jenkins
systemctl status jenkins.service

ll /data/jenkins
rm /data/jenkins/jenkins-old.data-20201127.tar.gz -f
vi  /etc/sysconfig/jenkins
# 这个可以给用户改回去，还使用jenkins
JENKINS_USER="jenkins"
chown -R   jenkins.jenkins /data/jenkins/
chown -R   jenkins.jenkins /var/log/jenkins/
chown -R   jenkins.jenkins /var/cache/jenkins/
chown -R   jenkins.jenkins /usr/lib/jenkins/jenkins.war
chown -R   jenkins.jenkins /etc/sysconfig/jenkins

# 可能存在丟數據的情況
##### 1、gitlab的代码拉取的密钥得重做
##### 2、ssh 到远程主机的私钥及其他配置得重新配置
```


### 3.9 子節點配置

[https://blog.csdn.net/lusyoe/article/details/54880836](https://blog.csdn.net/lusyoe/article/details/54880836)

[![excell.png](https://i.postimg.cc/MZnp2B3Z/excell.png)](https://postimg.cc/K4y2t1yC)

[![excell.png](https://i.postimg.cc/5t9fPQFV/excell.png)](https://postimg.cc/gx59jJDS)


### 3.10 添加遠程主機

>免密登录

>第一步，jenkins机器的私钥放到KEY里去

[![excell.png](https://i.postimg.cc/FHQp7RGt/excell.png)](https://postimg.cc/GHqGfbZz)
	
>第二步，将jenkins机器的公钥分别传给要远程连接的机器

```shell
ssh-copy-id HOST-NAME
```

>測試

[![excell.png](https://i.postimg.cc/4N4NZbkj/excell.png)](https://postimg.cc/Z0ghxNqL)

### 3.11 清理缓存

```groovy
node {
    // This step utilizes the Workspace Cleanup Plugin: https://wiki.jenkins.io/display/JENKINS/Workspace+Cleanup+Plugin
    step([$class: 'WsCleanup'])  
}
```

## 4 部署

### 4.1 yum 方式

```shell
#!/bin/bash


# 1.修改镜像源
cd /etc/yum.repos.d && \
rm * -rf  && \
cat >epel-7.repo<<EOF
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
baseurl=http://mirrors.aliyun.com/epel/7/$basearch
failovermethod=priority
enabled=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

[epel-debuginfo]
name=Extra Packages for Enterprise Linux 7 - $basearch - Debug
baseurl=http://mirrors.aliyun.com/epel/7/$basearch/debug
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=0

[epel-source]
name=Extra Packages for Enterprise Linux 7 - $basearch - Source
baseurl=http://mirrors.aliyun.com/epel/7/SRPMS
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=0
EOF 

cat >Centos-7.repo<<EOF
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.
#
#

[base]
name=CentOS-$releasever - Base - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/os/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/os/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/updates/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/extras/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/centosplus/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/centosplus/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#contrib - packages by Centos Users
[contrib]
name=CentOS-$releasever - Contrib - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/contrib/$basearch/
        http://mirrors.aliyuncs.com/centos/$releasever/contrib/$basearch/
        http://mirrors.cloud.aliyuncs.com/centos/$releasever/contrib/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
EOF 

# 2.安装基本工具包
mkdir -p /root/software/ && \
yum -y install git vim telnet net-tools patch tree wget curl lrzsz && \


# 3.安装java环境
yum -y install java-1.8.0-openjdk && \

# 4.配置jenkins镜像源
cd /etc/yum.repos.d  && \
cat >jenkins.repo<<EOF
[jenkins]
name=Jenkins
baseurl=http://pkg.jenkins.io/redhat
gpgcheck=1
EOF

# 5.安装jenkins
# 提前下载并上传至/root/software
yum -y install jenkins-2.93-1.1.noarch.rpm && \

# 6.修改jenkins的启动用户
sed -i '29s/JENKINS_USER="jenkins"/JENKINS_USER="root"/' /etc/sysconfig/jenkins && \

# 7.可以选择修改默认的8080端口为9090
sed -i '56s/8080/9090/' /etc/sysconfig/jenkins && \

# 8.修改默认的数据目录
sed -i '10s#/var/lib/jenkins#/root/jenkins-data#' /etc/sysconfig/jenkins && \ 

# 9.创建日志目录，数据目录
ln -s /var/lib/jenkins jenkins-data && \
ln -s /var/log/jenkins jenkins-logs && \

# 10.设置开机启劢
chkconfig jenkins on && \ 
chkconfig --list jenkins && \

# 11.创建服务管理脚本
cat >start-jenkins.sh<<EOF
#!/bin/bash
/etc/init.d/jenkins start
EOF
cat >restart-jenkins.sh<<EOF
#!/bin/bash
/etc/init.d/jenkins restart
EOF
cat >stop-jenkins.sh<<EOF
#!/bin/bash
/etc/init.d/jenkins stop
EOF

# 12.开启或关闭防火墙 
sudo firewall-cmd --zone=public --add-port=9090/tcp --permanent && \
sudo firewall-cmd --reload && \
sudo firewall-cmd --list-all && \
systemctl stop firewalld && \
systemctl disable firewalld && \

sed -i '7s/enforcing/disabled/' /etc/selinux/config && \
setenforce 0 && \
getenforce  && \

# 12.访问jenkins管理界面
# http://192.168.216.108:9090/ 
# 初始密码：cat /var/lib/jenkins/secrets/initialAdminPassword


# 13.更新插件包
cd && \  
cd jenkins-data && \ 
mv plugins/ plugins.bak && \ 
tar -xzvf /root/software/plugins.tar.gz && \
chown -R jenkins.jenkins ./* && \
cd && \ 
./restart-jenkins.sh 


## 14 自行部署java 、maven、 python 、node/npm/yarn 代碼編譯構建的環境，並將環境變量與路徑同步在Jenkins 配置中
```



## 5 推荐博客

[二丫讲梵](https://wiki.eryajf.net/pages/2415.html#_1-%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E3%80%82)



## 6 Jenkinsfile(backend)

```yaml
pipeline {
//     agent any
    agent{
        node {
          label 'slave01'
        }
    }
//     agent {
//         docker { image 'harbor.dockerregistry.com/kubesphere/builder-nodejs:v3.2.0-1' }
//     }
    environment {
        //git凭证ID
        GITLAB_CREDENTIAL_ID = 'xxxxxxxxxxxx'
        //代码地址
        GIT_REGISTRY = 'xxxxxxxxxxxx'
        //分支名称
        BRANCH_NAME = 'xxxxx'
        //应用名称
        APP_NAME = 'xxxxxx'
        //Harbor地址
        HARBOR_HOST = 'xxxxxx'
        //Harbor登录用户
        HABOR_USERNAME = 'xxxx'
        //Harbor登录密码
        HARBOR_PASSWORD = 'xxxxxx'
        //Harbor仓库名称
        HARBOR_PROJECT = 'xxxxx'
        //Harbor凭证
        HARBOR_CREDENTIAL_ID = 'xxxx'
        //容器镜像名称
        CONTAINER_IMAGE = "$HARBOR_HOST/$HARBOR_PROJECT/$APP_NAME"
        //k8s集群名称
        K8S_NAMESPACE = 'xxxx'
        //容器端口
        CONTAINER_PORT = "xxxxx"
        //GIT_COMMIT_ID前七位数字
        CI_COMMIT_SHORT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    }

    stages {
        stage('ENV Test') {
            steps {
                sh 'echo "开始检查环境"'
                sh 'echo "GIT_COMMIT_ID前七位数字: $CI_COMMIT_SHORT_SHA"'
                sh 'docker --version'
//              sh 'node --version'
//              sh 'npm --version'
//              sh 'yarn --version'
            }
        }

        stage('Code Pull') {
          steps {
            git(branch: "$BRANCH_NAME", url: "$GIT_REGISTRY", credentialsId: "$GITLAB_CREDENTIAL_ID", changelog: true, poll: false)
          }
        }
        stage('Maven Install') {
            steps {
                sh 'echo "安装maven依赖"'
                sh '''
                    mvn -Dmaven.test.skip=true clean install -U
                '''
            }
        }
        stage('Maven Package') {
            steps {
                sh 'echo "开始maven打包"'
                sh '''
                    mvn -f  $APP_NAME -Dmaven.test.skip=true clean package
                '''
            }
        }
        // stage('Maven Deploy') {
        //     steps {
        //         sh 'echo "开始推送私库"'
        //         sh '''
        //             mvn -f $APP_NAME -Dmaven.test.skip=true clean deploy
        //         '''
        //     }
        // }
        stage('Docker Build') {
            steps {
                sh 'echo "开始Docker构建"'
                sh '''
                    cd $APP_NAME
                    docker build                                     \
                      --cache-from $CONTAINER_IMAGE                  \
                      --tag  $CONTAINER_IMAGE:latest                 \
                      --tag  $CONTAINER_IMAGE:$CI_COMMIT_SHORT_SHA   \
                      --file Dockerfile                              \
                      "."
                '''
            }
        }
        stage('Docker Push') {
            steps {
                sh 'echo "开始推送Docker镜像"'
                sh '''
                    docker login -u $HABOR_USERNAME -p $HARBOR_PASSWORD $HARBOR_HOST
                    docker push $CONTAINER_IMAGE:latest
                    docker push $CONTAINER_IMAGE:$CI_COMMIT_SHORT_SHA
                '''
            }
        }
        stage('Update Yaml') {
            steps {
                sh 'echo "开始更新部署YAML文件中的镜像标签"'
                sh '''
                    cd $APP_NAME
                    sed -ri s@latest@$CI_COMMIT_SHORT_SHA@ deploy/deploy.yaml
                    grep 'image:' deploy/deploy.yaml
                '''
            }
        }
        stage('Deploy To K8') {
            steps {
                sh 'echo "开始部署应用$APP_NAME 到K8命名空间 $K8S_NAMESPACE"'
                sh '''
                    cd $APP_NAME
                    kubectl apply -f deploy --kubeconfig=/root/.kube/kubeconfig
                '''
            }
        }
    }

//     post {
//         always {
//             echo 'This will always run'
//         }
//         success {
//             echo 'This will run only if successful'
//         }
//         failure {
//             echo 'This will run only if failed'
//         }
//         unstable {
//             echo 'This will run only if the run was marked as unstable'
//         }
//         changed {
//             echo 'This will run only if the state of the Pipeline has changed'
//             echo 'For example, if the Pipeline was previously failing but is now successful'
//         }
//     }
    post {
        cleanup {
            /* clean up our workspace */
            deleteDir()
            /* clean up tmp directory */
            dir("${workspace}@tmp") {
                deleteDir()
            }
            /* clean up script directory */
            dir("${workspace}@script") {
                deleteDir()
            }
        }
    }
}
```





## 7 Jenkinsfile(frontend)

```yaml
pipeline {
//     agent any
    agent{
        node {
          label 'slave03'
        }
    }

    environment {
        //git凭证ID
        GITLAB_CREDENTIAL_ID = 'xxxxxxxxxxxxxxxx'
        //代码地址
        GIT_REGISTRY = 'xxxxxxxxxxxxxx'
        //分支名称
        BRANCH_NAME = 'xxxxxxxxxxxxxx'
        //应用名称
        APP_NAME = 'xxxxxxxxxxx'
        //Harbor地址
        HARBOR_HOST = 'xxxxxxxxxxxxx'
        //Harbor登录用户
        HABOR_USERNAME = 'xxxx'
        //Harbor登录密码
        HARBOR_PASSWORD = 'xxxxxx'
        //Harbor仓库名称
        HARBOR_PROJECT = 'xxxxxx'
        //Harbor凭证
        HARBOR_CREDENTIAL_ID = 'xxxxx'
        //容器镜像名称
        CONTAINER_IMAGE = "$HARBOR_HOST/$HARBOR_PROJECT/$APP_NAME"
        //k8s集群名称
        K8S_NAMESPACE = 'xxxxx'
        //容器端口
        CONTAINER_PORT = "xxxxxxx"
        //GIT_COMMIT_ID前七位数字
        CI_COMMIT_SHORT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    }

    stages {
        stage('ENV Test') {
            steps {
                sh 'echo "开始检查环境"'
                sh 'echo "GIT_COMMIT_ID前七位数字: $CI_COMMIT_SHORT_SHA"'
                // sh 'docker --version'
                sh 'node --version'
                sh 'npm --version'
                sh 'yarn --version'
            }
        }

        stage('Code Pull') {
          steps {
            git(branch: "$BRANCH_NAME", url: "$GIT_REGISTRY", credentialsId: "$GITLAB_CREDENTIAL_ID", changelog: true, poll: false)
          }
        }
        stage('Code Build') {
            steps {
                sh 'echo "开始代码构建"'
                sh '''
                    npm install --registry=https://registry.npm.taobao.org
                    yarn run build
                '''
            }
        }
        stage('Docker Build') {
            steps {
                sh 'echo "开始Docker构建"'
                sh '''
                    docker build                                     \
                      --cache-from $CONTAINER_IMAGE                  \
                      --tag  $CONTAINER_IMAGE:latest                 \
                      --tag  $CONTAINER_IMAGE:$CI_COMMIT_SHORT_SHA   \
                      --file Dockerfile                              \
                      "."
                '''
            }
        }
        stage('Docker Push') {
            steps {
                sh 'echo "开始推送Docker镜像"'
                sh '''
                    docker login -u $HABOR_USERNAME -p $HARBOR_PASSWORD $HARBOR_HOST
                    docker push $CONTAINER_IMAGE:latest
                    docker push $CONTAINER_IMAGE:$CI_COMMIT_SHORT_SHA
                '''
            }
        }
        stage('Update Yaml') {
            steps {
                sh 'echo "开始更新部署YAML文件中的镜像标签"'
                sh '''
                    sed -ri s@latest@$CI_COMMIT_SHORT_SHA@ deploy/deploy.yaml
                    grep 'image:' deploy/deploy.yaml
                '''
            }
        }
        stage('Deploy To K8') {
            steps {
                sh 'echo "开始部署应用$APP_NAME 到K8命名空间 $K8S_NAMESPACE"'
                sh '''
                    kubectl apply -f deploy    --kubeconfig=/root/.kube/kubeconfig
                '''
            }
        }
    }

//     post {
//         always {
//             echo 'This will always run'
//         }
//         success {
//             echo 'This will run only if successful'
//         }
//         failure {
//             echo 'This will run only if failed'
//         }
//         unstable {
//             echo 'This will run only if the run was marked as unstable'
//         }
//         changed {
//             echo 'This will run only if the state of the Pipeline has changed'
//             echo 'For example, if the Pipeline was previously failing but is now successful'
//         }
//     }
    post {
        cleanup {
            /* clean up our workspace */
            deleteDir()
            /* clean up tmp directory */
            dir("${workspace}@tmp") {
                deleteDir()
            }
            /* clean up script directory */
            dir("${workspace}@script") {
                deleteDir()
            }
        }
    }
}
```

