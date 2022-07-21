

```
 docker exec -it aed10f1e614c sh
# gitlab-runner verify
Runtime platform                                    arch=amd64 os=linux pid=561 revision=76984217 version=15.1.0
Running in system-mode.                            
                                                   
ERROR: Verifying runner... is removed               runner=faDQf6_F
ERROR: Verifying runner... is removed               runner=MsUXFgx3
Verifying runner... is alive                        runner=Df8p4oox
FATAL: Failed to verify runners  
```


```
before_script:
  - echo "before-script!!"

variables:
  DOMAIN: example.com


stages:
  - build
  - test
  - codescan
  - deploy
  
build:
  before_script:
    - echo "before-script in job"
  stage: build
  tags:
    - build
  only:
    - master
  script:
    - ls
    - id
    - mvn clean package -DskipTests
    - ls target
    - echo "$DOMAIN"
    - false && true ; exit_code=$?
    - if [ $exit_code -ne 0 ]; then echo "Previous command failed"; fi;
    - sleep 2;
  after_script:
    - echo "after script in job"
  cache: 
    key: build
    paths:
      - .m2/repository/
      - target/



unittest:
  stage: test
  tags:
    - build
  only:
    - master
  script:
    - echo "run test"
    - ls target
  retry:
    max: 2
    when:
      - script_failure
  
    
interfacetest:
  stage: test
  tags:
    - build
  only:
    - master
  script:
    - echo "run test"
    - sleep 2;


deploy:
  stage: deploy
  tags:
    - deploy
  only:
    - master
  script:
    - echo "hello deploy"
    - sleep 2;
  #when: manual
  allow_failure: true
  
timedrollout:
  stage: deploy
  script: 
    - echo 'Rolling out 10% ...'
  when: delayed
  start_in: '2'
  
codescan:
  stage: codescan
  tags:
    - build
  script:
    - echo "codescan"
    - sleep 1;
  #parallel: 5
  rules:
    - exists:
      - Jenkinsfile
      when: on_success 
    - changes:
      - Jenkinsfile
      when: on_success
    - if: '$DOMAIN == "example.com"'
      when: on_success
    - when: on_success

after_script:
  - echo "after-script"
  - ech
```