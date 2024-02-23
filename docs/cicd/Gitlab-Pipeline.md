



## 1 `Gitlab `添加内网信任

**目的：**

1. 内网域名或地址默认被限制的。



**操作过程**：

1. 进入管理员页面。

2. 进入设置。

3. 进入网络。

4. 进入出站请求。

5. 勾选选项：

   ☑️允许来自 `webhooks `和集成对本地网络的请求

   ☑️允许系统钩子向本地网络发送请求

6. 在空白框中，粘贴需要调用的内网IP、域名。

7. 登录`Gitlab` 服务器，添加HOSTS 自定义域名映射。

```shell
192.168.1.120 traefik.k8scluster.com
192.168.1.120 harbor.dockerregistry.com
192.168.1.120 argocd.k8scluster.com
192.168.1.120 grpc.argocd.k8scluster.com
```









## 2 `Gitlab `集成 `Argocd`

**目的：**

1. 方便流水线中隐藏`Argocd `真实服务器信息同时可以操作应用。



**操作过程：**

1. 进入项目组页面。

2. 进入设置。

3. 进入CICD。

4. 进入变量。

5. 添加变量：

   | Type     | Key             | Value  | Options | Environments  |
   | -------- | --------------- | ------ | ------- | ------------- |
   | Variable | ARGOCD_USERNAME | 自定义 | Masked  | All (default) |
   | Variable | ARGOCD_SERVER   | 自定义 | Masked  | All (default) |
   | Variable | ARGOCD_PASSWORD | 自定义 | Masked  | All (default) |

   

## 3 `Gitlab `集成 `Minio`

**目的：**

1. 将制品上传到`Minio`中，减轻`Gitlab `服务器维护压力。



**操作过程：**

1. 修改 `Gitlab`服务器配置

```shell
# 修改配置
sudo vim /etc/gitlab/gitlab.rb
gitlab_rails['artifacts_enabled'] = true
gitlab_rails['artifacts_object_store_enabled'] = true
gitlab_rails['artifacts_object_store_remote_directory'] = 'gitlab-artifacts'
gitlab_rails['artifacts_object_store_connection'] = {
'provider' => 'AWS',
'region' => 'us-east-1',
'aws_access_key_id' => '08JbSVpoWEYc1fc8xFnf',
'aws_secret_access_key' => 'JECzRcjqXTa5DVk6ZsdOFWBcfZv32HFYV4arNUD4',
'endpoint' => 'http://192.168.1.120:9000',
'path_style' => true
}
     
# 重载配置
sudo gitlab-ctl reconfigure
```



## 4 `Gitlab `集成 `Harbor`

**目的：**

1. 将镜像文件制作完成后直接推送到Harbor ，不再需要重复配置。



**操作过程**：

1. 进入管理员，或者项目组管理界面。

2. 进入设置。

3. 进入集成。

4. 添加Harbor 集成。

5. 输入（用户信息建议使用机器人名称和密码）：

   - 域名
   - 项目名称
   - 用户
   - 用户密码

   



## 5 `Gitlab ` 文件大小限制

**目的：**

1. 流水线任务中，上传制品到 `Minio` 或者暂存`Gitlab`时默认被限制为 100m 或者 200m, 而实际远大于这些，需要调高限制以满足实际需要。



**操作过程**：

1. 进入管理员或者项目组或者项目页面（具体根据自己的权限来定）。
2. 进入设置。
3. 进入CICD。
4. 进入流水线通用设置。
5. 修改最大工件大小，具体按自己需要修改。
6. 若修改了上面配置还是不行，可以检查下 `Gitlab`服务器配置，可能需要同步更新下。

```shell
# 修改配置
sudo vim /etc/gitlab/gitlab.rb
nginx['client_max_body_size'] = '200m'

# 重载配置
sudo gitlab-ctl reconfigure
```





## 6 `Gitlab` 集成 CICD

目的：

1. 让机器人有权限自动更新指定部分的代码。



操作过程：

1. 进入项目页面。
2. 进入设置。
3. 进入访问令牌，创建一个长期有效的开发者身份的令牌。

