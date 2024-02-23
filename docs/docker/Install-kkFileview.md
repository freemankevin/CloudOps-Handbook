

## Compose file

```yaml
version: '3'
services:
  kkfileview:
   image: harbor.dockerregistry.com/app/kkfileview:379a272
   container_name: kkfileview
   ports:
     - 8012:8012
   volumes:
     - /etc/localtime:/etc/localtime:ro
     - ./application.properties:/opt/kkFileView-4.1.0-RELEASE/config/application.properties
   restart: always
```



## application.properties

```shell
server.port = ${KK_SERVER_PORT:8012}
server.servlet.context-path= ${KK_CONTEXT_PATH:/}
server.servlet.encoding.charset = utf-8
spring.servlet.multipart.max-file-size=500MB
spring.servlet.multipart.max-request-size=500MB
spring.freemarker.template-loader-path = classpath:/web/
spring.freemarker.cache = false
spring.freemarker.charset = UTF-8
spring.freemarker.check-template-location = true
spring.freemarker.content-type = text/html
spring.freemarker.expose-request-attributes = true
spring.freemarker.expose-session-attributes = true
spring.freemarker.request-context-attribute = request
spring.freemarker.suffix = .ftl
office.plugin.server.ports = 2001,2002
office.plugin.task.timeout = 5m
file.dir = ${KK_FILE_DIR:default}
office.home = ${KK_OFFICE_HOME:default}
cache.type =  ${KK_CACHE_TYPE:redis}
spring.redisson.address = ${KK_SPRING_REDISSON_ADDRESS:127.0.0.1:6379}
spring.redisson.password = ${KK_SPRING_REDISSON_PASSWORD:admin@123}
cache.clean.enabled = ${KK_CACHE_CLEAN_ENABLED:true}
cache.clean.cron = ${KK_CACHE_CLEAN_CRON:0 0 3 * * ?}
base.url = ${KK_BASE_URL:default}
trust.host = ${KK_TRUST_HOST:default}
cache.enabled = ${KK_CACHE_ENABLED:true}
simText = ${KK_SIMTEXT:txt,html,htm,asp,jsp,xml,json,properties,md,gitignore,log,java,py,c,cpp,sql,sh,bat,m,bas,prg,cmd}
media = ${KK_MEDIA:mp3,wav,mp4,flv}
media.convert.disable = ${KK_MEDIA_CONVERT_DISABLE:false}
convertMedias = ${KK_CONVERTMEDIAS:avi,mov,wmv,mkv,3gp,rm}
office.preview.type = ${KK_OFFICE_PREVIEW_TYPE:image}
office.preview.switch.disabled = ${KK_OFFICE_PREVIEW_SWITCH_DISABLED:false}
pdf.download.disable = ${KK_PDF_DOWNLOAD_DISABLE:true}
ftp.username = ${KK_FTP_USERNAME:ftpuser}
ftp.password = ${KK_FTP_PASSWORD:123456}
ftp.control.encoding = ${KK_FTP_CONTROL_ENCODING:UTF-8}
watermark.txt = ${WATERMARK_TXT:}
watermark.x.space = ${WATERMARK_X_SPACE:10}
watermark.y.space = ${WATERMARK_Y_SPACE:10}
watermark.font = ${WATERMARK_FONT:微软雅黑}
watermark.fontsize = ${WATERMARK_FONTSIZE:18px}
watermark.color = ${WATERMARK_COLOR:black}
watermark.alpha = ${WATERMARK_ALPHA:0.2}
watermark.width = ${WATERMARK_WIDTH:180}
watermark.height = ${WATERMARK_HEIGHT:80}
watermark.angle = ${WATERMARK_ANGLE:10}
```

