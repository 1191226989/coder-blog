---
title: Golang项目GitlabCICD流水线配置
created: '2023-08-27T02:24:10.613Z'
modified: '2023-12-01T02:51:26.369Z'
---

# Golang项目GitlabCICD流水线配置

#### 参考文档
- https://docs.gitlab.com/
- https://docs.gitlab.cn/
- https://golangci-lint.run/

```yml
stages:
  - lint
  - test
  - build
  - deploy
```
流水线以job为单位运行，每个阶段可以定义多个job。同一阶段的job会并行执行。每个阶段是串行执行。
```sh
go test -short go list ./...
```

#### 全局变量
```yml
variables:
  IMAGE_GROUP: xxx
  NAME_SPACE: xxx
  GOPATH: ${CI_PROJECT_DIR}/.go
  GOMODCACHE: ${CI_PROJECT_DIR}/.go/pkg/mod
  GOCACHE: ${CI_PROJECT_DIR}/.go/.cache/go-build
  GOLANGCI_LINT_CACHE: ${CI_PROJECT_DIR}/.go/.cache/golangci-lint
```
自定义一些变量，在流水线执行过程中以环境变量的形式存在。
- GOPATH：指定GOPATH为项目目录下的`.go`，流水线缓存只能缓存项目目录下的文件。
- GOMODCACHE：Go依赖缓存。
- GOCACHE：`go build` 产生的缓存。
- GOLANGCI_LINT_CACHE：`golangci-lint` 代码质量检查工具也会产生缓存。

#### 公共模板

##### job公共模板
```yaml
.template:
  image: 172.30.3.150/xxx/go-tools:latest
  tags:
    - 172.30.3.219-runner
  interruptible: true
```
- go-tools：运行流水线的基础镜像，封装了go运行环境。

Dockerfile
```Dockerfile
FROM golang:1.18
USER root

ENV GOPATH /go
ENV PATH ${GOPATH}/bin:$PATH

# 设置私服
RUN go env -w GOPRIVATE=xxx.com
# 设置忽略私服的https证书校验
RUN go env -w GOINSECURE=xxx.com
RUN git config --global http.sslverify false
RUN git config --global https.sslverify false

RUN go env -w GOPROXY="https://goproxy.cn,direct"
RUN go env -w GO111MODULE=on

# install docker
RUN curl -O https://get.docker.com/builds/Linux/x86_64/docker-latest.tgz \
    && tar zxvf docker-latest.tgz \
    && cp docker/docker /usr/local/bin/ \
    && rm -rf docker docker-latest.tgz
```

- tags：指定运行流水线的gitlab runner。

- .template：流水线会将.开头代码作为模板，不会执行，其他的job会去extends此模板，达到代码复用的效果。

- interruptible：支持可中断，例如上一次流水线还没跑完，又触发了一次，这种情况下会取消上一次流水线。还需要勾选"设置"》"CI/CD"》"流水线通用设置"》"Auto-cancel redundant pipelines"

#### 缓存模板
```yaml
.go_cache:
  cache:
    key: go-cache-${CI_PROJECT_PATH_SLUG}
    paths:
      - .go
```
创建流水线缓存，以项目名称为key，缓存的目录为项目目录下的`.go`

#### lint阶段
##### 代码检测
```yaml
golangci_lint:
  stage: lint
  only:
    - merge_requests
    - /^release\/.*$/
  extends:
    - .go_cache
    - .template
  script:
    - make golangci_lint
```
- stage：指定此job属于lint阶段
- only: 触发方式为合并请求下的提交，和以release开头的分支上的提交。
- extends：继承模板，代码复用。
- script：job核心内容，指定此job运行的程序代码。

#### test阶段
单元测试、检查数据竞争
```yaml
unit_test:
  stage: test
  only:
    - merge_requests
    - /^release\/.*$/
  extends:
    - .go_cache
    - .template
  cache:
    policy: pull
  script:
    - make unit_test
```
cache.policy：默认缓存策略会pull and push，这个job中只使用了缓存，但没有产生缓存，所以不需要上传缓存。

