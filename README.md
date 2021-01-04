## 基于springboot构建流水线示例项目

Jenkinsfile in SCM 意为将 Jenkinsfile 文件本身作为源代码管理 (Source Control Management) 的一部分，根据该文件内的流水线配置信息快速构建工程内的 CI/CD 功能模块，比如阶段 (Stage)，步骤 (Step) 和任务 (Job)。因此，在代码仓库中包含 Jenkinsfile。

## 项目介绍 

#### 本项目为kubesphere 基于springboot构建流水线示例项目，具体参见kubesphere  [V2.1教程](https://v2-1.docs.kubesphere.io/docs/zh-CN/quick-start/devops-online/),[V3.0教程](https://kubesphere.com.cn/docs/devops-user-guide/how-to-use/create-a-pipeline-using-jenkinsfile/)

项目中包含**Jenkinsfile in SCM** :Jenkinsfile-online文件（Jenkinsfile in SCM 意为将 Jenkinsfile 文件本身作为源代码管理 (Source Control Management) 的一部分)，kubesphere 内置Jenkins容器，**Jenkins**可以根据该文件内的流水线配置信息快速构建工程内的 CI/CD 功能模块，比如阶段 (Stage)，步骤 (Step) 和任务 (Job)。

 Jenkinsfile 来创建流水线，流水线共包括 8 个阶段，最终将演示示例部署到 KubeSphere 集群中的开发环境和生产环境且能够通过公网访问。 仓库中的 dependency 分支为缓存测试用例，测试方式与 master 分支类似，对 dependency 的多次构建可体现出利用缓存可以有效的提升构建速度。

## 项目使用

* 项目完成fork后，根据教程修改 Jenkinsfile-online中的环境变量为您自己值。



* 根据教程，使用项目管理员 `project-admin`账号登录 KubeSphere，在之前创建的企业空间 (demo-workspace) 下，点击 **项目 → 创建**，创建两个 **资源型项目** `kubesphere-sample-dev` 、kubesphere-sample-prod

  * 名称：固定为 `kubesphere-sample-dev`，kubesphere-sample-prod，若需要修改项目名称则需在本项目中的 [[deploy/dev-ol/](deploy/dev-ol/)] 、 [[deploy/prod-ol/](deploy/prod-ol/)]中修改 namespace 属性

  ##   Jenkinsfile-online文件介绍

  考虑到初学者可能对Jenkins文件不熟悉，对此文件进行介绍，方便您理解我们的流水线做了什么.

  ``` yaml
  pipeline {
    agent {
      node {
        label 'maven'   // 定义流水线的代理为 maven，kubesphere内置了四个默认代理，在目前版本当中我们内置了 4 种类型的 podTemplate，base、						//	nodejs、maven、go，并且在 Pod 中提供了隔离的 Docker 环境。具体参见官方文档
      }
    }
  
      parameters {
          string(name:'TAG_NAME',defaultValue: '',description:'') //定义 流水线描述
      }
          environment {                                        //定义流水线环境变量
          DOCKER_CREDENTIAL_ID = 'dockerhub-id'
          GITHUB_CREDENTIAL_ID = 'github-id'
          KUBECONFIG_CREDENTIAL_ID = 'demo-kubeconfig'
          REGISTRY = 'docker.io'
          DOCKERHUB_NAMESPACE = 'docker_username'
          GITHUB_ACCOUNT = 'kubesphere'
          APP_NAME = 'devops-java-sample'
      }
  ```

  **[Jenkins Agent 说明]( https://v2-1.docs.kubesphere.io/docs/zh-CN/devops/jenkins-agent/)**

  * **第一步**检出代码

  ```yaml
       stages {
              stage ('checkout scm') {
                  steps {
                      checkout(scm)
                  }
              }
  ```

  * **第二步** 执行单元测试

  ```yaml
        stage ('unit test') {
              steps {
                  container ('maven') {
                      sh 'mvn clean  -gs `pwd`/configuration/settings.xml test'
                  }
              }
          }
  ```

  * **第三步** 执行单元测试

  ```yaml
      stage('push latest'){
             when{
               branch 'master'
             }
             steps{
                  container ('maven') {
                    sh 'docker tag  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest '
                    sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest '
                  }
             }
          }
  
  ```

  * **第四步** 编译并推送

  ```yaml
          stage ('build & push') {
              steps {
                  container ('maven') {
                      sh 'mvn  -Dmaven.test.skip=true -gs `pwd`/configuration/settings.xml clean package'
                      sh 'docker build -f Dockerfile-online -t $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER .'
                      withCredentials([usernamePassword(passwordVariable : 'DOCKER_PASSWORD' ,usernameVariable : 'DOCKER_USERNAME' ,credentialsId : "$DOCKER_CREDENTIAL_ID" ,)]) {
                          sh 'echo "$DOCKER_PASSWORD" | docker login $REGISTRY -u "$DOCKER_USERNAME" --password-stdin'
                          sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER'
                      }
                  }
              }
          }
  ```

    * **第五步** 推送至docker hub latest版本

  ```yaml
      stage('push latest'){
             when{
               branch 'master'
             }
             steps{
                  container ('maven') {
                    sh 'docker tag  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest '
                    sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest '
                  }
             }
          }
  
  ```

    * **第六步** 弹出审核确认，是否部署到开发环境

  ```yaml
   stage('deploy to dev') {
            when{
              branch 'master'
            }
            steps {
              input(id: 'deploy-to-dev', message: 'deploy to dev?')
              kubernetesDeploy(configs: 'deploy/dev-ol/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
            }
          }
          stage('push with tag'){
            when{
              expression{
                return params.TAG_NAME =~ /v.*/
              }
            }
            steps {
                container ('maven') {
                  input(id: 'release-image-with-tag', message: 'release image with tag?')
                    withCredentials([usernamePassword(credentialsId: "$GITHUB_CREDENTIAL_ID", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                      sh 'git config --global user.email "kubesphere@yunify.com" '
                      sh 'git config --global user.name "kubesphere" '
                      sh 'git tag -a $TAG_NAME -m "$TAG_NAME" '
                      sh 'git push http://$GIT_USERNAME:$GIT_PASSWORD@github.com/$GITHUB_ACCOUNT/devops-java-sample.git --tags --ipv4'
                    }
                  sh 'docker tag  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$TAG_NAME '
                  sh 'docker push  $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$TAG_NAME '
            }
            }
          }
  ```

    * **第七步** 部署到生产环境

  ```yaml
        stage('deploy to production') {
            when{
              expression{
                return params.TAG_NAME =~ /v.*/
              }
            }
            steps {
              input(id: 'deploy-to-production', message: 'deploy to production?')
              kubernetesDeploy(configs: 'deploy/prod-ol/**', enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
            }
          }
  ```

  
