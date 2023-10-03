# **1 使用 Argo CD 进行 GitOps 流水线改造**

Argo CD 是一个为 Kubernetes 而生的，遵循声明式 GitOps 理念的持续部署工具。

Argo CD 可在 Git 存储库更改时自动同步和部署应用程序。

Argo CD 遵循 GitOps 模式，使用 Git 仓库作为定义所需应用程序状态的真实来源，Argo CD 支持多种 Kubernetes 清单

* kustomize
* helm charts
* ksonnet applications
* jsonnet files
* Plain directory of YAML/json manifests
* Any custom config management tool configured as a config management plugin

Argo CD 可在指定的目标环境中自动部署所需的应用程序状态，应用程序部署可以在 Git 提交时跟踪对分支、标签的更新，或固定到清单的指定版本。

## 架构

![Alt Image Text](../images/ag1_1_1.png "Body image")

Argo CD 是通过 Kubernetes 控制器来实现的，它持续 watch 正在运行的应用程序并将当前的实时状态与所需的目标状态（ Git 存储库中指定的）进行比较。已经部署的应用程序的实际状态与目标状态有差异，则被认为是 OutOfSync 状态，Argo CD 会报告显示这些差异，同时提供工具来自动或手动将状态同步到期望的目标状态。在 Git 仓库中对期望目标状态所做的任何修改都可以自动应用反馈到指定的目标环境中去。

下面简单介绍下 Argo CD 中的几个主要组件：

**API 服务**：API 服务是一个 gRPC/REST 服务，它暴露了 Web UI、CLI 和 CI/CD 系统使用的接口，主要有以下几个功能：

* 应用程序管理和状态报告
* 执行应用程序操作（例如同步、回滚、用户定义的操作）
* 存储仓库和集群凭据管理（存储为 K8s Secrets 对象）
* 认证和授权给外部身份提供者
* RBAC
* Git webhook 事件的侦听器/转发器

**仓库服务**：存储仓库服务是一个内部服务，负责维护保存应用程序清单 Git 仓库的本地缓存。当提供以下输入时，它负责生成并返回 Kubernetes 清单：

* 存储 URL
* revision 版本（commit、tag、branch）
* 应用路径
* 模板配置：参数、ksonnet 环境、helm values.yaml 等


**应用控制器**：应用控制器是一个 Kubernetes 控制器，它持续 watch 正在运行的应用程序并将当前的实时状态与所期望的目标状态（repo 中指定的）进行比较。

**它检测应用程序的 OutOfSync 状态，并采取一些措施来同步状态，它负责调用任何用户定义的生命周期事件的钩子（PreSync、Sync、PostSync）**。

## 安装

当然前提是需要有一个 kubectl 可访问的 Kubernetes 的集群，直接使用下面的命令即可，这里我们安装最新的 v2.8.4 版本：

```
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.4/manifests/install.yaml
```

如果你要用在生产环境，则可以使用下面的命令部署一个 HA 高可用的版本：

```
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.4/manifests/ha/install.yaml
```

这将创建一个新的命名空间 argocd，Argo CD 的服务和应用资源都将部署到该命名空间。

```

$ kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          103s
argocd-applicationset-controller-68b9bdbd8b-jzcpf   1/1     Running   0          103s
argocd-dex-server-6b7745757-6mxwk                   1/1     Running   0          103s
argocd-notifications-controller-5b56f6f7bb-jqpng    1/1     Running   0          103s
argocd-redis-f4cdbff57-dr8jc                        1/1     Running   0          103s
argocd-repo-server-c4f79b4d6-7nh6n                  1/1     Running   0          103s
argocd-server-895675597-fr42g                       1/1     Running   0          103s
```

> 如果你对 UI、SSO、多集群管理这些特性不感兴趣，只想把应用变更同步到集群中，那么可以直接安装核心组件即可：kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.4/manifests/core-install.yaml。

然后我们可以在本地（选择对应的版本）安装 CLI 工具方便操作 Argo CD：

```
$ curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.8.4/argocd-linux-amd64
```

为 argocd CLI 赋予可执行权限：

```
$ chmod +x /usr/local/bin/argocd
```

现在我们就可以使用 **argocd** 命令了。

如果你是 Mac，则可以直接使用 `brew install argocd` 进行安装。

**Argo CD 会运行一个 gRPC 服务（由 CLI 使用）和 HTTP/HTTPS 服务（由 UI 使用）**，这两种协议都由 argocd-server 服务在以下端口进行暴露：

* 443 - gRPC/HTTPS
* 80 - HTTP（重定向到 HTTPS）

我们可以通过配置 Ingress 的方式来对外暴露服务，其他 Ingress 控制器的配置可以参考官方文档 [https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/) 进行配置。

