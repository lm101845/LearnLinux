##  配置使用分布式缓存





## minio

MinIO 是一个基于Apache License v2.0开源协议的对象存储服务。它兼容亚马逊S3云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几kb到最大5T不等。

MinIO是一个非常轻量的服务,可以很简单的和其他应用的结合，类似 NodeJS, Redis 或者 MySQL。

官方文档：https://docs.min.io/cn/deploy-minio-on-kubernetes.html 



### 基于Kubernetes部署

准备一个pv用于存储bucket数据。这里我使用的是本地的目录"/data/devops/minio-data"，根据大家不同的环境按需调整即可。

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ci-minio-pv
  namespace: devops
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/devops/minio-data"
---
```



```
kubectl create -f pv.yaml
kubectl get pv -n devops
```



#### Helm Install 

stable/minio	3.0.3  是目前最新的RELEASE版本

修改minio/values.yml中的service端口为80。(可选)

```
service:
  type: ClusterIP
  clusterIP: ~
  port: 80
  nodePort: 31311
  # externalIPs:
  #   - externalIp1
  annotations: {}
```

修改minio/values.yml中的Ingress配置。

```
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
    # kubernetes.io/ingress.allow-http: "false"
    # kubernetes.io/ingress.global-static-ip-name: ""
    # nginx.ingress.kubernetes.io/secure-backends: "true"
    # nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # nginx.ingress.kubernetes.io/whitelist-source-range: 0.0.0.0/0
  path: /
  hosts:
    - minio.devops.svc.cluster.local
  tls: []
```



```
helm install minio --namespace=devops --set persistence.size=50Gi,persistence.VolumeName=ci-minio-pv,persistence.storageClass=manual ./minio  
```

- persistence.size=50Gi  配置PV的大小为50Gi
- persistence.VolumeName=ci-minio-pv  指定已存在pv名称
- persistence.storageClass=manual 指定pv存储的类型为manual



----

部署成功会提示类似以下信息

```
NOTES:
Minio can be accessed via port 80 on the following DNS name from within your cluster:
minio.devops.svc.cluster.local

To access Minio from localhost, run the below commands:

  1. export POD_NAME=$(kubectl get pods --namespace devops -l "release=minio" -o jsonpath="{.items[0].metadata.name}")

  2. kubectl port-forward $POD_NAME 9000 --namespace devops

Read more about port forwarding here: http://kubernetes.io/docs/user-guide/kubectl/kubectl_port-forward/

You can now access Minio server on http://localhost:9000. Follow the below steps to connect to Minio server with mc client:

  1. Download the Minio mc client - https://docs.minio.io/docs/minio-client-quickstart-guide

  2. mc config host add minio-local http://localhost:9000 AKIAIOSFODNN7EXAMPLE wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY S3v4

  3. mc ls minio-local

Alternately, you can use your browser or the Minio SDK to access the server - https://docs.minio.io/categories/17
```





#### 访问验证

添加hosts解析测试

```
192.168.1.200 minio.devops.svc.cluster.local
```

![images](images/40.png)

---



#### 创建bucket

![iamges](images/41.png)



![iamges](images/42.png)



![iamges](images/43.png)





----



## 配置Runner使用S3存储



官方的runner 配置 examples是这样配置的，指定S3存储相关信息。

参考文档：https://docs.gitlab.com/runner/configuration/autoscale.html#distributed-runners-caching

```
[[runners]]
  limit = 10
  executor = "docker+machine"
  [runners.cache]
    Type = "s3"
    Path = "path/to/prefix"
    Shared = false
    [runners.cache.s3]
      ServerAddress = "s3.example.com"
      AccessKey = "access-key"
      SecretKey = "secret-key"
      BucketName = "runner"
      Insecure = false
```

如果您直接使用的vm或者docker安装的runner，可以直接更新runner的配置文件填写S3信息。如果您使用的是Kuberntes环境部署的runner则需要根据以下配置来修改Helm  chart。

 

步骤1：创建一个secret保存S3的认证信息，由于我的runner在gitlab-runner名称空间，所以这里指定的也是同样的。

```
kubectl create secret generic s3access --namespace=gitlab-runner --from-literal=accesskey="AKIAIOSFODNN7EXAMPLE" --from-literal=secretkey="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

步骤2： 编辑helm chart 中的values.yml文件，填写S3配置信息。