4. 进入CICD。
5. 进入变量，创建一个变量，具体如：

| Type     | Key(Click to sort descending) | Value  | Options | Environments  |
| -------- | ----------------------------- | ------ | ------- | ------------- |
| Variable | GITLAB_CI_TOKEN               | 自定义 | Masked  | All (default) |





## 7 Pipeline(backend)

```yaml
image: $HARBOR_HOST/kubernetes/maven:3.8.6-openjdk-8-slim
# image: maven:3.8.6-openjdk-8-slim

variables:
  AUTOMATIC_TRIGGER: "off"
  MAVEN_CLI_OPTS: "-B -U -s conf/settings.xml"
  BUILD_GOAL: "clean package"
  DEPLOY_ONLY: "false"
  DEST_ENVIRONMENT: "dev"
  REPO_URL: http://gitlab-ci-token:${GITLAB_CI_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git # Access Tokens & write_repository & CI/CD Variables
  
stages:
  - package
  - build
  - deploy
    
package:
  stage: package
  script:
    - mvn $MAVEN_CLI_OPTS $BUILD_GOAL
  cache:
    key: "${CI_COMMIT_REF_SLUG}"
    paths:
      - .m2/repository
  artifacts:
    paths:
      - target/*.jar
  tags:
    - kubernetes
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      when: manual 

    - if: '$DEPLOY_ONLY != "true"' 

build:
  image:
    name: $HARBOR_HOST/kubernetes/kaniko-project/executor:v1.14.0-debug
    # name: gcr.io/kaniko-project/executor:v1.14.0-debug
    entrypoint: [""]
  stage: build
  before_script:
    - echo "Logging in to Docker registry $HARBOR_HOST"
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$HARBOR_HOST\":{\"username\":\"$HARBOR_USERNAME\",\"password\":\"$HARBOR_PASSWORD\"}}}" > /kaniko/.docker/config.json
  script:
    - echo "Building and pushing Docker image"
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "$HARBOR_HOST/$HARBOR_PROJECT/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA"
      --cache=true
      --cache-ttl=24h
      --skip-tls-verify
      --build-arg HARBOR_HOST=${HARBOR_HOST}
  needs: ["package"]
  tags:
    - kubernetes
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      when: manual
    - if: '$DEPLOY_ONLY != "true"' 
      
update_config:
  image:
    name: $HARBOR_HOST/kubernetes/argocli-git:v2.7.14
  stage: deploy
  script:
    - echo "Updating Kustomize configs in Git with the latest commit SHA"
    - >-
      git checkout main &&
      cd k8s/overlays/$DEST_ENVIRONMENT/ &&
      sed "s/\${CI_COMMIT_SHORT_SHA}/\"${CI_COMMIT_SHORT_SHA}\"/g" kustomization.yaml.template > kustomization.yaml &&
      git add kustomization.yaml &&
      git commit -m "Update the image tag to ${CI_COMMIT_SHORT_SHA} [skip ci]" &&
      git push $REPO_URL
  needs: ["build"]
  tags:
    - kubernetes
  # rules:
  #   - if: $AUTOMATIC_TRIGGER == "off"
  #     when: manual

sync_application:
  image:
    name: $HARBOR_HOST/kubernetes/argocli-git:v2.7.14
  stage: deploy
  script:
    - echo "Logging in to ArgoCD" 
    - >-
      export APP_NAME="$CI_PROJECT_NAME" &&
      echo y | argocd login $ARGOCD_SERVER --password "$ARGOCD_PASSWORD" --username $ARGOCD_USERNAME
    - echo "Syncing application with ArgoCD"
    - >- 
      if argocd app list | grep -q $APP_NAME; then
        echo "Application $APP_NAME exists. Syncing..."
        argocd app sync $APP_NAME --prune
      else
        echo "Application $APP_NAME does not exist. Creating..."
        argocd app create $APP_NAME \
        --repo $REPO_URL \
        --path k8s/overlays/$DEST_ENVIRONMENT \
        --dest-server https://kubernetes.default.svc \
        --sync-policy automated
      fi
  needs: ["update_config"]
  environment:
    name: $CI_COMMIT_REF_NAME
    url: https://myargocd.mydomain.com:32271/kubernetes/cluster/namespaces/$NAMESPACE_NAME
  tags:
    - kubernetes
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      when: manual
```





