# **Gitlab Pipeline 复习**

### job

* 可以定义一个或多个作业(job)。
* 每个作业必须具有唯一的名称（不能使用关键字）。
* 每个作业是独立执行的。
* **每个作业至少要包含一个script**。

```
job1:
  script: "execute-script-for-job1"

job2:
  script: "execute-script-for-job2"
```

### **script**

```
script:
  - uname -a
  -  bundle exec rspec
```

### **`before_script`**

* 用于定义一个命令，该命令在每个作业之前运行。
* **`before_script`失败导致整个作业失败，其他作业将不再执行。作业失败不会影响`after_script`运行**。

### **`after_script`**

* 用于定义将在每个作业（包括失败的作业）之后运行的命令。
* 这必须是一个数组。
* 指定的脚本在新的shell中执行，与任何`before_script`或script脚本分开。

**可以在全局定义，也可以在job中定义。在job中定义会覆盖全局**

### **stages**

用于定义作业可以使用的阶段，**并且是全局定义的**。同一阶段的作业并行运行，不同阶段按顺序执行。

```
stages：
  - build
  - test
  - codescan
  - deploy
```

**未定义stages: 如果job没有定义stage则默认是test阶段。如果全局未定义stages,则按顺序运行 build,test,deploy。**



### **`. pre` & `.post`**

* `.pre` 始终是整个管道的第一个运行阶段，
* `.post` 始终是整个管道的最后一个运行阶段。

用户定义的阶段都在两者之间运行。`.pre`和`.post`的顺序无法更改。如果管道仅包含`.pre`或`.post`阶段的作业，则不会创建管道。

```
codescan:
  stage: .pre
  script:
    - echo "codescan"
```

### **stage**

**可能遇到的问题：阶段并没有并行运行。**

在这里我把这两个阶段在同一个runner运行了，所以需要修改runner每次运行的作业数量。默认是1，改为10.

### **variables**

定义变量，pipeline变量、job变量、Runner变量。**job变量优先级最大**。

### **tags-指定runner**

tags可让您使用指定了标签的跑步者来运行作业,此runner具有ruby和postgres标签。

```
job:
  tags:
    - ruby
    - postgres
```

**`allow_failure`**

`allow_failure` 允许作业失败，默认值为false 。

**启用后，如果作业失败，该作业将在用户界面中显示橙色警告. 但是，管道的逻辑流程将认为作业成功/通过，并且不会被阻塞**。

**假设所有其他作业均成功，则该作业的阶段及其管道将显示相同的橙色警告。但是，关联的提交将被标记为"通过"，而不会发出警告**。

```
job1:
  stage: test
  script:
    - execute_script_that_will_fail
  allow_failure: true
```

### **when-控制作业运行**

* `on_success`前面阶段中的所有作业都成功（或由于标记为`allow_failure`而被视为成功）时才执行作业。这是默认值。
* `on_failure`当前面阶段出现失败则执行。
* `always` 执行作业，而不管先前阶段的作业状态如何，放到最后执行。总是执行。
* `on_failure` 当前面阶段出现失败时执行。
* `manual` 手动执行作业。
* `delayed` 延迟执行作业。

* **manual 手动**

```
codescan:
  stage: codescan
  tags:
    - build
  script:
    - echo "codescan"
  when: on_success
```

```
unittest:
  stage: test
  script:
    - echo "run test"  
  when: delayed
  start_in: '5'
```

* **delayed 延迟**
 
delayed 延迟一定时间后执行作业

* **retry**

配置在失败的情况下重试作业的次数。

```
unittest:
  stage: test
  retry: 2
  script:
```
为了更好地控制retry哪些失败，可以是具有以下键的哈希值： 

**max ：最大重试次数 / when ：重试失败的案例.**