```
cache:
    ## General settings
    cacheType: s3
    cachePath: "gitlab-runner"
    cacheShared: true

    ## S3 settings
    s3ServerAddress: minio.devops.svc.cluster.local
    s3BucketName: gitlab-ci-runner-cache
    s3BucketLocation:
    s3CacheInsecure: false
    secretName: s3access
```

- cacheType：  缓存的类型，指定s3
- cachePath：缓存路径，值得是bucket中的目录。可以自定义。
- CacheShared：是否共享，如果存在多个runner则需要开启。
- s3ServerAddress： S3服务器地址，minio域名。
- s3BucketName： S3 bucket的名称，参考上面我们创建的名称。
- s3BucketLocation： Location 默认即可，可选。
- s3CacheInsecure：是否使用https。(这里官方chart有问题，配置的是不管是true还是false都是true，后面会修改)
- secretName：凭据名称， 我们在上面创建的s3凭据。

----

小问题： 发现s3CacheInsecure设置runner 配置 Insecure没有生效，解决方法。

方式一： 声明环境变量（不生效，放弃）

```
--cache-s3-insecure       Use insecure mode (without https) [$CACHE_S3_INSECURE]
```

```
envVars:
  - name: CACHE_S3_INSECURE
    value: false
```

方式二：修改文件templates/_cache.tpl

官方配置的是 这个值如果为true 则为true ，那如果我们要设置false呢？ 很明显没有配置了~~~。

```
 18 {{-       if .Values.runners.cache.s3CacheInsecure }}
 19 - name: CACHE_S3_INSECURE
 20   value: "true"
 21 {{-       end }}
```

改成下面的配置，即不加条件判断，根据s3CacheInsecure 实际的参数值。

```
- name: CACHE_S3_INSECURE
  value: {{ default "" .Values.runners.cache.s3CacheInsecure | quote }}
```



----

ok，上面我们遇到了一些小问题，但都通过自定义helmchart 解决了。 接下来更新我们的runner 配置。

```
helm upgrade gitlab-runner ./gitlab-runner --namespace gitlab-runner
```

更新完成，我们在pod中查看runner配置文件是否正常。大家可以参考以下图片为正常的。

![images](images/39.png)

---



## 运行流水线测试

这条流水线很简单，我们这样配置： 设计两个步骤，build，test。 build打包后会产生target目录。默认在不配置全局缓存的情况下test作业执行的时候是没有target目录的。

```
cache:
  paths:
    - target/
build:
  stage: build
  script:
    - mvn clean package
    - ls 
    
test:
  stage: test
  script:
    - ls 
    - ls target/
```



build作业开始运行，获取缓存。发现缓存不存在，运行任务。

![images](images/44.png)

build作业执行完成，收集缓存到S3。

![images](images/45.png)

test作业开始运行，发现build作业产生的缓存。

![images](images/46.png)

test作业运行完成，上传缓存到S3。

![images](images/47.png)



---

OK，到这里S3缓存已经配置完成了。 我们可以在minio服务中查看生成的数据。

![iamges](images/48.png)



![images](images/49.png)

----



## FAQ 

连接超时问题，这个问题与runners 3CacheInsecure配置参数有关。可以参考上面步骤解决。使用http模式即可。

```
Restoring cache
 Checking cache for default-1...
 WARNING: Retrying...                                error=Get https://minio.devops.svc.cluster.local/gitlab-ci-runner-cache/gitlab-runner/project/20/default-1?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20200516%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200516T061458Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=b604bdfdda3b26286e714ed3a4073c103f6216f13b4efb12a0c76734160b1fde: dial tcp 10.1.117.2:443: i/o timeout
 WARNING: Retrying...                                error=Get https://minio.devops.svc.cluster.local/gitlab-ci-runner-cache/gitlab-runner/project/20/default-1?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIOSFODNN7EXAMPLE%2F20200516%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20200516T061458Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=b604bdfdda3b26286e714ed3a4073c103f6216f13b4efb12a0c76734160b1fde: dial tcp 10.1.117.2:443: i/o timeout
 FATAL: Get https://minio.devops.svc.cluster.local/gitlab-ci-runner-cache/gitlab-runner/project/20/default-1?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=[FILTERED]&X-Amz-Date=20200516T061458Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=[FILTERED] dial tcp 10.1.117.2:443: i/o timeout 
```