#### build阶段
打包镜像的job模板
```yaml
.build_image:
  stage: build
  only:
    - tags
    - /^release\/.*$/
  extends:
    - .go_cache
    - .template
```

```yaml
build_xxx_admin_image:
  extends:
    - .build_image
  script:
    - make build_admin_app
    - make build_admin_image
    - make push_admin_image

build_xxx_interface_image:
  extends:
    - .build_image
  script:
    - make build_interface_app
    - make build_interface_image
    - make push_interface_image

build_xxx_job_image:
  extends:
    - .build_image
  script:
    - make build_job_app
    - make build_job_image
    - make push_job_image

build_xxx_task_image:
  extends:
    - .build_image
  script:
    - make build_task_app
    - make build_task_image
    - make push_task_image
```
一个build阶段有多个job

#### deploy阶段
部署到测试环境的job模板
```yaml
.deploy_to_test_k8s:
  stage: deploy
  only:
    - tags
    - /^release\/.*$/
  extends:
    - .template
  image:
    name: 172.30.3.150/xxx/kubectl:latest
    entrypoint: [ "" ]
```
image：指定运行此job的基础镜像，覆盖公共模板里的image。

172.30.3.150/xxx/kubectl：封装kubectl镜像，增加git、make命令。

Dockerfile
```dockerfile
FROM bitnami/kubectl

USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
    make git
```

#### 部署到测试环境job
```yaml
deploy_admin_to_test_k8s:
  extends:
    - .deploy_to_test_k8s
  needs: [ build_xxx_admin_image ]
  script:
    - make deploy_admin_to_test_k8s

deploy_interface_to_test_k8s:
  extends:
    - .deploy_to_test_k8s
  needs: [ build_xxx_interface_image ]
  script:
    - make deploy_interface_to_test_k8s

deploy_job_to_test_k8s:
  extends:
    - .deploy_to_test_k8s
  needs: [ build_xxx_job_image ]
  script:
    - make deploy_job_to_test_k8s

.deploy_task_to_test_k8s:
  extends:
    - .deploy_to_test_k8s
  needs: [ build_xxx_task_image ]
  script:
    - make deploy_task_to_test_k8s
```
needs：指定依赖关系，流水线默认是一个阶段一个阶段的运行。例如：部署admin模块只需要admin镜像制作好就可以执行，而不是等待整个build阶段运行完才执行。


