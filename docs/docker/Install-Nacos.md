## Compose file

```yaml
version: '3'
services:
  nacos:
    image: harbor.dockerregistry.com/app/nacos/nacos-server:v2.2.3
    networks:
      - middleware
    container_name: nacos
    ports:
      - "8858:8848"
      - "9848:9848"
      - "9849:9849"
      - "7848:7848"
    environment:
      NACOS_AUTH_ENABLE: 'true'
    env_file:
      - ./nacos/env/nacos-standlone-mysql.env
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./nacos/init.d/application.properties:/home/nacos/conf/application.properties
      - /data/middleware/nacos/standalone-logs/:/home/nacos/logs
    depends_on:
      mysql:
        condition: service_healthy
    restart: always

  mysql:
    image: harbor.dockerregistry.com/app/mysql:8.0.30
    networks:
      - middleware
    build:
      context: .
      dockerfile: ./build/Dockerfile
    container_name: mysql
    ports:
      - "3306:3306"
    env_file:
      - ./nacos/env/mysql.env
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /data/middleware/mysql:/var/lib/mysql
    healthcheck:
      test: [ "CMD", "mysqladmin" ,"ping", "-h", "localhost" ]
      interval: 5s
      timeout: 10s
      retries: 10
    restart: always
    
    
networks:
  middleware:
    driver: bridge
```





## application.properties

