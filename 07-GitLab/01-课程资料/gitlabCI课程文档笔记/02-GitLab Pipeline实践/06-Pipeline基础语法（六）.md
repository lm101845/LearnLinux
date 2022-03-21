# pipeline 基础语法



这次我们在学习语法时候需要准备一个注册docker执行器类型的runner。可以参考以下命令指定：

```
gitlab-runner register \
  --non-interactive \
  --executor "docker" \
  --docker-image alpine:latest \
  --url "http://192.168.1.200:30088/" \
  --registration-token "JRzzw2j1Ji6aBjwvkxAv" \
  --description "docker-runner" \
  --tag-list "newdocker" \
  --run-untagged="true" \
  --locked="false" \
  --access-level="not_protected"
```



```
[[runners]]
  name = "docker-runner"
  url = "http://192.168.1.200:30088/"
  token = "xuaLZD7xUVviTsyeJAWh"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
  [runners.docker]
    pull_policy = "if-not-present"
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
```



### image

默认在注册runner的时候需要填写一个基础的镜像，请记住一点只要使用执行器为docker类型的runner所有的操作运行都会在容器中运行。 如果全局指定了images则所有作业使用此image创建容器并在其中运行。 全局未指定image，再次查看job中是否有指定，如果有此job按照指定镜像创建容器并运行，没有则使用注册runner时指定的默认镜像。

```
#image: maven:3.6.3-jdk-8

before_script:
  - ls
  
  
build:
  image: maven:3.6.3-jdk-8
  stage: build
  tags:
    - newdocker
  script:
    - ls
    - sleep 2
    - echo "mvn clean "
    - sleep 10

deploy:
  stage: deploy
  tags:
    - newdocker
  script:
    - echo "deploy"

```



---



### services

工作期间运行的另一个Docker映像，并link到`image`关键字定义的Docker映像。这样，您就可以在构建期间访问服务映像.

服务映像可以运行任何应用程序，但是最常见的用例是运行数据库容器，例如`mysql` 。与每次安装项目时都安装`mysql`相比，使用现有映像并将其作为附加容器运行更容易，更快捷。

```

services:
  - name: mysql:latest
    alias: mysql-1

```



---



### environment

声明所部署的环境名称和访问地址，后续可以直接在gitlab 环境变量中查看。非常方便。

```
deploy to production:
  stage: deploy
  script: git push production HEAD:master
  environment:
    name: production
    url: https://prod.example.com
```



---



### inherit

使用或禁用全局定义的环境变量（variables）或默认值(default)。



*使用true、false决定是否使用，默认为true*

```
inherit:
  default: false
  variables: false
```



*继承其中的一部分变量或默认值使用list*

```
inherit:
  default:
    - parameter1
    - parameter2
  variables:
    - VARIABLE1
    - VARIABLE2
```

---