## 8 Pipeline(frontend)

```yaml
image: $HARBOR_HOST/kubernetes/node:16.20.2

variables:
  DEPLOY_ONLY: "false"
  DEST_ENVIRONMENT: "prod"
  REPO_URL: http://gitlab-ci-token:${GITLAB_CI_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git

stages:
  - check
  - package
  - build
  - deploy

check:
  stage: check
  script:
    - echo "Checking Node.js version..."
    - node --version
    - echo "Checking npm version..."
    - npm --version
    - echo "Checking Yarn version..."
    - yarn --version
  tags:
    - kubernetes
  rules:
    - if: '$DEPLOY_ONLY != "true"'

package:
  stage: package
  script:
    - if grep -q node-sass package.json; then yarn remove node-sass && yarn add sass; fi
    - npm install --force --registry=https://registry.npm.taobao.org --loglevel info
    - npm run build
  artifacts:
    paths:
      - dist/
  cache:
    key: "$CI_COMMIT_REF_NAME"
    paths:
      - node_modules/
  tags:
    - kubernetes
  rules:
    - if: '$DEPLOY_ONLY != "true"'

build:
  image:
    name: $HARBOR_HOST/kubernetes/kaniko-project/executor:v1.14.0-debug
    entrypoint: [""]
  stage: build
  before_script:
    - echo "Logging in to Docker registry $HARBOR_HOST"
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$HARBOR_HOST\":{\"username\":\"$HARBOR_USERNAME\",\"password\":\"$HARBOR_PASSWORD\"}}}" > /kaniko/.docker/config.json
  script:
    - echo "Building and pushing Docker image"
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "$HARBOR_HOST/$HARBOR_PROJECT/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA"
      --cache=true
      --cache-ttl=24h
      --skip-tls-verify
      --build-arg HARBOR_HOST=${HARBOR_HOST}
  needs: ["package"]
  tags:
    - kubernetes
  rules:
    - if: '$DEPLOY_ONLY != "true"'

update_config:
  image:
    name: $HARBOR_HOST/kubernetes/argocli-git:v2.7.14
  stage: deploy
  script:
    - echo "Updating Kustomize configs in Git with the latest commit SHA"
    - >-
      git checkout main &&
      cd k8s/overlays/$DEST_ENVIRONMENT/ &&
      sed "s/\${CI_COMMIT_SHORT_SHA}/\"${CI_COMMIT_SHORT_SHA}\"/g" kustomization.yaml.template > kustomization.yaml &&
      git add kustomization.yaml &&
      git commit -m "Update the image tag to ${CI_COMMIT_SHORT_SHA} [skip ci]" &&
      git push $REPO_URL
  needs: ["build"]
  tags:
    - kubernetes

sync_application:
  image:
    name: $HARBOR_HOST/kubernetes/argocli-git:v2.7.14
  stage: deploy
  script:
    - echo "Logging in to ArgoCD" 
    - >-
      export APP_NAME="$CI_PROJECT_NAME" &&
      echo y | argocd login $ARGOCD_SERVER --password $ARGOCD_PASSWORD --username $ARGOCD_USERNAME
    - echo "Syncing application with ArgoCD"
    - >- 
      if argocd app list | grep -q $APP_NAME; then
        echo "Application $APP_NAME exists. Syncing..."
        argocd app sync $APP_NAME --prune
      else
        echo "Application $APP_NAME does not exist. Creating..."
        argocd app create $APP_NAME \
        --repo $REPO_URL \
        --path k8s/overlays/$DEST_ENVIRONMENT \
        --dest-server https://kubernetes.default.svc \
        --sync-policy automated
      fi
  needs: ["update_config"]
  environment:
    name: $CI_COMMIT_REF_NAME
    url: https://$ARGOCD_SERVER/kubernetes/cluster/namespaces/$NAMESPACE_NAME
  tags:
    - kubernetes
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      when: manual
```