```shell
#
# Copyright 1999-2021 Alibaba Group Holding Ltd.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
#*************** Spring Boot Related Configurations ***************#
### Default web context path:
server.servlet.contextPath=/nacos
### Include message field
server.error.include-message=ALWAYS
### Default web server port:
server.port=8848
#*************** Network Related Configurations ***************#
### If prefer hostname over ip for Nacos server addresses in cluster.conf:
# nacos.inetutils.prefer-hostname-over-ip=false
### Specify local server's IP:
# nacos.inetutils.ip-address=
#*************** Config Module Related Configurations ***************#
### Deprecated configuration property, it is recommended to use `spring.sql.init.platform` replaced.
#spring.datasource.platform=${SPRING_DATASOURCE_PLATFORM:}
spring.sql.init.platform=${SPRING_DATASOURCE_PLATFORM:}
# nacos.plugin.datasource.log.enabled=true
### Count of DB:
db.num=1
### Connect URL of DB:
db.url.0=jdbc:mysql://mysql:3306/nacos_devtest?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC&useSSL=false&allowPublicKeyRetrieval=true
db.user.0=nacos
db.password.0=nacos
### Connection pool configuration: hikariCP
db.pool.config.connectionTimeout=30000
db.pool.config.validationTimeout=10000
db.pool.config.maximumPoolSize=20
db.pool.config.minimumIdle=2
#*************** Naming Module Related Configurations ***************#
### Data dispatch task execution period in milliseconds:
### If enable data warmup. If set to false, the server would accept request without local data preparation:
# nacos.naming.data.warmup=true
### If enable the instance auto expiration, kind like of health check of instance:
# nacos.naming.expireInstance=true
### will be removed and replaced by `nacos.naming.clean` properties
nacos.naming.empty-service.auto-clean=true
nacos.naming.empty-service.clean.initial-delay-ms=50000
nacos.naming.empty-service.clean.period-time-ms=30000
### Add in 2.0.0
### The interval to clean empty service, unit: milliseconds.
# nacos.naming.clean.empty-service.interval=60000
### The expired time to clean empty service, unit: milliseconds.
# nacos.naming.clean.empty-service.expired-time=60000
### The interval to clean expired metadata, unit: milliseconds.
# nacos.naming.clean.expired-metadata.interval=5000
### The expired time to clean metadata, unit: milliseconds.
# nacos.naming.clean.expired-metadata.expired-time=60000
### The delay time before push task to execute from service changed, unit: milliseconds.
# nacos.naming.push.pushTaskDelay=500
### The timeout for push task execute, unit: milliseconds.
# nacos.naming.push.pushTaskTimeout=5000
### The delay time for retrying failed push task, unit: milliseconds.
# nacos.naming.push.pushTaskRetryDelay=1000
### Since 2.0.3
### The expired time for inactive client, unit: milliseconds.
# nacos.naming.client.expired.time=180000
#*************** CMDB Module Related Configurations ***************#
### The interval to dump external CMDB in seconds:
# nacos.cmdb.dumpTaskInterval=3600
### The interval of polling data change event in seconds:
# nacos.cmdb.eventTaskInterval=10
### The interval of loading labels in seconds:
# nacos.cmdb.labelTaskInterval=300
### If turn on data loading task:
# nacos.cmdb.loadDataAtStart=false
#*************** Metrics Related Configurations ***************#
### Metrics for prometheus
#management.endpoints.web.exposure.include=*
### Metrics for elastic search
management.metrics.export.elastic.enabled=false
#management.metrics.export.elastic.host=http://localhost:9200
### Metrics for influx
management.metrics.export.influx.enabled=false
#management.metrics.export.influx.db=springboot
#management.metrics.export.influx.uri=http://localhost:8086
#management.metrics.export.influx.auto-create-db=true
#management.metrics.export.influx.consistency=one
#management.metrics.export.influx.compressed=true
#*************** Access Log Related Configurations ***************#
### If turn on the access log:
server.tomcat.accesslog.enabled=true
### accesslog automatic cleaning time
server.tomcat.accesslog.max-days=30
### The access log pattern:
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D %{User-Agent}i %{Request-Source}i
### The directory of access log:
server.tomcat.basedir=file:.
#*************** Access Control Related Configurations ***************#
### If enable spring security, this option is deprecated in 1.2.0:
#spring.security.enabled=false
### The ignore urls of auth, is deprecated in 1.2.0:
nacos.security.ignore.urls=/,/error,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-ui/public/**,/v1/auth/**,/v1/console/health/**,/actuator/**,/v1/console/server/**
### The auth system to use, currently only 'nacos' and 'ldap' is supported:
nacos.core.auth.system.type=nacos
### If turn on auth system:
nacos.core.auth.enabled=false
### Turn on/off caching of auth information. By turning on this switch, the update of auth information would have a 15 seconds delay.
nacos.core.auth.caching.enabled=true
### Since 1.4.1, Turn on/off white auth for user-agent: nacos-server, only for upgrade from old version.
nacos.core.auth.enable.userAgentAuthWhite=false
### Since 1.4.1, worked when nacos.core.auth.enabled=true and nacos.core.auth.enable.userAgentAuthWhite=false.
### The two properties is the white list for auth and used by identity the request from other server.
nacos.core.auth.server.identity.key=serverIdentity
nacos.core.auth.server.identity.value=security
### worked when nacos.core.auth.system.type=nacos
### The token expiration in seconds:
nacos.core.auth.plugin.nacos.token.expire.seconds=18000
### The default token (Base64 string):
nacos.core.auth.plugin.nacos.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789
### worked when nacos.core.auth.system.type=ldap?{0} is Placeholder,replace login username
#nacos.core.auth.ldap.url=ldap://localhost:389
#nacos.core.auth.ldap.basedc=dc=example,dc=org
#nacos.core.auth.ldap.userDn=cn=admin,${nacos.core.auth.ldap.basedc}
#nacos.core.auth.ldap.password=admin
#nacos.core.auth.ldap.userdn=cn={0},dc=example,dc=org
#nacos.core.auth.ldap.filter.prefix=uid
#nacos.core.auth.ldap.case.sensitive=true
#*************** Istio Related Configurations ***************#
### If turn on the MCP server:
nacos.istio.mcp.server.enabled=false
###*************** Add from 1.3.0 ***************###
#*************** Core Related Configurations ***************#
### set the WorkerID manually
# nacos.core.snowflake.worker-id=
### Member-MetaData
# nacos.core.member.meta.site=
# nacos.core.member.meta.adweight=
# nacos.core.member.meta.weight=
### MemberLookup
### Addressing pattern category, If set, the priority is highest
# nacos.core.member.lookup.type=[file,address-server]
## Set the cluster list with a configuration file or command-line argument
# nacos.member.list=192.168.16.101:8847?raft_port=8807,192.168.16.101?raft_port=8808,192.168.16.101:8849?raft_port=8809
## for AddressServerMemberLookup
# Maximum number of retries to query the address server upon initialization
# nacos.core.address-server.retry=5
## Server domain name address of [address-server] mode
# address.server.domain=jmenv.tbsite.net
## Server port of [address-server] mode
# address.server.port=8080
## Request address of [address-server] mode
# address.server.url=/nacos/serverlist
#*************** JRaft Related Configurations ***************#
### Sets the Raft cluster election timeout, default value is 5 second
# nacos.core.protocol.raft.data.election_timeout_ms=5000
### Sets the amount of time the Raft snapshot will execute periodically, default is 30 minute
# nacos.core.protocol.raft.data.snapshot_interval_secs=30
### raft internal worker threads
# nacos.core.protocol.raft.data.core_thread_num=8
### Number of threads required for raft business request processing
# nacos.core.protocol.raft.data.cli_service_thread_num=4
### raft linear read strategy. Safe linear reads are used by default, that is, the Leader tenure is confirmed by heartbeat
# nacos.core.protocol.raft.data.read_index_type=ReadOnlySafe
### rpc request timeout, default 5 seconds
# nacos.core.protocol.raft.data.rpc_request_timeout_ms=5000
#*************** Distro Related Configurations ***************#
### Distro data sync delay time, when sync task delayed, task will be merged for same data key. Default 1 second.
# nacos.core.protocol.distro.data.sync.delayMs=1000
### Distro data sync timeout for one sync data, default 3 seconds.
# nacos.core.protocol.distro.data.sync.timeoutMs=3000
### Distro data sync retry delay time when sync data failed or timeout, same behavior with delayMs, default 3 seconds.
# nacos.core.protocol.distro.data.sync.retryDelayMs=3000
### Distro data verify interval time, verify synced data whether expired for a interval. Default 5 seconds.
# nacos.core.protocol.distro.data.verify.intervalMs=5000
### Distro data verify timeout for one verify, default 3 seconds.
# nacos.core.protocol.distro.data.verify.timeoutMs=3000
### Distro data load retry delay when load snapshot data failed, default 30 seconds.
# nacos.core.protocol.distro.data.load.retryDelayMs=30000
### enable to support prometheus service discovery
#nacos.prometheus.metrics.enabled=true

```