```
always ：在发生任何故障时重试（默认）.
unknown_failure ：当失败原因未知时。
script_failure ：脚本失败时重试。
api_failure ：API失败重试。
stuck_or_timeout_failure ：作业卡住或超时时。
runner_system_failure ：运行系统发生故障。
missing_dependency_failure: 如果依赖丢失。
runner_unsupported ：Runner不受支持。
stale_schedule ：无法执行延迟的作业。
job_execution_timeout ：脚本超出了为作业设置的最大执行时间。
archived_failure ：作业已存档且无法运行。
unmet_prerequisites ：作业未能完成先决条件任务。
scheduler_failure ：调度程序未能将作业分配给运行scheduler_failure。
data_integrity_failure ：检测到结构完整性问题。
```

```
...
when: delayed
retry:
  max: 2
  when: script_failure
```

### **timeout 超时**

```
test:
  script: rspec
  timeout: 3h 30m
```

### parallel


**配置要并行运行的作业实例数,此值必须大于或等于2并且小于或等于50。**

```
parallel: 5
```

```
unittest:
  stage: test
  tags:
    - build
  script:
    - ech "run test"  
  when: delayed
  start_in: '5'
  allow_failure: true
  retry:
    max: 2
    when: 
      - script_failure
```

### **only & except (逐步退出）**

only和except是两个参数用分支策略来限制jobs构建：

* only**定义哪些分支和标签的git项目将会被job执行**。
* except**定义哪些分支和标签的git项目将不会被job执行**

```
job:
  only:
    - main
```

```
job:
  # use regexp
  only:
    - /issue-.*$/
  # use special keyword
  except:
    - branches
```

### **rules**

* rules允许按顺序评估单个规则对象的列表，直到一个匹配并为作业动态提供属性.
* 请注意， **rules不能only/except与only/except组合使用**。


**可用的规则条款包括：**

* if （类似于only:variables ）
* changes （ only:changes 相同）
* exists

**if**

* 如果DOMAIN的值匹配，则需要手动运行。
* 不匹配`on_succes`s。
* **条件判断从上到下，匹配即停止**。
* 多条件匹配可以使用&& ||

```
rules:
  - if: '$DOMAIN == "example.com"'
     when: manual
  - when: on_success
```
```
rules:
  - if: '$DOMAIN == "example.com"'
    when: manual
  - if: '$DOMAIN == "example.com"'
    when: delayed
    start_in: '5'
  - when: on_success
```

### `rules-changes`-文件变化

* 接受文件路径数组。
* 如果提交中`*file`文件发生的变化则为`true`

```
rules:
    - changes:
      - Dockerfile
      when: manual
```

* **`rules:exists`** 

接受文件路径数组。**当仓库中存在指定的文件时操作**。

```
rules:
    - exists:
      - Dockerfile
      when: manual 
    - changes:
      - Dockerfile
      when: on_success
    - if: '$DOMAIN == "example.com"'
      when: on_success
    - when: on_success
```

* **rules-`allow_failure`**

* 使用`allow_failure: true`
* rules:在不停止管道本身的情况下允许作业失败或手动作业等待操作。

```
rules:
 - if: '$CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "main"'
   when: manual
   allow_failure: true
```
 
在此示例中，如果第一个规则匹配，则作业将具有以下`when: manual`和`allow_failure: true`。


### **workflow:rules创建管道**

* 顶级workf1ow关键字适用于整个管道，并将确定是否创建管道。
* when：可以设置为always或never， 如果未提供，则默认值always。


* 顶级`workflow`关键字适用于整个管道，并将确定是否创建管道。
* when：可以设置为always或never， 如果未提供，则默认值always

```
workflow:
  rules:
    - if: '$DOMAIN == "example.com"'
    - when: always
```

## **cache 关键字**

**不要使用缓存在阶段之间传递工件，因为缓存主要是存储编译项目所需的运行时依赖项。**

**如果在job范围之外定义了cache ，则意味着它是全局设置，所有job都将使用该定义**。如果未全局定义或未按job定义则禁用该功能。

* 存储编译项目所需的运行时依赖项，指定项目工作空间中需要在job之间缓存的文件或目录
* 全局cache定义在job之外，针对所有job生效。**job中cache优先于全局**。

* **`cache:paths`**

`$CI_PROJECT_DIR` 项目目录。在job build中定义缓存，将会缓存target目录下的所有`.jar`文件。