## 9 Pipeline + template(Backend)

```yaml
include: '.gitlab-ci-template.yml'

image: $HARBOR_HOST/kubernetes/maven:3.6.3-openjdk-8-slim

variables:
  DEST_ENVIRONMENT: 'dev'
  HARBOR_PROJECT: 'papd-bom'
  APP1_NAME: 'cloud-gateway'
  APP2_NAME: 'cloud-auth'
  APP3_NAME: 'cloud-system'
  APP1_NAME_PATH: 'cloud-services/cloud-gateway'
  APP2_NAME_PATH: 'cloud-services/cloud-auth'
  APP3_NAME_PATH: 'cloud-services/cloud-system'
  APP1_JAR_NAME: 'sys-gateway-1.1.0.jar'
  APP2_JAR_NAME: 'sys-auth-1.1.0.jar'
  APP3_JAR_NAME: 'sys-system-1.1.0.jar'

stages:
  - install
  - package
  - build
  - update
  - deploy

################################################# local installing ################################
maven-install:
  extends: .install_template


################################################# local packaging #################################
package-cloud-gateway:
  extends: .package_template
  variables:
    MODULE_PATH: $APP1_NAME_PATH
    ARTIFACT_NAME: $APP1_JAR_NAME

package-cloud-auth:
  extends: .package_template
  variables:
    MODULE_PATH: $APP2_NAME_PATH
    ARTIFACT_NAME: $APP2_JAR_NAME

package-cloud-system:
  extends: .package_template
  variables:
    MODULE_PATH: $APP3_NAME_PATH
    ARTIFACT_NAME: $APP3_JAR_NAME

################################################# image creation ##############################
build-cloud-gateway:
  extends: .build_template
  variables:
    CI_PROJECT_NAME: $APP1_NAME
    CONTEXT_PATH: $APP1_NAME_PATH
  needs: ["package-cloud-gateway"]

build-cloud-auth:
  extends: .build_template
  variables:
    CI_PROJECT_NAME: $APP2_NAME
    CONTEXT_PATH: $APP2_NAME_PATH
  needs: ["package-cloud-auth"]

build-cloud-system:
  extends: .build_template
  variables:
    CI_PROJECT_NAME: $APP3_NAME
    CONTEXT_PATH: $APP3_NAME_PATH
  needs: ["package-cloud-system"]

################################################# update label #############################

update-label-cloud-gateway:
  extends: .update_template
  variables:
    CONTEXT_PATH: $APP1_NAME_PATH
  needs: ["build-cloud-gateway"]


update-label-cloud-auth:
  extends: .update_template
  variables:
    CONTEXT_PATH: $APP2_NAME_PATH
  needs: ["build-cloud-auth"]


update-label-cloud-system:
  extends: .update_template
  variables:
    CONTEXT_PATH: $APP3_NAME_PATH
  needs: ["build-cloud-system"]

################################################# app sync #########################

deploy-cloud-gateway:
  extends: .sync_application_template
  variables:
    CI_PROJECT_NAME: $APP1_NAME
    CONTEXT_PATH: $APP1_NAME_PATH
  needs: ["update-label-cloud-gateway"]

deploy-cloud-auth:
  extends: .sync_application_template
  variables:
    CI_PROJECT_NAME: $APP2_NAME
    CONTEXT_PATH: $APP2_NAME_PATH
  needs: ["update-label-cloud-auth"]

deploy-cloud-system:
  extends: .sync_application_template
  variables:
    CI_PROJECT_NAME: $APP3_NAME
    CONTEXT_PATH: $APP3_NAME_PATH
  needs: ["update-label-cloud-system"]
```



> .gitlab-ci-template.yml

