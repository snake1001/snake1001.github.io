---
title: "[unblock@badfail.info].ETH 勒索病毒分析"
date: 2024-08-16
category: [REVERSE]
tags: [mmAnalysis]
img_path: /assets/images/
---

这篇文章会详细展示这款ETH病毒的逻辑架构以及一些实现细节，仅供研究使用，千万别把它当成"How to write a extortion virus"。因此引起的后果本人不负责任。



### 1、入口点以及主线程框架

![图1]({{ page.name | remove: '.md' }}/start.jpg)

简单流程：**RC4解出API符号表，重建API跳转表———判断是否具有Admin权限———拷贝自身到指定目录，修改启动项———解出后缀符号表，建立检索数据结构———开启线程循环关闭某些进程和服务———删除磁盘卷影备份———开启线程检索网络资源并加密———开启线程检索本地文件并加密。**

### 2、Windows API 导入表的封装

![图2]({{ page.name | remove: '.md' }}/make_IAT.jpg)

![图 3]({{ page.name | remove: '.md' }}/api.jpg)

程序并没有直接利用跳转表，而是进行了一层封装。



### 3、判断是否有管理员权限

![图4]({{ page.name | remove: '.md' }}/hasadmin.jpg)

并没有提权操作，只是简单判断操作系统版本，判断是否具有 admin 权限。

### 4、感染

![图5]({{ page.name | remove: '.md' }}/copyself.jpg)

将自身拷贝到 4 个地方，修改两处注册表项，以获得开机运行的机会。

### 5、关闭服务及删除卷影

![图6]({{ page.name | remove: '.md' }}/closeps.jpg)

关闭了一些进程和服务，释放文件占用，方便加密对用户来说关键的文件。

![图7]({{ page.name | remove: '.md' }}/deletejy.jpg)

防止加密后使用分区卷影恢复，调用cmd命令删除所有卷影。

### 6、搜索网络/本地文件进行加密

` sub_409AA0(groupSuffixes, gSync);		//加密网络文件
sub_409780(groupSuffixes, gSync);		//加密本地文件	`

两者实际上都调用了 `sub_4093B0`来处理密码生成、文件选取和加密。

![图8]({{ page.name | remove: '.md' }}/sub_4093b0.jpg)

针对每个逻辑磁盘都开启了两组加密线程，一组针对预定义的后缀文件加密，一组针对其他后缀文件加密。

每组加密线程由一个文件选择线程和4个加密线程组成，使用一个独立的随机密钥，加密算法是AES-256 cbc mode。

![图9]({{ page.name | remove: '.md' }}/threadgroup.jpg)

文件选择上对于系统启动相关的文件不加密。

![图10]({{ page.name | remove: '.md' }}/bootfiles.jpg)

根据文件大小的不同，加密方式也有所不同。小于等于 0 x180000 字节的文件字节全文加密。

![图 11]({{ page.name | remove: '.md' }}/encryptfiles.jpg)



对于大文件，实际上是选取开头、中间、结尾的3个0x40000字节块，组合成0xC0000的大块，经过AES-256加密，加上0x38的选块位置描述数据结构，放到了文件的末尾，然后再把那3个0x40000的字节块清零。

![图12]({{ page.name | remove: '.md' }}/largefileencrypt.jpg)

![图13]({{ page.name | remove: '.md' }}/largefileencrypt1.jpg)

### 7、秘钥的生成与加密

这里需要重点感谢po 叔大神的指点。秘钥的生成主要是获取一些processid、RDTSC等数，经过多轮 SHA1 的运算后产生。秘钥的加密过程经过代码混淆，提取出来的算法是：先将 256 位的秘钥和另外的非零随机数组成 1024 位的大数，然后 RSA 加密。

```python
x = text
for i in range(16):
  x = x**2 % key
result = x * text % key
```

加密后的 128 字节数据和初始向量 `iv`会写入加密后的文件末尾，供解密使用。



### 后记

病毒使用的 CRC32、RC4、SHA1、RSA 都是高效的 C 算法，对习惯了调用库函数的开发者，这些 raw C 算法代码还原出来还是很值得学习的。

这个病毒只要修改一下内置的加密大数N，就是一个新版本，作者显然也是这么想的，程序里对加密用到的大数N进行了SHA1，并将这个值写到了加密后文件的末尾，用来识别到底是哪一个版本。

病毒除了代码混淆的部分，还有一些空函数，联想到病毒没有提权以及特别的利用漏洞感染的能力，这些空出来的地方，实际上都可以利用来进行改造的。