```
build:
  script: test
  cache:
    paths:
      - target/*.jar
```

当在全局定义了`cache:paths`会被job中覆盖。以下实例将缓存`binaries`目录。

```
cache:
  paths:
    - my/files

build:
  script: echo "hello"
  cache:
    key: build
    paths:
      - target/
```

**如何让不同的job缓存不同的cache呢？设置不同的`cache:key`。**

* cache:key 缓存标记

为缓存做个标记，可以配置job、分支为key来实现分支、作业特定的缓存。

为不同 job 定义了不同的 `cache:key` 时， 会为每个 job 分配一个独立的 cache。

**cache:key 变量可以使用任何预定义变量，默认default ，从GitLab 9.0开始，默认情况下所有内容都在管道和作业之间共享**。

**按照分支设置缓存**

```
cache:
  key: ${CI_COMMIT_REF_SLUG}
```


* `cache:key:fles ` —— 文件变化自动创建缓存

files：文件发生变化自动重新生成缓存(files最多指定两个文件)，提交的时候检查指定的文件。

**根据指定的文件生成密钥计算SHA校验和，如果文件未改变值为default**。

```
cache:
  key:
    files:
      - Gemfile.lock
      - package.json
  paths:
    - vendor/ruby
    - node_modules
```

* **`cache: key:prefix` -组合生成SHA校验和**

prefix: 允许给定prefix的值与指定文件生成的秘钥组合。

在这里定义了全局的cache，如果文件发生变化则值为 `rspec-xxx111111111222222` ，未发生变化为rspec-default。

```
cache:
  key:
    files:
      - Gemfile.lock
    prefix: ${CI_JOB_NAME}
  paths:
    - vendor/ruby
```

例如，添加`$CI_JOB_NAME prefix`将使密钥看起来像：

**`rspec-feef9576d21ee9b6a32e30c5c79d0a0ceb68d1e5`**


* `cache:policy `策略
	* **默认：在执行开始时下载文件，并在结束时重新上传文件。称为`pull-push`缓存策略**
	* `policy: pull` 跳过下载步骤
	* `policy: push` 跳过上传步骤

```
rspec:
  stage: test
  cache:
    key: gems
    paths:
      - vendor/bundle
    policy: pull
  script:
    - bundle exec rspec ...
```

### **artifacts/dependencies**

* `artifacts:paths`

```
artifacts:
  paths:
    - target/
```

* `artifacts:expose_as` -MR展示制品

关键字`expose_as`可用于在合并请求 UI中公开作业工件。

```
artifacts:
    expose_as: 'artifact 1'
    paths: 
      - path/to/file.txt
```

* `artifacts:name`

通过name指令定义所创建的工件存档的名称。可以为每个档案使用唯一的名称。

`artifacts:name`变量可以使用任何预定义变量。默认名称是artifacts，下载artifacts改为artifacts.zip。

```
job:
  artifacts:
    name: "$CI_JOB_NAME"
    paths:
      - binaries/
```

使用内部分支或标记的名称（仅包括binaries目录）创建存档，

```
job:
  artifacts:
    name: "$CI_COMMIT_REF_NAME"
    paths:
      - binaries/
```
**使用当前作业的名称和当前分支或标记（仅包括二进制文件目录）创建存档**

`name: "$CI_JOB_STAGE-$CI_COMMIT_REF_NAME"`

* `artifacts:when`  用于在作业失败时或成功而上传工件。
	* 	`on_success` 仅在作业成功时上载工件默认值。
	*  `on_failure`仅在作业失败时上载工件。
	*  always 上载工件，无论作业状态如何。

* `artifacts: expire_in` 制品保留时间,定义过期时间，则默认为30天。
* `artifacts:reports`  artifacts:reports:junit —— 单元测试报告

```
 artifacts:
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_NAME"
    when: on_success
    #expose_as: 'artifact 1'
    paths:
      - target/*.jar
      #- target/surefire-reports/TEST*.xml
    reports:
      junit: target/surefire-reports/TEST-*.xml
      cobertura: target/site/cobertura/coverage.xml
  coverage: '/Code coverage: \d+\.\d+/'
```