```yaml
variables:
  AUTOMATIC_TRIGGER: "off"
  MAVEN_CLI_OPTS: "-B -U -s $CI_PROJECT_DIR/conf/settings.xml"
  INSTALL_GOAL: "clean install -DskipTests"
  BUILD_GOAL: "clean package -DskipTests"
  DEPLOY_ONLY: "false"
  REPO_URL: http://gitlab-ci-token:${GITLAB_CI_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git # Access Tokens & write_repository & CI/CD Variables

.install_template: # Define local packaging template
  stage: install
  script:
    - mvn $MAVEN_CLI_OPTS $INSTALL_GOAL
  cache:
    key: "${CI_COMMIT_REF_SLUG}"
    paths:
      - $CI_PROJECT_DIR/.m2/repository
    #  artifacts:
    #    expire_in: 1 day
    #    paths:
    #      - $CI_PROJECT_DIR/$GATEWAY_PATH/target/sys-gateway-5.1.0.jar
    #      - $CI_PROJECT_DIR/$AUTH_PATH/target/sys-auth-5.1.0.jar
    #      - $CI_PROJECT_DIR/$SYSTEM_PATH/target/sys-system-5.1.0.jar
  tags:
    - kubernetes
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      when: manual
    - if: '$DEPLOY_ONLY != "true"'

################################################# local packaging #################################################

.package_template: # Define local packaging template
  stage: package
  script:
    - cd $CI_PROJECT_DIR/${MODULE_PATH}
    - mvn $MAVEN_CLI_OPTS $BUILD_GOAL
  cache:
    key: "${CI_COMMIT_REF_SLUG}"
    paths:
      - $CI_PROJECT_DIR/.m2/repository
  artifacts:
    expire_in: 1 day
    paths:
      - $CI_PROJECT_DIR/${MODULE_PATH}/target/${ARTIFACT_NAME}
  tags:
    - kubernetes
  needs: ["maven-install"]
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      when: manual
#    - if: '$DEPLOY_ONLY != "true"'

################################################# image creation #################################################

.build_template: # Define image creation template
  image:
    name: $HARBOR_HOST/kubernetes/kaniko-project/executor:v1.14.0-debug
    entrypoint: [""]
  stage: build
  before_script:
    - echo "Logging in to Docker registry $HARBOR_HOST"
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$HARBOR_HOST\":{\"username\":\"$HARBOR_USERNAME\",\"password\":\"$HARBOR_PASSWORD\"}}}" > /kaniko/.docker/config.json
  script:
    - echo "Building and pushing Docker image"
    - >-
      /kaniko/executor
      --context "$CI_PROJECT_DIR/${CONTEXT_PATH}"
      --dockerfile "$CI_PROJECT_DIR/${CONTEXT_PATH}/Dockerfile"
      --destination "$HARBOR_HOST/$HARBOR_PROJECT/$DEST_ENVIRONMENT/${CI_PROJECT_NAME}:${CI_COMMIT_SHORT_SHA}"
      --cache=true
      --cache-ttl=24h
      --skip-tls-verify
      --build-arg HARBOR_HOST=${HARBOR_HOST}
  tags:
    - kubernetes
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      # when: manual
#    - if: '$DEPLOY_ONLY != "true"'

################################################# update label #################################################
.update_template: # Define update label template
  image:
    name: $HARBOR_HOST/kubernetes/argocli-git:v2.7.14
  stage: update
  script:
    - printenv
    - echo "Updating Kustomize configs in Git with the latest commit SHA"
    - echo "The current branch is $CI_COMMIT_REF_NAME"
    - echo "The latest commit is $CI_COMMIT_SHA"
    - git fetch origin $CI_COMMIT_REF_NAME
    - git reset --hard FETCH_HEAD
    - |
      cd $CI_PROJECT_DIR/${CONTEXT_PATH}/k8s/overlays/$DEST_ENVIRONMENT/
      sed "s/\${CI_COMMIT_SHORT_SHA}/\"${CI_COMMIT_SHORT_SHA}\"/g" kustomization.yaml.template > kustomization.yaml
      git add kustomization.yaml
      git commit -m "Update the image tag to ${CI_COMMIT_SHORT_SHA} [skip ci]" || echo "No changes to commit"
    - |
      attempt=0
      max_attempts=3
      until git push $REPO_URL HEAD:$CI_COMMIT_REF_NAME; do
        attempt=$((attempt+1))
        if [ "$attempt" -gt "$max_attempts" ]; then
          echo "Maximum push attempts reached. No more attempts left."
          exit 1
        fi
        echo "Push attempt $attempt failed. Attempting to pull and resolve conflicts..."
        git pull --rebase $REPO_URL $CI_COMMIT_REF_NAME
        echo "Retrying push. Attempt $attempt of $max_attempts."
        sleep 5
      done
  tags:
    - kubernetes
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      # when: manual
#    - if: '$DEPLOY_ONLY != "true"'

################################################# app sync #################################################
.sync_application_template: # Define app sync templates
  image:
    name: $HARBOR_HOST/kubernetes/argocli-git:v2.7.14
  stage: deploy
  script:
    - echo "Logging in to ArgoCD"
    - >-
      export APP_NAME="${CI_PROJECT_NAME}-${HARBOR_PROJECT}-${DEST_ENVIRONMENT}" &&
      echo y | argocd login $ARGOCD_SERVER --password "$ARGOCD_PASSWORD" --username $ARGOCD_USERNAME
    - echo "Syncing application with ArgoCD"
    - >-
      if argocd app list | grep -q $APP_NAME; then
        echo "Application $APP_NAME exists. Syncing..."
        argocd app sync $APP_NAME --prune
      else
        echo "Application $APP_NAME does not exist. Creating..."
        argocd app create $APP_NAME \
        --repo $REPO_URL \
        --revision $CI_COMMIT_REF_NAME \
        --path $CONTEXT_PATH/k8s/overlays/$DEST_ENVIRONMENT \
        --dest-server https://kubernetes.default.svc \
        --sync-policy automated \
        --loglevel debug
      fi
  tags:
    - kubernetes
  rules:
    - if: $AUTOMATIC_TRIGGER == "off"
      when: manual
  environment:
    name: $DEST_ENVIRONMENT
    url: https://$ARGOCD_SERVER/kubernetes/cluster/namespaces/$NAMESPACE_NAME
```





