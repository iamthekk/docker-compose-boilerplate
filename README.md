# docker-compose-boilerplate

> **基本特性：**
> 1. 快捷部署多人nginx+php的开发测试环境，也可以扩展构建其他语言；
> 2. 基于Docker和docker-compose，不依赖K8S等高级编排工具，成本低廉、部署简单；
> 3. Docker内置集成jenkins，一键添加开发测试角色，无需额外配置；
> 4. 支持微服务架构，适用于小公司or敏捷项目团队，**也可以作为Docker学习入门的case**；
 
## 一、背景

在角色分工明确的团队里，什么样的条件才算是最优雅的联调和测试环境？在大厂里肯定都有很多高级的解决方案，比如这些：
- [docker搭建大规模测试环境的实践](https://yq.aliyun.com/articles/163420) / [测试开发之路--k8s 下的大规模持续集成与环境治理](https://testerhome.com/topics/15058)
- [DevOps落地实践：BAT系列：CICD：iPipe vs CCI](https://blog.csdn.net/liumiaocn/article/details/77869653)
- [阿里 DevOps 转型实践](http://www.infoq.com/cn/presentations/ali-devops-transformation-practice)

大型团队的合作框架下，必须依赖更复杂的DevOps架构（参考：[DevOps详解
](http://www.infoq.com/cn/articles/detail-analysis-of-devops)）。但对于成员不多、负责的Web项目工程量也不大的团队，面临的问题肯定也更单纯：
1. 前后端角色工程解耦，开发环境分离；
2. 工程师只关注业务逻辑本身，持续集成；
3. 环境和角色一键创建、一键更新、一键销毁，环境之间不受影响；

即便是只有这些需求，在以往的“开发机”的联调环境里，一旦需要添加开发或者测试人员，或者需要更新nginx的配置，再或者需要更新PHP、Nodejs的版本……对于测试环境的维护来说都是很痛苦的。

## 二、快速开始

**注意：** 当前部署方案仅依赖：`Docker`,`Docker-compose`,`git`

### 1、下载代码

```
$ git clone https://github.com/xiongwilee/docker-compose-boilerplate.git
```

### 2、添加测试用户`demo`

```
$ cd docker-compose
$ sh build.sh -u demo -m admin:master
```

此时，在`app/`会创建`demo`目录，在`nginx/conf.d`会创建`demo.conf`文件。

### 3、启动服务

```
$ docker-compose up -d
```

此时，再执行`docker-compose ps`会发现创建了三个镜像。

配置hosts使`sample.demo.testdomain.com`指向当前机器，然后访问http://sample.demo.testdomain.com 返回`phpinfo()`信息，说明创建成功。

## 三、部署架构说明

> **TIPS:** 这个方案仅适用于小公司or敏捷项目团队联调测试环境的部署，同时也可以作为Docker学习入门的case，并不适用于有一定规模的生产环境。

在“开发机”上仅仅安装docker、docker-compose、git之后就能跑起来Nginx、PHP的应用，当然得益于docker容器化的思想。其实这个的实现也仅仅利用了容器化的这个特性，最终docker-compose打包的整个服务会长驻内存，无需太多的管理成本。

最终的实现还具备两个特点：
1. 基于这个实现的boilerplate你可以轻易的迁移到其他项目，以及其他语言；
2. 每个`sample`管理每个应用的仓储地址、环境变量配置、更新代码后的钩子等操作；

其实现原理为：**通过脚本文件，管理docker-compose隐射到宿主机的配置、源码，同时将docker-compose暴露出来以实现服务的管理**。架构图如下：

![](http://wx2.sinaimg.cn/large/7171171cgy1fuajq0ln4xj20hs0hst8p.jpg)

### 1、docker-compose配置文件：`docker-compose.yml`

先看docker-compose的配置文件`docker-compose.yml`（篇幅原因，删掉了一部分配置）：

```yaml
version: '3'
services:
	# 所有的PHP环境构建在app容器里
    php:
        build: ./php
        expose:
            - "9000"
    # nginx容器
    nginx:
        build: ./nginx
        # 端口映射
        ports:
            - "80:80"
        # 依赖关系声明，先跑php所有服务
        depends_on:
            - "php"
    # jenkins容器
    jenkins:
        image: jenkins:latest
        ports:
            - "8080:8080"
            - "50000:50000"
```

这其实就是一个普通的PHP开发环境示例：可以看到就`php`、`nginx`、`jenkins`三个基本容器，除了`jenkins`，其他的容器均使用Dockerfile（build配置）来构建。

### 2、构建脚本：`build.sh`

由于在docker中实现了nginx配置文件及php源码文件的映射到宿主机，需要通过管理宿主机上文件就可以管理代码的发布和部署了，`build.sh`就是用来做这件事情的。

当然了，如果需要在部署代码完成之后，做重启、编译等操作，通过sample目录下的钩子就可以实现了。

具体实现可以参考`build.sh`源码。

## 四、详细配置

### 1、开发测试环境泛域名解析

### 2、`docker-compose.yml`配置说明

### 3、模块配置

#### 1）部署脚本`build.sh`

```
Example:
  ./build.sh -u xiongweilie -m admin:online,service:online,tool:online
Usage:
  -u 必填，用户名                       示例：default
  -m 选填，要更新代码的业务模块         示例：admin:online,service:online,tool:online
  -e 选填，更新业务模块对应的环境变量   示例：admin:true,service:false,tool:false
  -d 选填，删除用户                     示例：default
```

#### 2）PHP模块

`app/Dockerfile`文件说明：

`app/sample`目录说明：

##### a. 仓储配置

- `.gitaddress`：声明当前模块的远程仓储地址

##### b. 钩子

- `on_add.sh`：创建角色时下载PHP模块代码完成之后的回调钩子，用已更新环境变量等文件
- `on_upd.sh`：某个模块更新完成之后的回调钩子，用以编译、重启服务等操作
- `on_env.sh`：更新环境变量的钩子

##### c. 示例目录`app/sample/sample`：

#### 3）Nginx配置

##### a. `nginx/conf.d`目录

##### b. `nginx/log`目录

#### 4）Jenkins配置方案

##### a. 安装插件获取当前用户

##### b. Docker镜像中的Jenkins与宿主机通信

##### c. 添加job

```
echo "正在将 admin-fe:${admin_fe},admin:${admin},service:${service},tool:${tool}  部署到 ${BUILD_USER_ID} 环境"

ssh apple@{jenkins内网IP} "sh ~/docker-compose/build.sh -u ${BUILD_USER_ID} -m admin-fe:${admin_fe},admin:${admin},service:${service},tool:${tool} -e admin:${admin_env},service:${service_env},tool:${tool_env}";
```

#### 5）Gitlab-ci/runner持续集成方案

##### a. 提交`.gitlab-ci.yml`文件

##### b. 注册gitlab runner

- [Kubernetes会不会被自身的复杂性压垮？](http://www.techug.com/post/will-kubernetes-collapse-under-the-weight-of-its-complexity.html)

当然了，如果需要在PHP的服务基础上集成其他语言的服务，比如Nodejs，涉及到的改动有：
1. 添加Nodejs镜像：`docker-compose.yml`
2. 添加部署任务：`build.sh`
	- 创建及删除用户流程
	- 部署流程
3. nginx配置文件示例：`nginx/conf.d/sample`

## 五、贡献

欢迎提供其他更专业的思路，欢迎提issue、fork；也可以邮件联系：xiongwilee[at]foxmail.com。
