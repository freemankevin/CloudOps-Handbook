## 1 Compose 部署

>**创建数据目录**

```shell
mkdir -p /data/nexus
chmod -R 775 /data/nexus
```

```yaml
version: '3.6'
services:
  nexus:
    image: myharbor.mydomain.com/app/sonatype/nexus3:latest
    container_name: nexus
    networks:
      - nexus
    restart: always
    environment:
      - TZ=Asia/Shanghai
    ports:
      - '8081:8081'
    cpus: '1.0'
    mem_limit: 2g
    volumes:
      - /data/nexus:/nexus-data

networks:
  nexus:
    external: false

```



>**查看初始密码**

```shell
cat /data/nexus/admin.password  
```



## 2 公有云加速


### 2.1 华为云

[官方地址](https://mirrors.huaweicloud.com/home)

```xml
<mirror>
    <id>huaweicloud</id>
    <mirrorOf>*</mirrorOf>
	    <url>https://repo.huaweicloud.com/repository/maven/</url>
</mirror>
```

### 2.2 阿里云

[官方地址](https://developer.aliyun.com/mvn/guide)

```xml
<mirror>
  <id>aliyunmaven</id>
  <mirrorOf>*</mirrorOf>
  <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```

## 3 角色管理

### 3.1 开发者角色

```yaml
nx-component-upload
#nx-repository-admin---read
nx-repository-view-*-*-*
nx-repository-view-*-*-add
#nx-reposltory-vlew-*-*-delete
nx-reposltory-view-*-*-read
nx-repository-view-*-*-edit
```



## 4 本地Jar 文件及目录树批量上传


```shell
#!/bin/bash
# copy and run this script to the root of the repository directory containing files
# this script attempts to exclude uploading itself explicitly so the script name is important
# Get command line params
while getopts ":r:u:p:" opt; do
	case $opt in
		r) REPO_URL="$OPTARG"
		;;
		u) USERNAME="$OPTARG"
		;;
		p) PASSWORD="$OPTARG"
		;;
	esac
done
 
find . -type f -not -path './mavenimport\.sh*' -not -path '*/\.*' -not -path '*/\^archetype\-catalog\.xml*' -not -path '*/\^maven\-metadata\-local*\.xml' -not -path '*/\^maven\-metadata\-deployment*\.xml' | sed "s|^\./||" | xargs -I '{}' curl -u "$USERNAME:$PASSWORD" -X PUT -v -T {} ${REPO_URL}/{} ;
```


>进入仓库目录，执行上传到 `Nexus` 远程仓库目录，必须是 `Hosted` 类型

```shell
[root@localhost repository]# pwd
/data/Maven/repository
[root@localhost repository]# ./mavenimport.sh -u developer -p Developer@123 -r http://10.3.2.40:8081/repository/maven-releases
```



或者可以通过阿里云云效的Jar包工具实现，文档参数地址：

https://help.aliyun.com/document_detail/131463.html