Argo CD 在同一端口 (443) 上提供多个协议 (gRPC/HTTPS)，所以当我们为 argocd 服务定义单个 nginx ingress 对象和规则的时候有点麻烦，因为 **`nginx.ingress.kubernetes.io/backend-protocol`** 这个 annotation 只能接受一个后端协议（例如 HTTP、HTTPS、GRPC、GRPCS）。


为了使用单个 ingress 规则和主机名来暴露 Argo CD APIServer，**必须使用 `nginx.ingress.kubernetes.io/ssl-passthrough` 这个 annotation 来传递 TLS 连接并校验 `Argo CD APIServer` 上的 TLS**。


```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
```

上述规则在 Argo CD APIServer 上校验 TLS，该服务器检测到正在使用的协议，并做出适当的响应。请注意，`nginx.ingress.kubernetes.io/ssl-passthrough` 注解要求将 `--enable-ssl-passthrough` 标志添加到 nginx-`ingress-controller` 的命令行参数中。

由于 `ingress-nginx` 的每个 Ingress 对象仅支持一个协议，因此另一种方法是定义两个 Ingress 对象。

**一个用于 `HTTP/HTTPS`，另一个用于 `gRPC`。**

如下所示为 HTTP/HTTPS 的 Ingress 对象：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-http-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: http
      host: argocd.k8s.local
  tls:
    - hosts:
        - argocd.k8s.local
      secretName: argocd-secret # do not change, this is provided by Argo CD
```

**gRPC 协议对应的 Ingress 对象如下所示**：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-grpc-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
      host: grpc.argocd.k8s.local
  tls:
    - hosts:
        - grpc.argocd.k8s.local
      secretName: argocd-secret # do not change, this is provided by Argo CD
```


**然后我们需要在禁用 TLS 的情况下运行 APIServer**。


**编辑 `argocd-server` 这个 Deployment 以将 `--insecure` 标志添加到 `argocd-server` 命令，或者简单地在 `argocd-cmd-params-cm` ConfigMap 中设置 `server.insecure: "true"` 即可。**

创建完成后，我们就可以通过 `argocd.k8s.local` 来访问 Argo CD 服务了，不过需要注意我们这里配置的证书是自签名的，所以在第一次访问的时候会提示不安全，强制跳转即可。


默认情况下 `admin` 帐号的初始密码是自动生成的，会以明文的形式存储在 Argo CD 安装的命名空间中名为 `argocd-initial-admin-secret` 的 `Secret` 对象下的 `password` 字段下，我们可以用下面的命令来获取：

```
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

使用用户名 admin 和上面输出的密码即可登录 Dashboard。

![Alt Image Text](../images/ag1_1_2.png "Body image")

同样我们也可以通过 ArgoCD CLI 命令行工具进行登录：

```
$ argocd login grpc.argocd.k8s.local
WARNING: server certificate had error: tls: failed to verify certificate: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
Username: admin
Password:
'admin:login' logged in successfully
Context 'grpc.argocd.k8s.local' updated
```

需要注意的是这里登录的地址为 gRPC 暴露的服务地址。

CLI 登录成功后，可以使用如下所示命令更改密码：

```
$ argocd account update-password
*** Enter current password:
*** Enter new password:
*** Confirm new password:
Password updated
Context 'argocd.k8s.local' updated
$ argocd version
argocd: v2.8.4+c279299
  BuildDate: 2023-09-13T19:43:37Z
  GitCommit: c27929928104dc37b937764baf65f38b78930e59
  GitTreeState: clean
  GoVersion: go1.20.7
  Compiler: gc
  Platform: darwin/arm64
argocd-server: v2.8.4+c279299
  BuildDate: 2023-09-13T19:12:09Z
  GitCommit: c27929928104dc37b937764baf65f38b78930e59
  GitTreeState: clean
  GoVersion: go1.20.6
  Compiler: gc
  Platform: linux/amd64
  Kustomize Version: v5.1.0 2023-06-19T16:58:18Z
  Helm Version: v3.12.1+gf32a527
  Kubectl Version: v0.24.2
  Jsonnet Version: v0.20.0
```

### 配置集群

由于 Argo CD 支持部署应用到多集群，所以如果你要将应用部署到外部集群的时候，需要先将外部集群的认证信息注册到 Argo CD 中，如果是在内部部署（运行 Argo CD 的同一个集群，默认不需要配置），直接使用 `https://kubernetes.default.svc` 作为应用的 `K8S APIServer` 地址即可。

首先列出当前 kubeconfig 中的所有集群上下文：

```
$ kubectl config get-contexts -o name
kubernetes-admin@kubernetes
testcluster
```

**从列表中选择一个上下文名称并将其提供给 `argocd cluster add CONTEXTNAME`**，比如对于 orbstack 上下文，运行：

```
$ argocd cluster add testcluster
```

## 创建应用

