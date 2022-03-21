

# Gitlab-runner



GitLab Runner是一个开源项目，用于运行您的作业并将结果发送回GitLab。它与[GitLab CI](https://about.gitlab.com/product/continuous-integration/)结合使用，[GitLab CI](https://about.gitlab.com/product/continuous-integration/)是[GitLab](https://about.gitlab.com/product/continuous-integration/)随附的用于协调作业的开源持续集成服务。



### 要求

- GitLab Runner是用[Go](https://golang.org/)编写的，可以作为一个二进制文件运行，不需要特定于语言的要求。它旨在在GNU / Linux，macOS和Windows操作系统上运行。只要您可以在其他操作系统上编译Go二进制文件，其他操作系统就可能会运行。

- 如果要[使用Docker](https://docs.gitlab.com/12.8/runner/executors/docker.html)，请安装最新版本。GitLab Runner需要最少的Docker `v1.13.0`。

- GitLab Runner版本应与GitLab版本同步。尽管较旧的Runner仍可以使用较新的GitLab版本，反之亦然，但在某些情况下，如果版本存在差异，则功能可能不可用或无法正常工作。在次要版本更新之间可以保证向后兼容性，但是请注意，GitLab的次要版本更新会引入新功能，这些新功能将要求Runner在同一次要版本上使用。



### 特点

- 允许运行：
  - 同时执行多个作业。
  - 对多个服务器（甚至每个项目）使用多个令牌。
  - 限制每个令牌的并行作业数。
- 可以运行作业：
  - 在本地。
  - 使用Docker容器。
  - 使用Docker容器并通过SSH执行作业。
  - 使用Docker容器在不同的云和虚拟化管理程序上自动缩放。
  - 连接到远程SSH服务器。
- 用Go编写并以单个二进制文件的形式分发，而没有其他要求。
- 支持Bash，Windows Batch和Windows PowerShell。
- 在GNU / Linux，macOS和Windows（几乎可以在任何可以运行Docker的地方）上运行。
- 允许自定义作业运行环境。
- 自动重新加载配置，无需重启。
- 易于使用的设置，并支持Docker，Docker-SSH，Parallels或SSH运行环境。
- 启用Docker容器的缓存。
- 易于安装，可作为GNU / Linux，macOS和Windows的服务。
- 嵌入式Prometheus指标HTTP服务器。
- 裁判工作者监视Prometheus度量标准和其他特定于工作的数据并将其传递给GitLab。