#### 完整的.gitlab-ci.yml
```yaml
stages:
  - lint
  - test
  - build
  - deploy

#定义全局变量
variables:
  IMAGE_GROUP: xxx
  NAME_SPACE: xxx
  GOPATH: ${CI_PROJECT_DIR}/.go
  GOMODCACHE: ${CI_PROJECT_DIR}/.go/pkg/mod
  GOCACHE: ${CI_PROJECT_DIR}/.go/.cache/go-build
  GOLANGCI_LINT_CACHE: ${CI_PROJECT_DIR}/.go/.cache/golangci-lint

########################### 公共模板 ###########################

#job公共模板
.template:
  image: 172.30.3.150/xxx/go-tools:latest
  tags:
    - 172.30.3.219-runner
  interruptible: true

#缓存模板
.go_cache:
  cache:
    key: go-cache-${CI_PROJECT_PATH_SLUG}
    paths:
      - .go

########################### lint阶段 ###########################

#代码检测job
golangci_lint:
  stage: lint
  only:
    - merge_requests
    - /^release\/.*$/
  extends:
    - .go_cache
    - .template
  script:
    - make golangci_lint

########################### test阶段 ###########################

#单元测试、检查数据竞争job
unit_test:
  stage: test
  only:
    - merge_requests
    - /^release\/.*$/
  extends:
    - .go_cache
    - .template
  cache:
    policy: pull
  script:
    - make unit_test

########################### build阶段 ###########################

#打包镜像的job模板
.build_image:
  stage: build
  only:
    - tags
    - /^release\/.*$/
  extends:
    - .go_cache
    - .template

#打包镜像job
build_xxx_admin_image:
  extends:
    - .build_image
  script:
    - make build_admin_app
    - make build_admin_image
    - make push_admin_image

build_xxx_interface_image:
  extends:
    - .build_image
  script:
    - make build_interface_app
    - make build_interface_image
    - make push_interface_image

build_xxx_job_image:
  extends:
    - .build_image
  script:
    - make build_job_app
    - make build_job_image
    - make push_job_image

build_xxx_task_image:
  extends:
    - .build_image
  script:
    - make build_task_app
    - make build_task_image
    - make push_task_image

########################### deploy阶段 ###########################

#部署到测试环境的job模板
.deploy_to_test_k8s:
  stage: deploy
  only:
    - tags
    - /^release\/.*$/
  extends:
    - .template
  image:
    name: 172.30.3.150/xxx/kubectl:latest
    entrypoint: [ "" ]

#部署到测试环境job
deploy_admin_to_test_k8s:
  extends:
    - .deploy_to_test_k8s
  needs: [ build_xxx_admin_image ]
  script:
    - make deploy_admin_to_test_k8s

deploy_interface_to_test_k8s:
  extends:
    - .deploy_to_test_k8s
  needs: [ build_xxx_interface_image ]
  script:
    - make deploy_interface_to_test_k8s

deploy_job_to_test_k8s:
  extends:
    - .deploy_to_test_k8s
  needs: [ build_xxx_job_image ]
  script:
    - make deploy_job_to_test_k8s

.deploy_task_to_test_k8s:
  extends:
    - .deploy_to_test_k8s
  needs: [ build_xxx_task_image ]
  script:
    - make deploy_task_to_test_k8s
```

#### Makefile
具体代码剧本抽取到`Makefile`文件，`.gitlab-ci.yml`只做流水线控制。而且`Makefile`支持任意环境执行，不依赖于gitlab流水线（使用到gitlab流水线变量的除外），开发可在本地运行`Makefile`的剧本。