Git 仓库 [https://github.com/argoproj/argocd-example-apps.git](https://github.com/argoproj/argocd-example-apps.git) 是一个包含留言簿应用程序的示例库，我们可以用该应用来演示 Argo CD 的工作原理

### 通过 CLI 创建应用

我们可以通过 argocd app create xxx 命令来创建一个应用：

```
$ argocd app create --help
Create an application

Usage:
  argocd app create APPNAME [flags]

Examples:

        # Create a directory app
        argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-namespace default --dest-server https://kubernetes.default.svc --directory-recurse

        # Create a Jsonnet app
        argocd app create jsonnet-guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path jsonnet-guestbook --dest-namespace default --dest-server https://kubernetes.default.svc --jsonnet-ext-str replicas=2

        # Create a Helm app
        argocd app create helm-guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path helm-guestbook --dest-namespace default --dest-server https://kubernetes.default.svc --helm-set replicaCount=2

        # Create a Helm app from a Helm repo
        argocd app create nginx-ingress --repo https://charts.helm.sh/stable --helm-chart nginx-ingress --revision 1.24.3 --dest-namespace default --dest-server https://kubernetes.default.svc

        # Create a Kustomize app
        argocd app create kustomize-guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path kustomize-guestbook --dest-namespace default --dest-server https://kubernetes.default.svc --kustomize-image gcr.io/heptio-images/ks-guestbook-demo:0.1

        # Create a app using a custom tool:
        argocd app create kasane --repo https://github.com/argoproj/argocd-example-apps.git --path plugins/kasane --dest-namespace default --dest-server https://kubernetes.default.svc --config-management-plugin kasane


Flags:
......
```

直接执行如下所示命令即可：

```
$ argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
application 'guestbook' created
```

### 通过 UI 创建应用

除了可以通过 CLI 工具来创建应用，我们也可以通过 UI 界面来创建，定位到 argocd.k8s.local 页面，登录后，点击 +New App 新建应用按钮，如下图：

![Alt Image Text](../images/ag1_1_3.png "Body image")

将应用命名为 guestbook，使用 default project，并将同步策略设置为 Manual：

![Alt Image Text](../images/ag1_1_4.png "Body image")

然后在下面配置 Repository URL 为 [https://github.com/argoproj/argocd-example-apps.git](https://github.com/argoproj/argocd-example-apps.git)，由于某些原因我们这里使用的是 Gitee 仓库地址 [https://gitee.com/cnych/argocd-example-apps](https://gitee.com/cnych/argocd-example-apps)，将 Revision 设置为 HEAD，并将路径设置为 guestbook。

**然后下面的 Destination 部分，将 cluster 设置为 inCluster 和 namespace 为 default：**

![Alt Image Text](../images/ag1_1_5.png "Body image")

填写完以上信息后，点击页面上方的 Create 安装，即可创建 guestbook 应用，创建完成后可以看到当前应用的处于 OutOfSync 状态：

![Alt Image Text](../images/ag1_1_6.png "Body image")

Argo CD 默认情况下每 3 分钟会检测 Git 仓库一次，用于判断应用实际状态是否和 Git 中声明的期望状态一致，如果不一致，状态就转换为 OutOfSync。默认情况下并不会触发更新，除非通过 syncPolicy 配置了自动同步。 


### 通过 CRD 创建

除了可以通过 CLI 和 Dashboard 可以创建 Application 之外，其实也可以直接通过声明一个 Application 的资源对象来创建一个应用，如下所示：

```

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
spec:
  destination:
    namespace: default
    server: "https://kubernetes.default.svc"
  source:
    path: guestbook
    repoURL: "https://github.com/cnych/argocd-example-apps"
    targetRevision: HEAD
  project: default
  syncPolicy:
    automated: null
```

## 部署应用

由于上面我们在创建应用的时候使用的同步策略为 Manual，所以应用创建完成后没有自动部署，需要我们手动去部署应用。同样可以通过 CLI 和 UI 界面两种同步方式。


### 使用 CLI 同步

应用创建完成后，我们可以通过如下所示命令查看其状态：

```
$ argocd app get argocd/guestbook
Name:               argocd/guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://grpc.argocd.k8s.local/applications/guestbook
Repo:               https://gitee.com/cnych/argocd-example-apps
Target:             HEAD
Path:               guestbook
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        OutOfSync from HEAD (f3736e6)
Health Status:      Missing

GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH   HOOK  MESSAGE
       Service     default    guestbook-ui  OutOfSync  Missing
apps   Deployment  default    guestbook-ui  OutOfSync  Missing
```

应用程序状态为初始 OutOfSync 状态，因为应用程序尚未部署，并且尚未创建任何 Kubernetes 资源。要同步（部署）应用程序，可以执行如下所示命令：

```
$ argocd app sync argocd/guestbook
```

此命令从 Git 仓库中检索资源清单并执行 kubectl apply 部署应用，执行上面命令后 guestbook 应用便会运行在集群中了，现在我们就可以查看其资源组件、日志、事件和评估其健康状态了。

### 通过 UI 同步

直接添加 UI 界面上应用的 Sync 按钮即可开始同步：

![Alt Image Text](../images/ag1_1_7.png "Body image")

同步完成后可以看到我们的资源状态，甚至还可以直接查看应用的日志信息：

![Alt Image Text](../images/ag1_1_8.png "Body image")

也可以通过 kubectl 查看到我们部署的资源：

```
$ kubectl get pods
NAME                                 READY   STATUS      RESTARTS       AGE
guestbook-ui-6c96fb4bdc-bdwh9        1/1     Running     0              3m3s

➜  ~ kubectl get svc
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
guestbook-ui         ClusterIP      10.100.170.117   <none>         80/TCP                       3m16s
kubernetes           ClusterIP      10.96.0.1        <none>         443/TCP 
```

和我们从 Git 仓库中同步 guestbook 目录下面的资源状态也是同步的，证明同步成功了。

## 流水线改造

前面我们通过 Jenkins Pipeline 已经成功的将应用部署到了集群中了，但是我们使用的是传统的主动 push 方式，接下来我们需要将这个流程改造成为一个 GitOps 的流水线，这样我们就可以通过 Git 来管理应用的部署了。

> 使用到的代码仓库位于 https://github.com/cnych/drone-k8s-demo，然后迁移到内部的 gitlab 环境上实验。

前面 Jenkins Pipeline 中我们在发布应用的时候是通过 helm 方式来部署的，现在我们只需要将流水线的 CD 部分进行改造，比如将镜像构建后推送到镜像仓库，然后去修改 git 仓库中的 values 文件，Argo CD 来同步部署应用即可。

首先我们将应用的部署资源清单单独放一个 config 的仓库下面 [http://gitlab.k8s.local/cnych/k8s-devops-demo-config.git](http://gitlab.k8s.local/cnych/k8s-devops-demo-config.git)，将前面应用的 helm 目录上传到该仓库中。这样方便和 Argo CD 进行对接，整个项目下面只有用于应用部署的 Helm Chart 模板。

![Alt Image Text](../images/ag1_1_9.png "Body image")

如果有多个团队，每个团队都要维护大量的应用，就需要用到 Argo CD 的另一个概念：项目（Project）。Argo CD 中的项目（Project）可以用来对 Application 进行分组，不同的团队使用不同的项目，这样就实现了多租户环境。项目还支持更细粒度的访问权限控制：

* 限制部署内容（受信任的 Git 仓库）；
* 限制目标部署环境（目标集群和 namespace）；
* 限制部署的资源类型（例如 RBAC、CRD、DaemonSets、NetworkPolicy 等）；
* 定义项目角色，为 Application 提供 RBAC（例如 OIDC group 或者 JWT 令牌绑定）。

比如我们这里创建一个名为 demo 的项目，将该应用创建到该项目下，只需创建一个如下所示的 AppProject 对象即可：

```
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  # 项目名
  name: demo
  namespace: argocd
spec:
  # 目标
  destinations:
    # 此项目的服务允许部署的 namespace，这里为全部
    - namespace: "*"
      # 此项目允许部署的集群，这里为默认集群，即为Argo CD部署的当前集群
      server: https://kubernetes.default.svc
  # 允许的数据源
  sourceRepos:
    - http://gitlab.k8s.local/cnych/k8s-devops-demo-config.git
```

该对象中有几个核心的属性：

* sourceRepos：**项目中的应用程序可以从中获取清单的仓库引用**
* destinations：**项目中的应用可以部署到的集群和命名空间**
* roles：项目内资源访问定义的角色

直接创建该对象即可：

```
$ kubectl get appproject -n argocd
NAME      AGE
default   47h
demo      6s
```

然后前往 Argo CD 的 Settings 页面点击 + CONNECT REPO 添加仓库：

![Alt Image Text](../images/ag1_1_10.jpeg "Body image")


**需要注意的是这里的密码需要使用 AccessToken，** 我们可以前往 GitLab 的页面 `http://gitlab.k8s.local/-/profile/personal_access_tokens` 创建。

![Alt Image Text](../images/ag1_1_11.png "Body image")

更多配置信息可以前往文档 https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/ 查看，项目创建完成后，在该项目下创建一个 Application，代表环境中部署的应用程序实例。

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devops-demo
  namespace: argocd
spec:
  destination:
    namespace: default
    server: "https://kubernetes.default.svc"
  project: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  source:
    path: helm # 从 Helm 存储库创建应用程序时，chart 必须指定 path
    repoURL: "http://gitlab.k8s.local/cnych/k8s-devops-demo-config.git"
    targetRevision: HEAD
    helm:
      parameters:
        - name: replicaCount
          value: "2"
      valueFiles:
        - my-values.yaml
```

这里我们定义了一个名为 devops-demo 的应用，应用源来自于 helm 路径，使用的是 `my-values.yaml` 文件，此外还可以通过 source.helm.parameters 来配置参数。

 同步策略可以选择使用自动的方式，该策略下面还有两个属性可以配置：

![Alt Image Text](../images/ag1_1_12.png "Body image")

正常创建后这个应用会出现 Degraded 的错误，这是因为我们 Values 中的镜像默认为 latest，而我们没有将镜像推送到镜像仓库，所以会出现错误。

![Alt Image Text](../images/ag1_1_13.png "Body image")

接下来我们去修改 Jenkins Pipeline 的流水线，将 CD 部分进行修改。

```
podTemplate(cloud: "Kubernetes", nodeSelector: "kubernetes.io/hostname=node2", containers: [
  containerTemplate(name: 'golang', image: 'golang:1.18.3-alpine3.16', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker:latest', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'cnych/kubectl', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'helm', image: 'cnych/helm', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'yq', image: 'cnych/yq-jq:git', command: 'cat', ttyEnabled: true)
], serviceAccount: 'jenkins', envVars: [
  envVar(key: 'DOCKER_HOST', value: 'tcp://docker-dind:2375')  // 环境变量
]) {
  node(POD_LABEL) {
    def myRepo = checkout scm
    def gitConfigRepo = "gitlab.k8s.local/cnych/k8s-devops-demo-config.git"
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH

    // 获取 git commit id 作为镜像标签
    def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    // 仓库地址
    def registryUrl = "harbor.k8s.local"
    def imageEndpoint = "course/devops-demo"
    // 镜像
    def image = "${registryUrl}/${imageEndpoint}:${imageTag}"

    stage('单元测试') {
      echo "测试阶段"
    }
    stage('代码编译打包') {
        try {
            container('golang') {
            echo "2.代码编译打包阶段"
            sh """
                export GOPROXY=https://goproxy.io
                GOOS=linux GOARCH=amd64 go build -v -o demo-app
                """
            }
        } catch (exc) {
            println "构建失败 - ${currentBuild.fullDisplayName}"
            throw(exc)
        }
    }
    stage('构建 Docker 镜像') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: 'docker-auth',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASSWORD']]) {
            container('docker') {
                echo "3. 构建 Docker 镜像阶段"
                sh """
                docker login ${registryUrl} -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}
                docker build -t ${image} .
                docker push ${image}
                """
            }
        }
    }
    stage('修改 Config Repo') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: 'gitlab-auth',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_PASSWORD']]) {
                container('yq') {
                    echo "3. 修改 Config Repo 仓库 Values"
                    // Bed6gAYq
                    sh """
                    git clone http://${GIT_USER}:${GIT_PASSWORD}@${gitConfigRepo}
                    cd k8s-devops-demo-config
                    yq write -i -y helm/my-values.yaml image.tag "${imageTag}"

                    git add helm/my-values.yaml

                    git config --global user.name "cnych"
                    git config --global user.email "cnych@youdianzhishi.com"

                    git commit -m "update image tag to ${imageTag}"

                    git push http://${GIT_USER}:${GIT_PASSWORD}@${gitConfigRepo}
                    """
                }
        }

    }
    stage('运行 Kubectl') {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
            container('kubectl') {
                sh "mkdir -p ~/.kube && cp ${KUBECONFIG} ~/.kube/config"
                echo "5.查看应用"
                sh "kubectl get all -n kube-ops -l app=devops-demo"
            }
        }
    }

  }
}
```

上面的流水线中我们在应用构建成镜像后，直接去修改了 Config Repo 仓库中的 values 文件，然后提交到仓库中，这样 Argo CD 就会自动同步部署应用了。

**由于 Argo CD 默认并不是实时去监测 Config Repo 的变化的，如果要更快的检测到变化我们可以使用 Git Webhook 的方式**。

默认情况下 Argo CD 每三分钟轮询一次 Git 存储库，以检测清单的更改。为了消除轮询延迟，可以将 API 服务器配置为接收 Webhook 事件。Argo CD 支持来自 GitHub、GitLab、Bitbucket、Bitbucket Server 和 Gogs 的 Git webhook 通知。

**同样方式我们可以在 `k8s-devops-demo-config` 仓库下面创建一个 `Webhook`，Git 提供程序中配置的有效负载 URL 应使用 Argo CD 实例的 `/api/webhook` 端点（例如 https://argocd.example.com/api/webhook）**。


![Alt Image Text](../images/ag1_1_14.png "Body image")

```
$ kubectl edit secret argocd-secret -n argocd
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
data:
...