## 10 Pipeline + template(frontend)

```yaml
include: '.gitlab-ci-template.yml'

image: $HARBOR_HOST/kubernetes/nodejs:v4

variables:
  DEST_ENVIRONMENT: 'prod'
  CI_PROJECT_NAME: 'frame-ui'
  HARBOR_PROJECT: 'papd-bom'

stages:
  - check
  - package
  - build
  - update
  - deploy

######################################################### check env #########################################################
check:
  extends: .check_template

######################################################### packaged  #########################################################
package:
  extends: .package_template
  script:
    # Replace node-sass with sass if it exists in package.json
    - if grep -q node-sass package.json; then pnpm remove node-sass && pnpm add sass; fi
    # Use taobao repository
    - pnpm install --force --registry=https://registry.npmmirror.com/ --loglevel info
    # Pull subproject
    - git submodule update --init -- src/projects/system
    - git submodule update --remote -- src/projects/system
    # Build the project
    - pnpm run build:system
  artifacts:
    paths:
      - sys/

######################################################### build images #########################################################
build:
  extends: .build_template

##################################################### update labels #########################################################
update-label:
  extends: .update_template

#################################################### deploy app #########################################################
deploy-app:
  extends: .sync_application_template

```



> .gitlab-ci-template.yml

```yaml
variables:
  DEPLOY_ONLY: 'false'
  REPO_URL: http://gitlab-ci-token:${GITLAB_CI_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git

# _template
######################################################### check env #########################################################
.check_template: # Define template for local environment checks
  stage: check
  script:
    - echo "Checking Node.js version..."
    - node --version
    - echo "Checking npm version..."
    - npm --version
    - echo "Checking Yarn version..."
    - yarn --version
    - echo "Checking pnpm version..."
    - pnpm --version
  tags:
    - kubernetes
  rules:
    - if: '$DEPLOY_ONLY != "true"'
    # - when: manual
    #   allow_failure: false

######################################################### packaged #########################################################

.package_template: # Define packaged template
  stage: package
  script:
    # Replace node-sass with sass if it exists in package.json
    - if grep -q node-sass package.json; then pnpm remove node-sass && pnpm add sass; fi
    # Use taobao repository
    - pnpm install --force --registry=https://registry.npmmirror.com/ --loglevel info
    # Pull subproject
    # - pnpm run pullGit system
    - git submodule update --init -- src/projects/system
    - git submodule update --remote -- src/projects/system
    # Build the project
    - pnpm run build:system
  artifacts:
    paths:
      - sys/
    expire_in: 1 day
  cache:
    key: '$CI_COMMIT_REF_NAME'
    paths:
      - node_modules/
  needs: ['check']
  tags:
    - kubernetes
  rules:
    - when: manual
      allow_failure: false

################################################ build images #########################################################
.build_template: # Define the template for making images
  image:
    name: $HARBOR_HOST/kubernetes/kaniko-project/executor:v1.14.0-debug
    entrypoint: ['']
  stage: build
  before_script:
    - echo "Logging in to Docker registry $HARBOR_HOST"
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$HARBOR_HOST\":{\"username\":\"$HARBOR_USERNAME\",\"password\":\"$HARBOR_PASSWORD\"}}}" > /kaniko/.docker/config.json
  script:
    - echo "Building and pushing Docker image"
    - >-
      /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "$HARBOR_HOST/$HARBOR_PROJECT/$DEST_ENVIRONMENT/$CI_PROJECT_NAME:$CI_COMMIT_SHORT_SHA"
      --cache=true
      --cache-ttl=24h
      --skip-tls-verify
      --build-arg HARBOR_HOST=${HARBOR_HOST}
  needs: ['package']
  tags:
    - kubernetes
  # rules:
  #   - when: manual
  #     allow_failure: true

#################################################### update labels #########################################################
.update_template: # Define template for update tags
  image:
    name: $HARBOR_HOST/kubernetes/argocli-git:v2.7.14
  stage: update
  script:
    - echo "Updating Kustomize configs in Git with the latest commit SHA"
    - >-
      cd k8s/overlays/$DEST_ENVIRONMENT/ &&
      sed "s/\${CI_COMMIT_SHORT_SHA}/\"${CI_COMMIT_SHORT_SHA}\"/g" kustomization.yaml.template > kustomization.yaml &&
      git add kustomization.yaml &&
      git commit -m "build: update the image tag to ${CI_COMMIT_SHORT_SHA} [skip ci]" &&
      git push $REPO_URL HEAD:$CI_COMMIT_REF_NAME
  needs: ['build']
  tags:
    - kubernetes

######################################################### deploy app #########################################################
.sync_application_template: # Define app sync template
  image:
    name: $HARBOR_HOST/kubernetes/argocli-git:v2.7.14
  stage: deploy
  script:
    - echo "Logging in to ArgoCD"
    - >-
      export APP_NAME="${CI_PROJECT_NAME}-${HARBOR_PROJECT}-${DEST_ENVIRONMENT}" &&
      echo y | argocd login $ARGOCD_SERVER --password $ARGOCD_PASSWORD --username $ARGOCD_USERNAME
    - echo "Syncing application with ArgoCD"
    - echo "The branch being used is $CI_COMMIT_REF_NAME "
    - export ARGOCD_GIT_MODULES_ENABLED=false
    - >-
      if argocd app list | grep -q $APP_NAME; then
        echo "Application $APP_NAME exists. Syncing..."
        argocd app sync $APP_NAME --prune
      else
        echo "Application $APP_NAME does not exist. Creating..."
        argocd app create $APP_NAME \
        --repo $REPO_URL \
        --revision $CI_COMMIT_REF_NAME \
        --path k8s/overlays/$DEST_ENVIRONMENT \
        --dest-server https://kubernetes.default.svc \
        --sync-policy automated \
        --loglevel debug 
      fi
  needs: ['update-label']
  environment:
    name: $DEST_ENVIRONMENT
    url: https://$ARGOCD_SERVER/kubernetes/cluster/namespaces/$NAMESPACE_NAME
  tags:
    - kubernetes
  when: manual
  allow_failure: false
```