### **dependencies**

定义要获取工件的作业列表，只能从当前阶段之前执行的阶段定义作业。定**义一个空数组将跳过下载该作业的任何工件不会考虑先前作业的状态**，因此，如果它失败或是未运行的手动作业，则不会发生错误。

```
unittest:
  dependencies:
    - build
```

### **needs/artifacts**

* needs 并行阶段

可无序执行作业，无需按照阶段顺序运行某些作业，可以让多个阶段同时运行。

```
module-a-test:
  stage: test
  script: 
    - echo "hello3a"
    - sleep 10
  needs: ["module-a-build"]
```

**制品下载**

**在使用needs，可通过`artifacts: true`或`artifacts: false`来控制工件下载。默认不指定为true。**

```
 needs: 
    - job: "module-a-build"
      artifacts: true
```

相同项目中的管道制品下载,通过将project关键字设置为当前项目的名称，并指定引用，可以使用needs从当前项目的不同管道中下载工件

```
needs:
  - project: group/same-project-name
     job: build-1
     ref: other-ref
     artifacts: true
```

### **include/local**

```
include:
  local: 'ci/localci.yml'
```

**include: local-引入本地配置**

* file

引入 `demo/demo-java-service `项目 main 分支 .`gitlab-ci.yml` 配置



```
include:
  - project: demo/demo-java-service
    ref: main
    file: '.gitlab-ci.yml'
```

* include: template-引入官方配置

```
include:
  - template: Auto-DevOps.gitlab-ci.yml
```

### **extends - 继承作业配置**

```
testjob:
  extends: .tests
```

### **trigger 管道触发**

当Gitlab从trigger定义创建的作业启动时，将创建一个下游管道。允许创建多项目管道和子管道。将trigger与w`hen:manual`一起使用会导致错误

* 多项目管道：跨多个项目设置流水线，以便一个项目中的管道可以触发另一个项目中的管道。[微服务架构]
* 父子管道：在同一项目中管道可以触发一组同时运行的子管道，子管道仍然按照阶段顺序执行其每个作业，但是可以自由地继续执行各个阶段，而不必等待父管道中无关的作业完成。

```
staging:
  variables:
      ENVIRONMENT: staging
  stage: deploy
  trigger:
      project: gitlab-instance-8f6af96c/demo-java-service
      branch: main
      # strategy: depend
```

* project关键字，用于指定下游项目的完整路径。该branch关键字指定由指定的项目分支的名称。
* 使用variables关键字将变量传递到下游管道。全局变量也会传递给下游项目。
* 上游管道优先于下游管道。如果在上游和下游项目中定义了两个具有相同名称的变量，则在上游项目中定义的变量将优先。
* 默认情况下，一旦创建下游管道，trigger作业就会以success状态完成。
* strategy: depend将自身状态从触发的管道合并到源作业。

### **父子管道 类似于 Include**

父子流水线在一个项目中

```
staging2:
  variables:
    ENVIRONMENT: staging
  stage: deploy
  trigger: 
    include: ci/child01.yml  
    strategy: depend
```

### **services**

工作期间运行的另一个Docker映像，并link到image关键字定义的Docker映像。这样，您就可以在构建期间访问服务映像.

服务映像可以运行任何应用程序，但是最常见的用例是运行数据库容器，例如mysql 。与每次安装项目时都安装mysql相比，使用现有映像并将其作为附加容器运行更容易，更快捷

```
services:
  - name: mysql:latest
    alias: mysql-1
```
 
### **environment**

声明所部署的环境名称和访问地址，后续可以直接在gitlab 环境变量中查看。非常方便。

```
environment:
    name: production
    url: https://prod.example.com
```
 
### **inherit**

使用或禁用全局定义的环境变量（variables）或默认值(default)。

使用true、false决定是否使用，默认为true

```
inherit:
  default: false
  variables: false
```

继承其中的一部分变量或默认值使用list

```
inherit:
  default:
    - parameter1
    - parameter2
  variables:
    - VARIABLE1
    - VARIABLE2
```