stringData:
  # github webhook secret
  webhook.github.secret: shhhh! it's a GitHub secret

  # gitlab webhook secret
  webhook.gitlab.secret: shhhh! it's a GitLab secret

  # bitbucket webhook secret
  webhook.bitbucket.uuid: your-bitbucket-uuid

  # bitbucket server webhook secret
  webhook.bitbucketserver.secret: shhhh! it's a Bitbucket server secret

  # gogs server webhook secret
  webhook.gogs.secret: shhhh! it's a gogs server secret
```

可以直接使用 stringData 来配置 secret，这样就不用去手动编码了。

![Alt Image Text](../images/ag1_1_15.png "Body image")

因为 GitOps 的核心是 Git，所以我们一定要将部署到集群中的资源清单文件全都托管到 Git 仓库中，这样才能实现 GitOps 的自动同步部署。上面我们是在 CI 流水线中去修改 Git 仓库中的资源清单文件，其实我们也可以通过其他方式去修改，比如 Argo CD 也提供了一个新的工具 Argo CD Image Updater。

## Argo CD Image Updater

Argo CD Image Updater 是一种自动更新由 Argo CD 管理的 Kubernetes 工作负载的容器镜像的工具。 

该工具可以检查与 Kubernetes 工作负载一起部署的容器镜像的新版本，并使用 Argo CD 自动将其更新到允许的最新版本。

它通过为 Argo CD 应用程序设置适当的应用程序参数来工作，类似于 `argocd app set --helm-set image.tag=v1.0.1`，但以完全自动化的方式。

Argo CD Image Updater 会定期轮询 Argo CD 中配置的应用程序，并查询相应的镜像仓库以获取可能的新版本。如果在仓库中找到新版本的镜像，并且满足版本约束，Argo CD 镜像更新程序将指示 Argo CD 使用新版本的镜像更新应用程序。

### 特征

* 更新由 Argo CD 管理且由 Helm 或 Kustomize 工具生成的应用程序镜像
* 根据不同的更新策略更新应用镜像
	* semver：根据给定的镜像约束更新到允许的最高版本
	* latest：更新到最近创建的镜像标签
	* name：更新到按字母顺序排序的列表中的最后一个标签
	* digest：更新到可变标签的最新推送版本

* 支持广泛使用的容器镜像仓库
* 通过配置支持私有容器镜像仓库
* 可以将更改写回 Git
* 能够使用匹配器函数过滤镜像仓库返回的标签列表
* 在 Kubernetes 集群中运行，或者可以从命令行独立使用
* 能够执行应用程序的并行更新

另外需要注意的是使用该工具目前有几个限制：

* 想要更新容器镜像的应用程序必须使用 Argo CD 进行管理。不支持未使用 Argo CD 管理的工作负载。
* Argo CD 镜像更新程序只能更新其清单使用 Kustomize 或 Helm 呈现的应用程序的容器镜像，特别是在 Helm 的情况下，模板需要支持使用参数（即`image.tag`）。
* 镜像拉取密钥必须存在于 Argo CD Image Updater 运行（或有权访问）的同一 Kubernetes 集群中。目前无法从其他集群获取这些机密信息。

### 安装

建议在运行 Argo CD 的同一个 Kubernetes 命名空间集群中运行 Argo CD Image Updater，但这不是必需的。事实上，甚至不需要在 Kubernetes 集群中运行 Argo CD Image Updater 或根本不需要访问任何 Kubernetes 集群。但如果不访问 Kubernetes，某些功能可能无法使用，所以强烈建议使用第一种安装方法。

运行镜像更新程序的最直接方法是将其作为 Kubernetes 工作负载安装到运行 Argo CD 的命名空间中。这样就不需要任何配置，也不会对你的工作负载产生任何影响。

```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

