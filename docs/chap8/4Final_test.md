# **Gitlab 认证考题**

## 笔试考题

1. True/False: Continuous Delivery and Continuous Deployment are interchangeable terms with the same definition
	* **False**
2. What type of runner can be used by any project?
	* **Shared**
3. Which GitLab tiers have access to the Operations Dashboard? (check all that apply)
	* **GitLab Premium**
	* **GitLab Ultimate**
4. What parameter is used to tell the pipeline which runners to execute on?
	* **Tags**
5. What allows you to start a downstream pipeline?
	* **Trigger**
6. Which of the following keywords are able to be used for Job names?
	* **test**
7. True/False: The two main foundations of a CI/CD Pipeline at GitLab are a `.gitlab-ci.yml` file and a
GitLab Runner.
	* **True**
8. Which executor directly run commands as if writing them into terminal (bash or sh) or command prompt (cmd)? 
	* **Shell**

9. Which of the following benefits of Artifacts does not apply when using GitLab?
	* Can be used on subsequent jobs if they have been added to the current active pipeline
	* Can have an expiration value to control disk usage
	* Used for stage results that will be passed between stages
	* **All of the above**
10. What parameter determines where an app is deployed?
	* **Environments**
11. What GitLab stage houses Source Code Management and Web IDE?
	* **Create**
12. What GitLab feature allows you to publish static websites directly from a repository in GitLab?
	* **GitLab Pages**
13. Which of the following is always a component of a gitlab-ci.yml?
	* **build**
14. Which of the following is NOT needed to run a CI/CD Pipeline in GitLab?
	* **Wiki**
15. True/False: The rules keyword is a way to set job policies that determine whether or not jobs are
added to pipelines.
	* **True**


## Gitlab CI/CD考试题

1.如果Runner服务器发生切换，前面阶段打包好的文件在下一阶段找不到，应该如何处理 ？
	
* **用artifacts缓存作业产物，使用needs获取作业产物**

> 各阶段可以通过artifacts将作业产物缓存至阶段，**后续阶段可通过needs从阶段缓存中获取作业产物，适用于跨runner执行流水线的场景**
> 
> **Cache适用于单runner执行的场景**

2.以上脚本，表示执行什么操作 ？

```
script:
- mvn compile -e sonar:sonar
-Dsonar.projectKey=${SONAR_PROJECT_KEY}
-Dsonar.projectName=${SONAR_PROJECT_NAME}
-Dsonar.host.url=${SONAR_HOST_URL}
-Dsonar.login=${SONAR_LOGIN_TOKEN}
```

* **代码扫描**

> **maven项目的代码扫描脚本**

3.根据以上配置，可以看出流水线有几个阶段？

```
stages:
- build
- test
- sonarscan
- package
- deploy
- release
```

> **6**

4.`docker push ${IMAGE_NAME}`  以上配置，在做什么操作？

> **推送镜像**

5.以上配置，SONAR_PROJECT_PATH 是什么意思？

```
variables:
SONAR_PROJECT_KEY: "JK-jobpoint-employee"
SONAR_PROJECT_NAME: "JK-积分系统-员工端"
SONAR_PROJECT_PATH: "./src"
```

以上配置，`SONAR_PROJECT_PATH` 是什么意思？

> **指定代码扫描路径**

6.常见的流水线的触发方式有哪几种 

* **A 提交代码**
* **B 定时触发**
* **C URL触发**
* **D 手动触发**

7.关于`Gitlab-Runner-Tag`分布作用说明，哪些是正确的？

* **A `$CI_PROJECT_NAME`**
* **B `$CI_PROJECT_PATH`**

> **CI开头的变量为gitlab系统变量**

8.以上配置，表示缓存作业产物

```
 needs:
- job: "package-job"
   artifacts: true
```

**以上配置，表示缓存作业产物**

> 获取作业产物

9.`sonar-tag`用于静态扫描的作业 

* **对**

10.使用tag的主要目的是对不同的jobs进行分流，在有多个不同类型job同时需要执行时，tags让不同服务器负责不同的任务。

* **对**

11.docker与shell类型runner的主要区别在于shell无法使用image和services 

* 对

