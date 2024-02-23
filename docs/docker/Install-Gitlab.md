

## Compose file

参考博客：[https://soulteary.com/2021/07/14/gitlab-14-lightweight-operation-solution.html](https://soulteary.com/2021/07/14/gitlab-14-lightweight-operation-solution.html)



> 应该需要root 用户启动才行，内置数据库目前权限在提权用户下有点问题。



```yaml
version: "3"


services:
  gitlab:
    restart: always
    image: gitlab/gitlab-ce:15.11.13-ce.0
    container_name: gitlab
    hostname: gitlab.localhost.com
    ports:
      - "80:80"
      - '443:443' 
      - "22:22"
    volumes:
      - ./config:/etc/gitlab
      - ./data:/var/opt/gitlab
    environment:
      TZ: Asia/Shanghai
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.localhost.com'
        gitlab_rails['time_zone'] = 'Asia/Shanghai'

        # 关闭电子邮件相关功能
        gitlab_rails['smtp_enable'] = false
        gitlab_rails['gitlab_email_enabled'] = false
        gitlab_rails['incoming_email_enabled'] = false

        # Terraform
        gitlab_rails['terraform_state_enabled'] = false

        # Usage Statistics
        gitlab_rails['usage_ping_enabled'] = false
        gitlab_rails['sentry_enabled'] = false
        grafana['reporting_enabled'] = false

        # 关闭容器仓库功能
        gitlab_rails['gitlab_default_projects_features_container_registry'] = false
        gitlab_rails['registry_enabled'] = false
        registry['enable'] = false
        registry_nginx['enable'] = false

        # 包仓库
        gitlab_rails['packages_enabled'] = false
        gitlab_rails['dependency_proxy_enabled'] = false

        # GitLab KAS
        gitlab_kas['enable'] = false
        gitlab_rails['gitlab_kas_enabled'] = false

        # Mattermost
        mattermost['enable'] = false
        mattermost_nginx['enable'] = false

        # Kerberos
        gitlab_rails['kerberos_enabled'] = false
        sentinel['enable'] = false

        # GitLab Pages
        gitlab_pages['enable'] = false
        pages_nginx['enable'] = false

        # 禁用 PUMA 集群模式
        puma['worker_processes'] = 0
        puma['min_threads'] = 1
        puma['max_threads'] = 2

        # 降低后台守护进程并发数
        sidekiq['max_concurrency'] = 5

        gitlab_ci['gitlab_ci_all_broken_builds'] = false
        gitlab_ci['gitlab_ci_add_pusher'] = false

        # 关闭监控
        prometheus_monitoring['enable'] = false
        alertmanager['enable'] = false
        node_exporter['enable'] = false
        redis_exporter['enable'] = false
        postgres_exporter['enable'] = false
        pgbouncer_exporter['enable'] = false
        gitlab_exporter['enable'] = false
        grafana['enable'] = false
        sidekiq['metrics_enabled'] = false 
```

```shell
docker-compose up -d
chmod 0600 conf/ssh_host_ed25519_key
docker-compose down
docker-compose up -d 
```





## Password acquisition


```shell
# root
docker exec -it gitlab-ce grep 'Password:' /etc/gitlab/initial_root_password

# or
docker exec -it gitlab-ce grep 'admin_password' /etc/gitlab/gitlab-secrets.json
```

