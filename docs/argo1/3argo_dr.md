# 3 DevOps中的Argo CD — 灾难恢复

## 为什么要对 Argo CD 进行灾难恢复？


重要的是要了解 Argo CD 不直接与任何数据库交互，仅利用 Redis 作为缓存机制，使其看起来是无状态的。正如我们之前讨论的，主要通过增加每个部署的副本数量来实现高可用性安装是可能的。


但是，Argo CD 确实以应用程序定义（例如 Git 源和目标集群）、访问 Kubernetes 集群的信息以及连接到私有 Git 或 Helm 存储库的详细信息的形式维护状态。**Argo CD 的此状态信息存储在 Kubernetes 资源中。其中一些是本机功能，例如连接详细信息的秘密，而另一些是针对应用程序和应用程序约束的自定义功能**。

不可预见的情况，包括删除 Kubernetes 集群或 Argo CD 命名空间等人为行为，或者云提供商的复杂情况，都可能导致灾难。

还可能存在需要将 Argo CD 安装从一个集群转移到另一个集群的情况。所以Argo CD灾难恢复对于生产环境来说是必要的。

另外，请记住在 Argo CD 中：

* **并非所有内容都会保存到 Git 存储库中**。
* **从 Git 存储库重新创建所有内容可能需要花费大量时间，尤其是当您有大量应用程序时**

## 什么是 Argo CD 灾难恢复？

Argo CD 灾难恢复是在灾难性事件发生后恢复 Argo CD 部署的过程，例如人为错误（例如，删除 Argo CD 命名空间或 Kubernetes 集群）、软件错误或基础设施问题（例如，云提供商中断或数据中心）失败）。

灾难恢复的目的是使您的系统恢复到正常运行状态，同时最大限度地减少停机时间和数据丢失。这包括恢复应用程序部署状态、恢复 Argo CD 数据库以及恢复 Git 存储库和 Kubernetes 集群之间的同步。

**它通常涉及以下步骤：**

* **备份**：定期备份重要数据，例如 Argo CD 数据库和 Argo CD 用于连接 Git 存储库或 Kubernetes 集群的 Kubernetes Secrets。备份频率通常取决于您更改应用程序的频率。
* **恢复**：如果发生灾难，您可以从备份恢复Argo CD 安装。这通常涉及创建新的 Argo CD 实例并将备份数据加载到其中。 
* **验证**：恢复后，验证恢复是否成功非常重要。这可能涉及检查所有应用程序是否处于正确状态，并且 Argo CD 可以正确地将所需状态从 Git 同步到 Kubernetes 集群。


### “argocd admin”命令

“argocd admin”命令包含一组对 Argo CD 管理员有用的命令，并且需要直接 Kubernetes 访问。

语法如下：

```
$ argocd admin [flags] [commands]

Usage:
  argocd admin [flags]
  argocd admin [command]

Available Commands:
  app              Manage applications configuration
  cluster          Manage clusters configuration
  dashboard        Starts Argo CD Web UI locally
  export           Export all Argo CD data to stdout (default) or a file
  import           Import Argo CD data from stdin (specify `-') or a file
  initial-password Prints initial password to log in to Argo CD for the first time
  notifications    Set of CLI commands that helps manage notifications settings
  proj             Manage projects configuration
  repo             Manage repositories configuration
  settings         Provides set of commands for settings validation and troubleshooting
```

**创建备份**

现在，让我们连接到集群并创建备份。您应该连接到安装了 Argo CD 的集群。


运行以下命令，这将根据当前日期和时间创建一个具有自定义名称的文件：

```
$ argocd admin export -n argocd > argocd-backup-$(date +"%Y-%m-%d_%H:%M").yml

$ ls -l
-rw-rw-r-- 1 txu  txu  63010 Jun 14 20:18 argocd-backup-2023-06-14_20:18.yml
```

下一步，您应该获取此备份文件并将其保存在云存储系统（例如 AWS S3、Azure Blob 或 Google Cloud Storage）上，对其进行加密并制定访问策略。

### 从备份恢复

恢复备份需要在目标集群中初始安装 Argo CD。备份包括其配置以及所有相关的 ConfigMap 和 Secret，其中应包含初始设置期间所做的所有修改。

运行以下命令：

```
$ argocd admin import - < argocd-backup-2023-06-14_20:18.yml
...
/ConfigMap argocd-cm created
/ConfigMap argocd-rbac-cm created
/ConfigMap argocd-ssh-known-hosts-cm created
/ConfigMap argocd-tls-certs-cm created
/Secret argocd-secret created
/Secret autopilot-secret created
argoproj.io/AppProject default created
argoproj.io/AppProject testing created
argoproj.io/Application argo-cd created
argoproj.io/Application autopilot-bootstrap created
argoproj.io/Application cluster-resources-in-cluster created
argoproj.io/Application root created
argoproj.io/ApplicationSet cluster-resources created
argoproj.io/ApplicationSet testing created
```

恢复过程完成后，您的新安装应该复制备份时的状态，包括所有应用程序、集群和 Git 存储库。

**<mark>关键区别在于 Redis 缓存中没有数据。因此，Argo CD 将需要开始重新处理 Git 存储库中的所有清单，这最初可能会影响系统性能。然而，这种情况应该会在几分钟后稳定下来，使操作恢复到正常状态。</mark>**


### 自动化备份过程

一旦完全测试了备份和恢复命令，您就应该开始自动化备份过程。

自动化 Argo CD 备份过程可以带来显着的好处，尤其是在生产环境中：

* 一致性：自动化确保备份过程每次都一致执行，减少人为错误的可能性。它还保证备份所有必要的资源。

* 频率：可以安排自动备份定期运行，确保您始终拥有最新的备份。这对于频繁发生变化的活跃环境尤其重要。

* 效率：自动备份过程比手动备份过程高效得多，可以释放人力资源来执行更复杂的任务。

* 速度和恢复时间：当灾难发生时，拥有一个经过验证的自动化备份过程可以显着加快恢复时间，从而最大限度地减少停机时间。

