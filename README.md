# Automated-Release

Kubernetes + CI/CD 自动化发布项目

## 项目简介

通过 GitLab + Jenkins + Docker + Kubernetes 实现一个简单的微服务项目从代码提交、自动构建镜像、推送至镜像仓库、K8s 自动部署上线的完整 CI/CD 流水线。

## 项目架构图

开发者提交代码（GitLab）
           ↓
       Jenkins拉取代码
           ↓
     构建Docker镜像
           ↓
     推送至Harbor仓库
           ↓
   Jenkins触发K8s部署更新
           ↓
       应用上线 & 回滚支持

## 所需环境

组件    版本参考    用途
Ubuntu    20.04+    系统环境
GitLab    自建或GitLab.com    Git仓库
Jenkins    最新 LTS    自动构建流水线
Docker    20+    镜像构建与部署
Harbor    可选    镜像私有仓库
Kubernetes    1.25+    容器编排
Helm    可选    应用模板化部署
Nginx    可选    对外服务入口（Ingress）

## 项目实施步骤

### 第一步：部署Gatlab和Jenkins

- 安装Gatlab
- 安装Jenkins，并安装如下插件
  - Gatlab Plugin
  - Docker Pieline
  - Kubernetes Plugin
  - Git Parameter Plugin

配置 Jenkins 凭据（GitLab Token、Harbor 密码等）

### 第二步：编写微服务项目

```
myapp/
├── Dockerfile
├── k8s/
│   └── deployment.yaml
├── jenkins/
│   └── Jenkinsfile
├── src/
│   └── app.py
```

### 第三步：编写Jenkinsfile（流水线核心）

```
pipeline {
  agent any

  environment {
    IMAGE_NAME = "harbor.mydomain.com/devops/myapp"
    KUBECONFIG_CREDENTIAL_ID = 'kubeconfig-cred'
  }

  stages {
    stage('拉取代码') {
      steps {
        git credentialsId: 'gitlab-cred', url: 'https://gitlab.com/yourgroup/myapp.git'
      }
    }

    stage('构建镜像') {
      steps {
        sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER .'
      }
    }

    stage('推送镜像') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'harbor-cred', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
          sh '''
            echo "$PASSWORD" | docker login harbor.mydomain.com -u "$USERNAME" --password-stdin
            docker push $IMAGE_NAME:$BUILD_NUMBER
          '''
        }
      }
    }

    stage('部署到K8s') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-cred', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$KUBECONFIG_FILE
            sed "s|{{IMAGE}}|$IMAGE_NAME:$BUILD_NUMBER|g" k8s/deployment.yaml | kubectl apply -f -
          '''
        }
      }
    }
  }

  post {
    success {
      echo '发布成功！'
    }
    failure {
      echo '构建失败，请检查日志！'
    }
  }
}

```

### 第四步：Kubernetes部署模板

`k8s/deployment.yaml`

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: myapp
      template:
        metadata:
          labels:
            app: myapp
        spec:
          containers:
          - name: myapp
            image: {{IMAGE}}
            ports:
            - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: myapp
    spec:
      selector:
        app: myapp
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
    

## 项目亮点

- ✅ 真正实现“代码提交即上线”

- ✅ 可实现版本号控制、失败回滚

- ✅ 可扩展通知功能（企业微信/钉钉）

- ✅ 可集成 Helm 模板化部署