## Dockerfile

```shell
FROM mysql:8.0.31
#ADD https://raw.githubusercontent.com/alibaba/nacos/develop/distribution/conf/mysql-schema.sql /docker-entrypoint-initdb.d/nacos-mysql.sql
ADD ./mysql-schema.sql /docker-entrypoint-initdb.d/nacos-mysql.sql
RUN chown -R mysql:mysql /docker-entrypoint-initdb.d/nacos-mysql.sql
EXPOSE 3306
CMD ["mysqld", "--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]
```



## mysql-schema.sql

```sql
/*
 * Copyright 1999-2018 Alibaba Group Holding Ltd.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

/******************************************/
/*   表名称 = config_info                  */
/******************************************/
CREATE TABLE `config_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) DEFAULT NULL,
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `c_desc` varchar(256) DEFAULT NULL,
  `c_use` varchar(64) DEFAULT NULL,
  `effect` varchar(64) DEFAULT NULL,
  `type` varchar(64) DEFAULT NULL,
  `c_schema` text,
  `encrypted_data_key` text NOT NULL COMMENT '密钥',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfo_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info';

/******************************************/
/*   表名称 = config_info_aggr             */
/******************************************/
CREATE TABLE `config_info_aggr` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `datum_id` varchar(255) NOT NULL COMMENT 'datum_id',
  `content` longtext NOT NULL COMMENT '内容',
  `gmt_modified` datetime NOT NULL COMMENT '修改时间',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfoaggr_datagrouptenantdatum` (`data_id`,`group_id`,`tenant_id`,`datum_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='增加租户字段';