12.` docker build -t ${IMAGE_NAME} -f ${DOCKER_FILE_PATH} .` 表示根据dockerfile文件推送镜像至harbor服务器 

* **错**

> 根据dockerfile文件构建本地镜像

13.表示：提交代码触发流水

```
rules:
- if: $CI_COMMIT_TAG
```

* **错**

> 打tag触发流水线


14.在Gitlab CI/CD中，不可以将用户名、密码等私密信息设置成变量 

* **错**

15.软件包注册表，可用来存放归档制品

* **对**

16.Gitlab部署CI默认使用`.gitlab-ci.yml`

* **对**



## GitLab Certified CI/CD Associate Hands On Assessment 

* GitLab Docs- https://docs.gitlab.com/ee/README.html 
* Add a `gitlab-ci.yml` file with test, build, review, and deploy stages
* Add an image tag to your pipeline
* Define a default environment parameter for your pipeline
* Add environment variables to your pipeline
* Scaffold out a job policy pattern that uses feature branches and tags to gate review, release, and staging/production job execution
* Dockerize the application and publish a Docker image to the built-in GitLab Docker Registry.
* Enable SAST on your pipeline


```
image: registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image:latest

variables:
  AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID
  AWS_SECRET_ACCESS_KEY: $AWS_SECRET_ACCESS_KEY

stages:         # List of stages for jobs, and their order of execution
  - build
  - test
  - sast 
  - review
  - deploy
  - release
  

build-job:       # This job runs in the build stage, which runs first.
  stage: build
  script:
    - echo $AWS_ACCESS_KEY_ID
    - echo "Compiling the code..."
    - echo "Compile complete."

build-docker:
  image: docker:20.10.17
  stage: build
  services:
    - name: docker:20.10.17-dind
      alias: thedockerhost
  variables:
    IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
    DOCKER_HOST: tcp://thedockerhost:2375/
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: ""
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

unit-test-job:   # This job runs in the test stage.
  stage: test    # It only starts when the job in the build stage completes successfully.
  script:
    - echo "Running unit tests... This will take about 60 seconds."
    - sleep 10
    - echo "Code coverage is 90%"

lint-test-job:   # This job also runs in the test stage.
  stage: test    # It can run at the same time as unit-test-job (in parallel).
  script:
    - echo "Linting code... This will take about 10 seconds."
    - sleep 10
    - echo "No lint issues found."

review-job:       
  stage: review
  script:
    - echo "Reviewing the code..."
    - echo "Review complete."
  only:
  - feature*
  - tags

deploy-job:      # This job runs in the deploy stage.
  stage: deploy  # It only runs when *both* jobs in the test stage complete successfully.
  script:
    - echo "Deploying application..."
    - echo "Application successfully deployed."
  environment:
    name: production
    url: https://gitlab.cn/

release-job:       
  stage: release
  script:
    - echo "Releasing the code..."
    - echo "Release successfully complete."
  only:
  - feature*
  - tags

production-job:       
  stage: release
  script:
    - echo "Releasing the code on production"
    - echo "Release successfully complete."
  only:
  - feature*
  - tags

staging-job:       
  stage: release
  script:
    - echo "Releasing the code on stage"
    - echo "Release successfully complete."
  only:
  - feature*
  - tags

sast-job:
  stage: sast
  image: docker:20.10.17-dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_HOST: tcp://thedockerhost:2375/
    DOCKER_TLS_CERTDIR: ""
  allow_failure: true
  artifacts:
    reports:
      sast: gl-sast-report.json
  services:
    - name: docker:20.10.17-dind
      alias: thedockerhost
  script:
    # - export SAST_VERSION=$(echo "$CI_SERVER_VERSION" | sed 's/^\([0-9]*\)\.\([0-9]*\).*/\1-\2-stable/')
    - docker run 
        --env SAST_CONFIDENCE_LEVEL="${SAST_CONFIDENCE_LEVEL:-3}" 
        --env SAST_DISABLE_REMOTE_CHECKS="${SAST_DISABLE_REMOTE_CHECKS:-false}" 
        --volume "$PWD:/code" 
        --volume /var/run/docker.sock:/var/run/docker.sock 
        "registry.gitlab.com/gitlab-org/security-products/sast:latest" /app/bin/run /code
```