```sh
VERSION=$(shell git describe --tags --always)

#################################### 代码检查 ####################################

.PHONY: golangci_lint
golangci_lint:
    go mod tidy # 必须先下载依赖，否则golangci-lint会报：could not load export data: no export data for
    go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.45.0
    golangci-lint run -v --timeout=5m --color always --out-format colored-line-number

#################################### 单元测试 ####################################

.PHONY: unit_test
unit_test:
    go test -short `go list ./...`

#################################### 构建应用二进制执行文件 ####################################

.PHONY: build_all_app
build_all_app: build_admin_app build_interface_app build_job_app build_task_app

.PHONY: build_admin_app
build_admin_app:
    make build_app SUB_MODULE=admin

.PHONY: build_interface_app
build_interface_app:
    make build_app SUB_MODULE=interface

.PHONY: build_job_app
build_job_app:
    make build_app SUB_MODULE=job

.PHONY: build_task_app
build_task_app:
    make build_app SUB_MODULE=task

.PHONY: build_app
build_app:
    mkdir -p bin
    cd cmd/$(SUB_MODULE) && CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags "-X main.version=$(VERSION)" -o ../../bin/ && cd -

#################################### 打镜像 ####################################

.PHONY: build_all_image
build_all_image: build_admin_image build_interface_image build_job_image build_task_image

.PHONY: build_admin_image
build_admin_image:
    make build_image SUB_MODULE=admin

.PHONY: build_interface_image
build_interface_image:
    make build_image SUB_MODULE=interface

.PHONY: build_job_image
build_job_image:
    make build_image SUB_MODULE=job

.PHONY: build_task_image
build_task_image:
    make build_image SUB_MODULE=task

.PHONY: build_image
build_image:
    docker build --build-arg SUB_MODULE=$(SUB_MODULE) -t xxx-$(SUB_MODULE):$(VERSION) .

#################################### 推镜像到私仓 ####################################

.PHONY: push_all_image
build_all_image: push_admin_image push_interface_image push_job_image push_task_image

.PHONY: push_admin_image
push_admin_image:
    make push_image SUB_MODULE=admin

.PHONY: push_interface_image
push_interface_image:
    make push_image SUB_MODULE=interface

.PHONY: push_job_image
push_job_image:
    make push_image SUB_MODULE=job

.PHONY: push_task_image
push_task_image:
    make push_image SUB_MODULE=task

.PHONY: push_image
push_image:
    docker login "${DOCKER_REGISTRY_SERVER}" --username "${DOCKER_REGISTRY_USER}" --password "${DOCKER_REGISTRY_PASSWORD}"

    docker tag "xxx-$(SUB_MODULE):$(VERSION)" "${DOCKER_REGISTRY_ADDR}/${IMAGE_GROUP}/xxx-$(SUB_MODULE):$(VERSION)"
    docker push "${DOCKER_REGISTRY_ADDR}/${IMAGE_GROUP}/xxx-$(SUB_MODULE):$(VERSION)"
    docker rmi -f "${DOCKER_REGISTRY_ADDR}/${IMAGE_GROUP}/xxx-$(SUB_MODULE):$(VERSION)"

    docker tag "xxx-$(SUB_MODULE):$(VERSION)" "${DOCKER_REGISTRY_ADDR}/${IMAGE_GROUP}/xxx-$(SUB_MODULE):latest"
    docker push "${DOCKER_REGISTRY_ADDR}/${IMAGE_GROUP}/xxx-$(SUB_MODULE):latest"
    docker rmi -f "${DOCKER_REGISTRY_ADDR}/${IMAGE_GROUP}/xxx-$(SUB_MODULE):latest"

    docker rmi -f "xxx-$(SUB_MODULE):$(VERSION)"

#################################### 部署到k8s ####################################

.PHONY: deploy_all_to_test_k8s
deploy_all_to_test_k8s: deploy_admin_to_test_k8s deploy_interface_to_test_k8s deploy_job_to_test_k8s deploy_task_to_test_k8s

.PHONY: deploy_admin_to_test_k8s
deploy_admin_to_test_k8s:
    make deploy_to_test_k8s SUB_MODULE=admin

.PHONY: deploy_interface_to_test_k8s
deploy_interface_to_test_k8s:
    make deploy_to_test_k8s SUB_MODULE=interface

.PHONY: deploy_job_to_test_k8s
deploy_job_to_test_k8s:
    make deploy_to_test_k8s SUB_MODULE=job

.PHONY: deploy_task_to_test_k8s
deploy_task_to_test_k8s:
    make deploy_to_test_k8s SUB_MODULE=task

.PHONY: deploy_to_test_k8s
deploy_to_test_k8s:
    echo ${K8S_CONFIG_192} | base64 -id > /.kube/config
    sed -i "s/IMAGE_VERSION/$(VERSION)/g" deploy/$(SUB_MODULE)/test/*.yaml
    sed -i "s/NAME_SPACE/${NAME_SPACE}/g" deploy/$(SUB_MODULE)/test/*.yaml
    sed -i "s/DOCKER_REGISTRY_ADDR/${DOCKER_REGISTRY_ADDR}/g" deploy/$(SUB_MODULE)/test/*.yaml
    sed -i "s/IMAGE_GROUP/${IMAGE_GROUP}/g" deploy/$(SUB_MODULE)/test/*.yaml
    kubectl apply -f deploy/$(SUB_MODULE)/test
```

- VERSION：此变量用于构建镜像的版本号，如果当前分支有tag，此变量的值就为tag，没有tag值为commit hash标识。
 
- .PHONY：伪目标，可以防止在`Makefile`中定义的命令目标和工作目录下的实际文件出现名字冲突。
 
- DOCKER_REGISTRY_SERVER：需要在gitlab管理系统配置的变量（"设置"》"CI/CD"》"变量"），可根据实际情况放到不同级别下。例如：gitlab全局，项目群组，项目。