---
title: PHP使用GitlabCICD部署项目方案
created: '2023-08-27T02:24:10.613Z'
modified: '2023-12-01T02:51:26.369Z'
---

# PHP使用GitlabCICD部署项目方案

#### 参考文档
- https://docs.gitlab.com/
- https://docs.gitlab.cn/

#### gitlab-runner执行器为shell

1. 首先，在GitLab上创建一个新的项目，将PHP应用程序代码上传到该项目中。

2. 配置CI/CD Pipeline：在项目中创建一个`.gitlab-ci.yml`文件，用于定义CI/CD Pipeline的配置。可以指定构建、测试、部署的步骤，并配置部署到服务器的命令。
```yaml
image: php:7.4

stages:
  - build
  - deploy

build:
  stage: build
  script:
    - composer install
    - phpunit

deploy:
  stage: deploy
  script:
    - ssh user@your-server1 'cd /path/to/deploy && git pull origin master'
    - ssh user@your-server2 'cd /path/to/deploy && git pull origin master'
```

3. 在部署目标服务器上，配置PHP运行环境，确保服务器上安装了PHP解释器、Web服务器（如Apache、Nginx）、数据库等必要的组件。

4. 配置SSH密钥：在GitLab项目设置中，添加部署目标服务器的SSH密钥，以便GitLab能够通过SSH连接到服务器进行部署操作。

5. 触发CI/CD Pipeline：提交代码变更后，GitLab会自动触发CI/CD Pipeline执行，自动构建、测试和部署PHP应用程序。


#### gitlab-runner执行器为docker

1. 配置Docker仓库：首先，在GitLab项目设置中，启用Docker仓库功能。可以在项目设置中的“CI / CD”菜单下找到“Docker仓库”选项，并启用该功能。

2. 创建Dockerfile：在GitLab项目中创建一个`Dockerfile`，用于定义如何构建Docker镜像。
```dockerfile
FROM php:7.4-apache

COPY . /var/www/html

RUN docker-php-ext-install mysqli pdo_mysql

EXPOSE 80

```

3. 配置CI/CD Pipeline：在项目中创建一个`.gitlab-ci.yml`文件，用于定义CI/CD Pipeline的配置。可以指定构建Docker镜像并将其推送到GitLab的Docker仓库，以及部署镜像到目标服务器。

```yaml
image: docker:stable

before_script:
  - ls
  - docker version
  - export env="dev"
  - date

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
  only:
    - master

deploy:
  stage: deploy
  script:
    - docker run -d -p 80:80 $CI_REGISTRY_IMAGE
  only:
    - master
  tags:
    - php-web1
    - php-web2
    - php-web3
```
在上面的示例中，build阶段用于构建Docker镜像并推送到GitLab的Docker仓库，deploy阶段用于部署镜像到目标服务器。

4. 触发CI/CD Pipeline：提交代码变更后，GitLab会自动触发CI/CD Pipeline执行，自动构建Docker镜像并部署PHP应用程序。