/******************************************/
/*   表名称 = config_info_beta             */
/******************************************/
CREATE TABLE `config_info_beta` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `beta_ips` varchar(1024) DEFAULT NULL COMMENT 'betaIps',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `encrypted_data_key` text NOT NULL COMMENT '密钥',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfobeta_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_beta';

/******************************************/
/*   表名称 = config_info_tag              */
/******************************************/
CREATE TABLE `config_info_tag` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `tag_id` varchar(128) NOT NULL COMMENT 'tag_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(50) DEFAULT NULL COMMENT 'source ip',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfotag_datagrouptenanttag` (`data_id`,`group_id`,`tenant_id`,`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_tag';

/******************************************/
/*   表名称 = config_tags_relation         */
/******************************************/
CREATE TABLE `config_tags_relation` (
  `id` bigint(20) NOT NULL COMMENT 'id',
  `tag_name` varchar(128) NOT NULL COMMENT 'tag_name',
  `tag_type` varchar(64) DEFAULT NULL COMMENT 'tag_type',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `nid` bigint(20) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`nid`),
  UNIQUE KEY `uk_configtagrelation_configidtag` (`id`,`tag_name`,`tag_type`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_tag_relation';

/******************************************/
/*   表名称 = group_capacity               */
/******************************************/
CREATE TABLE `group_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `group_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Group ID，空字符表示整个集群',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数，，0表示使用默认值',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_group_id` (`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='集群、各Group容量信息表';

/******************************************/
/*   表名称 = his_config_info              */
/******************************************/
CREATE TABLE `his_config_info` (
  `id` bigint(20) unsigned NOT NULL,
  `nid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `data_id` varchar(255) NOT NULL,
  `group_id` varchar(128) NOT NULL,
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL,
  `md5` varchar(32) DEFAULT NULL,
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `src_user` text,
  `src_ip` varchar(50) DEFAULT NULL,
  `op_type` char(10) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `encrypted_data_key` text NOT NULL COMMENT '密钥',
  PRIMARY KEY (`nid`),
  KEY `idx_gmt_create` (`gmt_create`),
  KEY `idx_gmt_modified` (`gmt_modified`),
  KEY `idx_did` (`data_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='多租户改造';


/******************************************/
/*   表名称 = tenant_capacity              */
/******************************************/
CREATE TABLE `tenant_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `tenant_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Tenant ID',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='租户容量信息表';


CREATE TABLE `tenant_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `kp` varchar(128) NOT NULL COMMENT 'kp',
  `tenant_id` varchar(128) default '' COMMENT 'tenant_id',
  `tenant_name` varchar(128) default '' COMMENT 'tenant_name',
  `tenant_desc` varchar(256) DEFAULT NULL COMMENT 'tenant_desc',
  `create_source` varchar(32) DEFAULT NULL COMMENT 'create_source',
  `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) NOT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_info_kptenantid` (`kp`,`tenant_id`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='tenant_info';

CREATE TABLE `users` (
	`username` varchar(50) NOT NULL PRIMARY KEY,
	`password` varchar(500) NOT NULL,
	`enabled` boolean NOT NULL
);

CREATE TABLE `roles` (
	`username` varchar(50) NOT NULL,
	`role` varchar(50) NOT NULL,
	UNIQUE INDEX `idx_user_role` (`username` ASC, `role` ASC) USING BTREE
);

CREATE TABLE `permissions` (
    `role` varchar(50) NOT NULL,
    `resource` varchar(255) NOT NULL,
    `action` varchar(8) NOT NULL,
    UNIQUE INDEX `uk_role_permission` (`role`,`resource`,`action`) USING BTREE
);

INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);

INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
```



## nacos-standlone-mysql.env

```shell
PREFER_HOST_MODE=hostname
MODE=standalone
SPRING_DATASOURCE_PLATFORM=mysql
MYSQL_SERVICE_HOST=mysql
MYSQL_SERVICE_DB_NAME=nacos_devtest
MYSQL_SERVICE_PORT=3306
MYSQL_SERVICE_USER=nacos
MYSQL_SERVICE_PASSWORD=nacos
MYSQL_SERVICE_DB_PARAM=characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false&allowPublicKeyRetrieval=true
NACOS_AUTH_IDENTITY_KEY=2222
NACOS_AUTH_IDENTITY_VALUE=2xxx
NACOS_AUTH_TOKEN=SecretKey012345678901234567890123456789012345678901234567890123456789
```





## mysql.env

```shell
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=nacos_devtest
MYSQL_USER=nacos
MYSQL_PASSWORD=nacos
LANG=C.UTF-8
```







## 开启鉴权

### 1.1 服务端更新

> 根据自己的部署模式来进行调整即可。

#### 1.1.1 Compose

>默认配置路径：`init.d/application.properties`
>**相关默认配置**：

```java
nacos.core.auth.system.type=nacos // nacos 内置鉴权插件
nacos.core.auth.enabled=false     // 不鉴权
nacos.core.auth.caching.enabled=true // 开启鉴权缓存
nacos.core.auth.enable.userAgentAuthWhite=false // 不开白名单
nacos.core.auth.server.identity.key=serverIdentity // 内置nacos 集群节点间鉴权key值
nacos.core.auth.server.identity.value=security     // 内置nacos 集群节点间鉴权value值
nacos.core.auth.plugin.nacos.token.expire.seconds=18000 // token 默认超时5h
nacos.core.auth.plugin.nacos.token.secret.key=SecretKey012345678901234567890123456789012345678901234567890123456789 // 默认token
```

>**相关新的配置：**

```java
nacos.core.auth.plugin.nacos.token.secret.key=${NACOS_AUTH_TOKEN:lXi@SD5j#7uYuFCbQuzQ4cjLvoNGiDWFqK!xo1YcZHlYy7qMSfZN0ZJcRWI%}
nacos.core.auth.enabled=${NACOS_AUTH_ENABLE:false}
nacos.core.auth.server.identity.key=${NACOS_AUTH_IDENTITY_KEY:1}
nacos.core.auth.server.identity.value=${NACOS_AUTH_IDENTITY_VALUE:1}
```

>**更新Compose 部署文件中Nacos 环境变量：**

```yaml
version: '3'
services:
  nacos:
    image: harbor.dockerregistry.com/app/nacos/nacos-server:v2.2.3
    networks:
      - middleware
    container_name: nacos
    ports:
      - "8848:8848"
      - "9848:9848"
    environment:
      NACOS_AUTH_ENABLE: 'true'
    env_file:
      - ./env/nacos-standlone-mysql.env
...
```

>重启Nacos 服务即可。
>如果修改Nacos 默认密码，需要同步修改后端服务启动时的环境变量，当然前提是代码里已内置过Nacos用户密码的配置支持，具体细节可以参考后面部署部分。

#### 1.1.2 K8S

>暂无

### 1.2 客户端更新

#### 1.2.1 代码

##### 1.2.1.1  服务注册

>JAVA 后端SDK，内置nacos默认用户密码集成：

```java 
...
       Properties props = System.getProperties();
        // 通用注册
        props.setProperty("spring.cloud.nacos.discovery.server-addr", LauncherConstant.nacosAddr(profile));
        props.setProperty("spring.cloud.nacos.config.server-addr", LauncherConstant.nacosAddr(profile));
        props.setProperty("spring.cloud.nacos.config.shared-dataids", LauncherConstant.sharedDataIds(profile));
        props.setProperty("spring.cloud.nacos.config.refreshable-dataids", LauncherConstant.sharedDataIds(profile));
        props.setProperty("spring.cloud.nacos.username", "nacos");
        props.setProperty("spring.cloud.nacos.password", "nacos");
        builder.properties(props);
...
```

##### 1.2.1.2 Dockerfile

>JAVA 后端，Dockerfile默认启动参数集成：
>**防止后期Nacos 服务端内置密码修改后，JAVA侧内置密码失效无法正常注册**

```Dockerfile
FROM harbor.dockerregistry.com/app/adoptopenjdk/openjdk8-openj9:alpine-slim-local

WORKDIR /home

ADD ./target/*.jar ./app.jar


ENTRYPOINT [ \
    "java", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-Xshareclasses", \
    "-Xquickstart", \
    "-jar", \
    "app.jar", \
    "--spring.profiles.active=dev", \
    "--spring.cloud.nacos.discovery.server-addr=nacos-server:8848", \
    "--spring.cloud.nacos.config.server-addr=nacos-server:8848" , \
    "--spring.cloud.nacos.discovery.username=nacos", \
	"--spring.cloud.nacos.discovery.password=nacos", \
    "--spring.cloud.nacos.config.namespace=mynamespace", \
    "--spring.cloud.nacos.discovery.namespace=mynamespace" \
    ]
```

#### 1.2.2 部署

> 根据自己的部署模式来进行调整即可。

##### 1.2.2.1 Compose

>Docker-compose 启动的JAVA 后端服务，环境变量修改：
>**旧的服务配置：**

```yaml
version: '3'
x-service-template: &x-service-template
  restart: always
  extra_hosts:
    - "nacos-sysplat:10.3.2.36"
  environment:
    - SPRING_PROFILES_ACTIVE=dev
  volumes:
    - /etc/localtime:/etc/localtime:ro
  networks:
    - app

services:
  sys-gateway:
    <<: *x-service-template
    image: harbor.dockerregistry.com/gd-sysplat-base-dev/sys-gateway:9d4c146
    container_name: sys-gateway
    ports:
      - 80:80
      #- 443:443
  sys-system:
    <<: *x-service-template
    image: harbor.dockerregistry.com/gd-sysplat-base-dev/sys-system:9d4c146
    container_name: sys-system
    ports:
      - 8106:8106
...
```

>**新的服务配置：**

```yaml
version: '3'
x-service-template: &x-service-template
  restart: always
  extra_hosts:
    - "nacos-sysplat:10.3.2.36"
  environment:
    - SPRING_PROFILES_ACTIVE=dev
    - SPRING_CLOUD_NACOS_USERNAME=new_username
    - SPRING_CLOUD_NACOS_PASSWORD=new_password
  volumes:
    - /etc/localtime:/etc/localtime:ro
  networks:
    - app

services:
  sys-gateway:
    <<: *x-service-template
    image: harbor.dockerregistry.com/gd-sysplat-base-dev/sys-gateway:9d4c146
    container_name: sys-gateway
    ports:
      - 80:80
      #- 443:443
  sys-system:
    <<: *x-service-template
    image: harbor.dockerregistry.com/gd-sysplat-base-dev/sys-system:9d4c146
    container_name: sys-system
    ports:
      - 8106:8106
...
```

##### 1.2.2.2 K8S

>K8S 管理的JAVA 后端服务，环境变量修改：
>**旧的资源配置：**

```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-model
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: service-model
    spec:
      volumes:
        - name: host-time
          hostPath:
            path: /etc/localtime
            type: ''
      containers:
        - name: service-model
          image: 'harbor.dockerregistry.com/gd-sysplat-base-dev/service-model:62c306d'
          ports:
            - name: tcp-8040
              containerPort: 8040
              protocol: TCP
          resources: {}
          volumeMounts:
            - name: host-time
              readOnly: true
              mountPath: /etc/localtime
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: Always
```


>新的资源配置：

```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-model
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: service-model
    spec:
      volumes:
        - name: host-time
          hostPath:
            path: /etc/localtime
            type: ''
      containers:
        - name: service-model
          image: 'harbor.dockerregistry.com/gd-sysplat-base-dev/service-model:62c306d'
          env:
            - name: SPRING_CLOUD_NACOS_USERNAME
              value: new_username
            - name: SPRING_CLOUD_NACOS_PASSWORD
              value: new_password
          ports:
            - name: tcp-8040
              containerPort: 8040
              protocol: TCP
          resources: {}
          volumeMounts:
            - name: host-time
              readOnly: true
              mountPath: /etc/localtime
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: Always
```

##### 1.2.2.3 Docker


>Docker RUN 直接管理的JAVA 后端容器服务，相应命令行参数添加：
>**旧的命令：**

```shell
		docker run -d  --name=$project_name      \
		--memory-swap -1   --restart=always      \
		-v /etc/localtime:/etc/localtime:ro      \
		-v /original:/original                   \
		-v /temporary:/temporary                 \
		-v /management:/management               \
		-v /achievement:/achievement             \
		-v /usr/share/fonts:/usr/share/fonts     \
		--add-host=server157.esri.com:10.0.1.157 \
		--add-host=server156.esri.com:10.0.1.156 \
        --add-host=gjptgis.arcgis.cn:10.1.6.115  \
		--add-host=$nacos_env:$nacos_ip          \
		--net=host                               \
		-e "SPRING_PROFILES_ACTIVE=$service_env" \
		harbor.dockerregistry.com/gd-land-dev/$project_name:$image_tag
```


>**新的命令：**

```shell
docker run -d  --name=$project_name      \
--memory-swap -1   --restart=always      \
-v /etc/localtime:/etc/localtime:ro      \
-v /original:/original                   \
-v /temporary:/temporary                 \
-v /management:/management               \
-v /achievement:/achievement             \
-v /usr/share/fonts:/usr/share/fonts     \
--add-host=server157.esri.com:10.0.1.157 \
--add-host=server156.esri.com:10.0.1.156 \
--add-host=gjptgis.arcgis.cn:10.1.6.115  \
--add-host=$nacos_env:$nacos_ip          \
--net=host                               \
-e "SPRING_PROFILES_ACTIVE=$service_env" \
-e "SPRING_CLOUD_NACOS_USERNAME=new_username" \
-e "SPRING_CLOUD_NACOS_PASSWORD=new_password" \
harbor.dockerregistry.com/gd-land-dev/$project_name:$image_tag
```

##### 1.2.2.4 JAR

>Java -jar 直接使用JAVA 环境，运行JVM虚拟机命令修改：
>**旧的命令：**

```shell
java -jar app.jar \
--spring.profiles.active=dev  \
--spring.cloud.nacos.discovery.server-addr=nacos-server:8848 \
--spring.cloud.nacos.config.server-addr=nacos-server:8848    \
--spring.cloud.nacos.config.namespace=mynamespace            \
--spring.cloud.nacos.discovery.namespace=mynamespace
```

>**新的命令：**

```shell
java -jar app.jar \
--spring.profiles.active=dev  \
--spring.cloud.nacos.username=nacos \
--spring.cloud.nacos.password=nacos \
--spring.cloud.nacos.discovery.server-addr=nacos-server:8848 \
--spring.cloud.nacos.config.server-addr=nacos-server:8848    \
--spring.cloud.nacos.config.namespace=mynamespace            \
--spring.cloud.nacos.discovery.namespace=mynamespace
```





## Compose file(only nacos)

```yaml
version: '3'
services:
  nacos:
    image: harbor.dockerregistry.com/app-arm/nacos/nacos-server:v2.2.3-slim
    container_name: nacos
    ports:
      - "8848:8848"
      - "9848:9848"
      - "9849:9849"
      - "7848:7848"
    environment:
      NACOS_AUTH_ENABLE: 'true'
    env_file:
      - ./nacos-local/env/nacos-standlone-mysql.env
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./nacos-local/init.d/application.properties:/home/nacos/conf/application.properties
      - /data/middleware/nacos/standalone-logs/:/home/nacos/logs
      - /data/middleware/nacos/standalone-data/:/home/nacos/data
    restart: always
```