**Argo CD Image Updater** 安装完成后我们就可以直接去监听镜像是否发生了变化，而不需要在 CI 流水线中去手动提交修改资源清单到代码仓库了。

现在我们可以先去删除前面的 app：

```
$ argocd app delete devops-demo --cascade
Are you sure you want to delete 'devops-demo' and all its resources? [y/n] y
application 'devops-demo' deleted
```

然后接下来创建一个新的 Application 对象，对应的资源清单如下所示：

```
# demo-app2.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: devops-demo2
  annotations:
    argocd-image-updater.argoproj.io/image-list: myalias=cnych/devops-demo # Write repository name
    argocd-image-updater.argoproj.io/myalias.allow-tags: regexp:^.*$
    argocd-image-updater.argoproj.io/myalias.pull-secret: pullsecret:argocd/dockerhub-secret
    argocd-image-updater.argoproj.io/myalias.update-strategy: latest # There are several ways to update the image, but I'm using digest.
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/myalias.force-update: "true"
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  project: demo
  source:
    path: helm # 从 Helm 存储库创建应用程序时，chart 必须指定 path
    repoURL: http://gitlab.k8s.local/cnych/k8s-devops-demo-config.git
    targetRevision: main
    helm:
      parameters:
        - name: replicaCount
          value: "2"
      valueFiles:
        - my-values.yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**这个新的资源对象中，我们添加了一些注释，这些注释用于配置 Argo CD Image Updater。**

这些配置用于指定自动更新容器镜像的策略、参数和相关信息。以下是对这些注释的详细解释：


* `argocd-image-updater.argoproj.io/image-list`: 这个注解定义了应用中使用的镜像列表。
* `argocd-image-updater.argoproj.io/allow-tags`: 这个注解指定了允许更新的镜像标签，可以使用正则表达式的方式。-
* `argocd-image-updater.argoproj.io/<alias>.pull-secret`: 这个注解指定了用于拉取镜像的 Secret。
* `argocd-image-updater.argoproj.io/update-strategy`: 这个注解定义了镜像更新策略。这里的值是 latest，表示使用最新的镜像标签进行更新，还可以指定的值包括：digest、name、semver。
* `argocd-image-updater.argoproj.io/write-back-method`: 这个注解定义了更新后的配置写回方法。git 表示将更新后的配置写回到 Git 仓库。
* `argocd-image-updater.argoproj.io/git-branch`：这个注解定义了更新后的配置写回到 Git 仓库的分支。

注意上面我们配置了一个 pull-secret 的注解，如果使用的是 docker hub，需要在个人中心去创建一个 access token：

![Alt Image Text](../images/ag1_1_16.png "Body image")

然后使用如下命令创建一个 secret：

```
kubectl create -n argocd secret docker-registry dockerhub-secret \
  --docker-username xxxx \
  --docker-password xxxx \
  --docker-server "https://registry-1.docker.io"
