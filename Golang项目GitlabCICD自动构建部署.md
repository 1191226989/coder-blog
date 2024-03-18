---
title: Golang项目GitlabCICD自动构建部署
created: '2023-08-27T02:24:10.613Z'
modified: '2023-12-01T02:51:26.369Z'
---

# Golang项目GitlabCICD自动构建部署

#### 参考文档
- https://docs.gitlab.com/
- https://docs.gitlab.cn/
- https://golangci-lint.run/

#### 直接构建部署

1. 首先，在GitLab上创建一个新的项目，将Golang应用程序代码上传到该项目中。

2. 配置CI/CD Pipeline：在项目中创建一个`.gitlab-ci.yml`文件，用于定义CI/CD Pipeline的配置。在该文件中可以指定构建、测试、部署的步骤，并配置部署到目标服务器的命令。
```yaml
image: golang:1.18

stages:
  - build
  - deploy

build:
  stage: build
  script:
    - go get -d -v ./...
    - go build -v

deploy:
  stage: deploy
  script:
    - scp ./your-app user@your-server:/path/to/deploy/
    - ssh user@your-server 'systemctl restart your-app.service' # 也可以用docker方式部署运行项目
```

3. 配置服务器和部署环境：在部署目标服务器上配置运行环境。如果是docker方式运行则需要安装docker环境。

4. 配置SSH密钥：在GitLab项目设置中，添加部署目标服务器的SSH密钥，以便GitLab能够通过SSH连接到服务器进行部署操作。

5. 触发CI/CD Pipeline：提交代码变更后，GitLab会自动触发CI/CD Pipeline执行，自动构建Golang应用程序并部署到目标服务器。


#### docker方式构建和部署

1. 配置Docker仓库：在GitLab项目设置中，启用Docker仓库功能。可以在项目设置中的“CI/CD”菜单下找到“Docker仓库”选项，并启用该功能。也可以使用其他开源docker仓库。

2. 创建Dockerfile：在GitLab项目中创建一个`Dockerfile`，用于定义如何构建Docker镜像。
```dockerfile
FROM golang:1.18-alpine

WORKDIR /app

COPY . .

RUN go build -o myapp

CMD ["./myapp"]
```

3. 配置CI/CD Pipeline：在项目中创建一个`.gitlab-ci.yml`文件，用于定义CI/CD Pipeline的配置。
```yaml
image: docker:stable

services:
  - docker:dind

stages:
  - build
  - deploy

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE .
    - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE

deploy:
  stage: deploy
  script:
    - docker run -d $CI_REGISTRY_IMAGE # 也可以登录其他服务器以docker方式部署运行项目
```

4. 触发CI/CD Pipeline：提交代码变更后，GitLab会自动触发CI/CD Pipeline执行，自动构建Docker镜像并部署Golang应用程序。