```

然后我们就可以创建这个应用了：

```
$ kubectl apply -f demo-app2.yaml
```

创建后正常第一次会去同步部署应用。然后接下来我们可以去修改 Jenkins Pipeline 的流水线，只需要保留到镜像构建的部分即可，其他的部分都可以去掉了。

```
podTemplate(cloud: "Kubernetes", containers: [
  containerTemplate(name: 'golang', image: 'golang:1.18.3-alpine3.16', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker:latest', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'cnych/kubectl', command: 'cat', ttyEnabled: true)
], serviceAccount: 'jenkins', envVars: [
  envVar(key: 'DOCKER_HOST', value: 'tcp://docker-dind:2375')  // 环境变量
]) {
  node(POD_LABEL) {
    def myRepo = checkout scm
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH

    // 获取 git commit id 作为镜像标签
    def imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    // 仓库地址
    def registryUrl = "docker.io"
    def imageEndpoint = "cnych/devops-demo"
    // 镜像
    def image = "${registryUrl}/${imageEndpoint}:${imageTag}"

    stage('单元测试') {
      echo "测试阶段"
    }
    stage('代码编译打包') {
        try {
            container('golang') {
            echo "2.代码编译打包阶段"
            sh """
                export GOPROXY=https://goproxy.io
                GOOS=linux GOARCH=amd64 go build -v -o demo-app
                """
            }
        } catch (exc) {
            println "构建失败 - ${currentBuild.fullDisplayName}"
            throw(exc)
        }
    }
    stage('构建 Docker 镜像') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: 'docker-auth',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASSWORD']]) {
            container('docker') {
                echo "3. 构建 Docker 镜像阶段"
                sh """
                docker login ${registryUrl} -u ${DOCKER_USER} -p ${DOCKER_PASSWORD}
                docker build -t ${image} .
                docker push ${image}
                """
            }
        }
    }
}
```

重新提交上面的流水线过后，最终我们会将应用镜像推送到镜像仓库中去。

**然后 Argo CD Image Updater 将会每 2 分钟从镜像仓库去检索镜像版本变化，一旦发现有新的镜像版本，它将自动使用新版本来更新集群内工作负载的镜像，并将镜像版本回写到 Git 仓库重去**。我们可以去查看 Argo CD Image Updater 的日志变化：

```
$ kubectl logs -f argocd-image-updater-56d94c674d-npgqp -n argocd
# ......
time="2023-09-19T06:39:12Z" level=info msg="Starting image update cycle, considering 1 annotated application(s) for update"
time="2023-09-19T06:39:16Z" level=info msg="Setting new image to cnych/devops-demo:a6268b3" alias=myalias application=devops-demo2 image_name=cnych/devops-demo image_tag=739a588 registry=
time="2023-09-19T06:39:16Z" level=info msg="Successfully updated image 'cnych/devops-demo:739a588' to 'cnych/devops-demo:a6268b3', but pending spec update (dry run=false)" alias=myalias application=devops-demo2 image_name=cnych/devops-demo image_tag=739a588 registry=
time="2023-09-19T06:39:16Z" level=info msg="Committing 1 parameter update(s) for application devops-demo2" application=devops-demo2
time="2023-09-19T06:39:16Z" level=info msg="Starting configmap/secret informers"
time="2023-09-19T06:39:17Z" level=info msg="Configmap/secret informer synced"
time="2023-09-19T06:39:17Z" level=info msg="Initializing http://gitlab.k8s.local/cnych/k8s-demo-config.git to /tmp/git-devops-demo23205764981"
time="2023-09-19T06:39:17Z" level=info msg="rm -rf /tmp/git-devops-demo23205764981" dir= execID=14972
time="2023-09-19T06:39:17Z" level=info msg="secrets informer cancelled"
time="2023-09-19T06:39:17Z" level=info msg="configmap informer cancelled"
time="2023-09-19T06:39:17Z" level=info msg=Trace args="[rm -rf /tmp/git-devops-demo23205764981]" dir= operation_name="exec rm" time_ms=4.474982
time="2023-09-19T06:39:17Z" level=info msg="git fetch origin --tags --force" dir=/tmp/git-devops-demo23205764981 execID=08213
time="2023-09-19T06:39:17Z" level=info msg=Trace args="[git fetch origin --tags --force]" dir=/tmp/git-devops-demo23205764981 operation_name="exec git" time_ms=492.78976600000004
time="2023-09-19T06:39:17Z" level=info msg="git config user.name argocd-image-updater" dir=/tmp/git-devops-demo23205764981 execID=35e12
time="2023-09-19T06:39:17Z" level=info msg=Trace args="[git config user.name argocd-image-updater]" dir=/tmp/git-devops-demo23205764981 operation_name="exec git" time_ms=1.4469750000000001
time="2023-09-19T06:39:17Z" level=info msg="git config user.email noreply@argoproj.io" dir=/tmp/git-devops-demo23205764981 execID=6515c
time="2023-09-19T06:39:17Z" level=info msg=Trace args="[git config user.email noreply@argoproj.io]" dir=/tmp/git-devops-demo23205764981 operation_name="exec git" time_ms=1.593801
time="2023-09-19T06:39:17Z" level=info msg="git checkout --force main" dir=/tmp/git-devops-demo23205764981 execID=e5492
time="2023-09-19T06:39:17Z" level=info msg=Trace args="[git checkout --force main]" dir=/tmp/git-devops-demo23205764981 operation_name="exec git" time_ms=5.05169
time="2023-09-19T06:39:17Z" level=info msg="git clean -fdx" dir=/tmp/git-devops-demo23205764981 execID=5cca4
time="2023-09-19T06:39:17Z" level=info msg=Trace args="[git clean -fdx]" dir=/tmp/git-devops-demo23205764981 operation_name="exec git" time_ms=1.8230989999999998
time="2023-09-19T06:39:17Z" level=info msg="git commit -a -F /tmp/image-updater-commit-msg2911699728" dir=/tmp/git-devops-demo23205764981 execID=ac1b3
time="2023-09-19T06:39:17Z" level=info msg=Trace args="[git commit -a -F /tmp/image-updater-commit-msg2911699728]" dir=/tmp/git-devops-demo23205764981 operation_name="exec git" time_ms=7.143674
time="2023-09-19T06:39:17Z" level=info msg="git push origin main" dir=/tmp/git-devops-demo23205764981 execID=136ad
time="2023-09-19T06:39:18Z" level=info msg=Trace args="[git push origin main]" dir=/tmp/git-devops-demo23205764981 operation_name="exec git" time_ms=874.7453360000001
time="2023-09-19T06:39:18Z" level=info msg="Successfully updated the live application spec" application=devops-demo2
time="2023-09-19T06:39:18Z" level=info msg="Processing results: applications=1 images_considered=1 images_skipped=0 images_updated=1 errors=0"
time="2023-09-19T06:41:18Z" level=info msg="Starting image update cycle, considering 1 annotated application(s) for update"
time="2023-09-19T06:41:21Z" level=info msg="Processing results: applications=1 images_considered=1 images_skipped=0 images_updated=0 errors=0"
time="2023-09-19T06:43:21Z" level=info msg="Starting image update cycle, considering 1 annotated application(s) for update"
# ......
```

另外需要注意的是在回写时，ArgoCD Image Updater 并不会直接修改仓库的 `values.yaml` 文件，而是会创建一个专门用于覆盖 Helm Chart values.yaml 的 `.argocd-source-devops-demo2.yaml` 文件。

![Alt Image Text](../images/ag1_1_17.png "Body image")

自动提交变更后，Argo CD 就会自动同步部署应用了。

![Alt Image Text](../images/ag1_1_18.png "Body image")

