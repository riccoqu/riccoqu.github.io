---
title: Linux日志文件与Syslog函数介绍
date: 2016-03-24
tags:
 - Linux
---


在我们程序运行的过程中,由于需要不间断的运行。当发生错误的时候就得产生错误的信息并且反馈给管理人员知道。特别的对于运行于后台的daemon程序,产生的日志消息更是能够帮助我们更加清楚全面的掌握我们程序的运行状况。
除此之外,Linux系统中各种大大小小的消息和错误都会使用日志,所以通过日志分析也能够让我们知道系统出了什么问题,以及是否有不当的操作等等,这也是众多系统管理员的一大利器!

今天我们就谈谈在Linux平台上如何使用日志。
<!--more-->

------
# **syslog服务**
在具体的使用前我们需要先了解日志服务的一个deamon——rsyslogd。
rsyslogd相比较于syslogd提供了更多的服务,在我使用的Ubuntu15.10下,也是自带了rsyslogd。
rsyslog为我们提供日志的服务。它像众多的deamon程序一样默默的提供着服务。  
在Bash下输入
```
ps aux|grep rsyslogd
```
得到如下
![rsyslog](http://7xrxgj.com1.z0.glb.clouddn.com/Linux-syslog01.png)
可以看出rsyslog是一个由父进程为init的进程的守护进程。  
它也就是为我们提供日志服务的进程了!  
那么,这个进程为我们产生的日志消息又在哪呢?  
`/var/log/`目录下,有各种各样的消息日志在等着我们查看呢,快去先看一眼吧!  
## **日志文件内容的一般形式**
如果我们打开`/var/log/`这个目录一看,就会发现有很多的文件,除无后缀的文件外还包括以`.log`结尾和以`.数字`结尾的文件(这与后面会说到的"日志文件轮替"相关)。
让我们先随便打开一个文件
![日志格式示例](http://7xrxgj.com1.z0.glb.clouddn.com/Linux-syslog02.png)
这是从我系统中`/var/log/syslog`文件截取的一段,可以看到其中包含的日志信息格式:
`[日期][主机][模块][信息]`
所传达的意思就是:  
>3月22日的18:03:44　由Innocence这台主机的dnsmasq[PID为1441]传来的消息,消息内容为"using nameserver 115.159.55.17#53"

怎么样,这样的消息格式是不是很简单明了?
## **服务类型**
rsyslogd为我们提供了很多种的服务类型,如果你在Bash中`man syslog`一下,就可以在facility参数里看到全部的服务啦。
这里列举**一部分**后面会用到的:

- LOG_AUTH——认证系统:login、su等
- LOG_AUTHPRIV——同上,单值登陆到对应用户的可读文件中
- LOG_MAIL——电子邮件系统
- LOG_DAEMON——其他系统守护进程
- LOG_NEWS——网络新闻系统
- LOG_LOCAL0 through LOG_LOCAL7

这些都是rsyslogd支持的服务名称,那么这些是什么意思呢?
举个例子来说:  
在一些与邮件相关的软件中设计日志文件记录时，都会主动调用rsyslogd内的mail服务名称(LOG_MAIL),所以不管多少种不同的软件程序,由它们的日志消息在rsyslogd看来都是mail这一类型的服务了。所以我们只需要在软件中将消息统统抛给rsyslogd,再通过修改配置文件就能够让它帮我们组织管理日志文件了。   
——后面会有更详细的解释

## **信息等级**
既然我们已经有了记录消息的地方,那么不同的消息也要分个轻重缓急吧?  
如果是严重到会导致系统崩溃的错误消息,那么我们也当然希望能够越早知道越好的喽!  
同样,还是在`man syslog`中,我们可以在`level`参数中看到消息的等级了!

- LOG_EMERG
- LOG_ALERT
- LOG_CRIT
- LOG_ERR
- LOG_WARNING
- LOG_NOTICE
- LOG_INFO
- LOG_DEBUG

严重程度从上往下递减。这里的信息等级与上面提到过的服务类型都与我们接下来要说的rsyslogd的配置文件有关

## **rsyslogd的配置文件**
虽然是deamon进程,但是我们也可以通过配置文件来进行控制达到自己的目的  
打开`/etc/rsyslog.conf`这个文件,让我们一步一步来解读...  
在这个文件头部有一行注释
```
#Default logging rules can be found in /etc/rsyslog.d/50-default.conf
```
说明了默认的日志规则在`/etc/rstslog.d/50-default.conf`下,那让我们马上去打开那个文件!  
以下为截取的一部分该文件的内容
```
auth,authpriv. *       /var/log/auth.log
mail.err               /var/log/mail.err
news.crit              /var/log/news/news.crit
news.err               /var/log/news/news/err
daemon.*;mail.*;\
	news.err;\
	*.=debuf;*.=info;\
	*.=notice;*.=warn  |/dev/xconsole
```
是不是已经看出一点配置文件的味道了?

## **syslog.conf的书写规则**
现在我们已经知道了日志文件,rsyslogd服务的服务类型与消息级别。就到我们可以动手的时候了!
在上面的文件中我截取了部分的文件内容,如下
```
mail.err                /var/log/mail.err
news.crit              -/var/log/news/news.crit
news.err               /var/log/news/news/err
```
在这个文件中`.`前面的字母表示上面提到的服务名。`.`后面表示消息的等级  
例如:当我们写`mail.err   /var/log/mail.err`的时候意思就是把所有`mail`服务类型的并且消息等级大于`err`的消息记录在`/var/log/mail.err`这个文件中!,其实也很简单是吧?  
**注意!**第二行中文件路径名前有一个`-`是什么意思呢?可以认为该项信息将存储在内存中,直到产生的信息足够大时才放回进磁盘内,这样处理对于频繁产生的信息可以增加访问的性能!

- `.`  代表比后面还要高的消息等级都会记录下来
- `.=` 代表只有后面的这个消息等级会被记录下来
- `.!` 代表除了后面的这个消息等级,其他的都会被记录下来

那么如果我们想记录除了mail和news的所有服务类型的所有级别的信息应该怎么写呢?
```
*.*;news,mail.none /var/log/message
或者
*.*;news.none;mail.none /var/log/message
```
上面两个都是可以的!  
从上面我们可以看出

- 支持通配符`*`
- 能够将服务名以`,`连接,并在最后再加消息级别
- 能够通过`;`分割开不同的服务和消息级别

那么远程主机呢?
只要将消息的去向改为`@主机名`或`@IP地址`就可以啦!  
rsyslogd默认使用514端口。
___
# **syslog API**
## **openlog、syslog、closelog函数**
Linux下通过包含`<syslog.h>`头文件就可以使用rsyslogd服务了!
```
#include<syslog.h>
void openlog(const char *ident,int option,int facility);
void syslog(int priority,const char *format,...);
void closelog(void);
```
**注:**`openlog`的调用是可以选择的,如果不调用`openlog`则会在第一次调用`syslog`时自动调用`openlog`。调用`closelog`也是可以选择的,它只是关闭被用于与rsyslogd进程通信的描述符。`ident`参数是加在消息前的一段字符串,通常将其设置为程序名称。

**option**用于openlog()的option参数可以是以下几个的组合  
`LOG_CONS`如果送到system   logger时发生问题，直接写入系统console。   
`LOG_NDELAY`立即开启连接(通常，连接是在第一次写入讯息时才打开的)。      
`LOG_PERROR`将讯息也同时送到stderr      
`LOG_PID`将PID含入所有讯息中      
**facility** 该参数用来指定何种程式在记录讯息，这可让设定档来设定何种讯息如何处理  
`LOG_AUTH`认证系统：login、su、getty等  
`LOG_AUTHPRIV`同LOG_AUTH，但只登录到所选择的单个用户可读的文件中  
`LOG_CRON`cron守护进程  
`LOG_DAEMON`其他系统守护进程，如routed  
`LOG_FTP`文件传输协议：ftpd、tftpd  
`LOG_KERN`内核产生的消息  
`LOG_LOCAL0~LOG_LOCAL7`为本地使用保留  
`LOG_LPR`系统打印机缓冲池：lpr、lpd  
`LOG_MAIL`电子邮件系统  
`LOG_NEWS`网络新闻系统  
`LOG_SYSLOG`由syslogd（8）产生的内部消息  
`LOG_USER`随机用户进程产生的消息  
`LOG_UUCP`UUCP子系统  
**level**决定讯息的重要性(以下的等级重要性逐次递减)  
`LOG_EMERG`紧急情况  
`LOG_ALERT`应该被立即改正的问题，如系统数据库破坏  
`LOG_CRIT`重要情况，如硬盘错误  
`LOG_ERR`错误  
`LOG_WARNING`警告信息  
`LOG_NOTICE`不是错误情况，但是可能需要处理  
`LOG_INFO`情报信息  
`LOG_DEBUG`包含情报的信息，通常旨在调试一个程序时使用  
更多的信息请参考man手册`man syslog`
## **一个小小的示例**
现在让我们打开`/etc/rsyslog.conf`  
并在最后写下这么一段...
![](http://7xrxgj.com1.z0.glb.clouddn.com/Linux-syslog04.png)
然后重启rsyslog服务让配置文件生效...
![](http://7xrxgj.com1.z0.glb.clouddn.com/Linux-syslog05.png)
写一个小小的C程序来调用syslogAPI...
![](http://7xrxgj.com1.z0.glb.clouddn.com/Linux-syslog06.png)
运行后发现目录下多了个mytest文件!里面正是我们发送的消息!
![](http://7xrxgj.com1.z0.glb.clouddn.com/Linux-syslog07.png)
怎么样,是不是简单又好用!
___
# **日志文件的轮替--logrotate**
什么是日志文件的轮替?  
当越来越多的日志消息塞满我们的文件的时候,多么希望它能够按照我们需要的方式以一天为单位保存消息啊!没错,logrotate可以帮我们做到。  
我们可以通过对logrotate配置规则使得日志文件按我们的方式组织和更新,这样可就方便多了。  
rsyslog进程是在系统启动时启动的daemon,但是logrotate却是在cron下在规定的时间到达后才被启动进行日志文件的轮替工作。所以我们可以在`/etc/cron.daily`文件夹中找到`logrotate`就是记录了每天要进行的日志文件轮替的行为。

## **logrotate的配置文件**
是啊,什么都少不了配置文件。通过配置文件才能使得程序完成我们的需求。  
logrotate的配置文件位于`/etc/logrotate.conf`,同样让我们通过一小段配置文件来理解  
![](http://7xrxgj.com1.z0.glb.clouddn.com/Linux-syslog03.png)
可以看到,我们在rsyslogd的配置文件中书写的是不同消息的存放文件规则,那么在logrotate的配置文件中我们则写的是日志文件的替换规则。这规则就是:  
```
文件名{
		参数1
		参数2
		·
		·
		·
}
```
我们可以通过`man logrotate`来获取所有的参数和详细描述。这里列出一部分:

- `daily` 指定转储周期为每天
* `weekly` 指定转储周期为每周
* `monthly` 指定转储周期为每月
* `compress` 通过gzip 压缩转储以后的日志
* `nocompress` 不需要压缩时，用这个参数
* `copytruncate` 用于还在打开中的日志文件，把当前日志备份并截断
* `nocopytruncate` 备份日志文件但是不截断
* `create mode(文件权限) owner(拥有者) group(组)` 转储文件，使用指定的文件模式创建新的日志文件
* `nocreate` 不建立新的日志文件
* `delaycompress` 和 compress 一起使用时，转储的日志文件到下一次转储时才压缩
* `nodelaycompress` 覆盖 delaycompress 选项，转储同时压缩。
* `errors address` 转储时的错误信息发送到指定的Email 地址
* `ifempty` 即使是空文件也转储，(logrotate 的缺省选项)
* `notifempty` 如果是空文件的话，不转储
* `mail address` 把转储的日志文件发送到指定的E-mail 地址
* `nomail` 转储时不发送日志文件
* `olddir directory` 转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统
* `noolddir` 转储后的日志文件和当前日志文件放在同一个目录下
* `prerotate/endscript` 在转储以前需要执行的命令可以放入这个对，这两个关键字必须单独成行
* `postrotate/endscript` 在转储以后需要执行的命令可以放入这个对，这两个关键字必须单独成行
* `rotate count` 指定日志文件删除之前转储的次数，0 指没有备份，5 指保留5 个备份
* `tabootext [+] LIST` 让logrotate 不转储指定扩展名的文件，缺省的扩展名是：.rpm-orig, .rpmsave, v, 和 ~
* `size SIZE` 当日志文件到达指定的大小时才转储，Size 可以指定 bytes (缺省)以及KB (sizek)或者MB (sizem)

那么我们现在再看`/var/log/`下的文件。比如syslog日志文件。当logrotate被启动工作时,如果满足了配置文件的规则。就会将`syslog`改名为`syslog.1`并新创建一个`syslog`文件用于最新消息的记录,当文件再次被轮替时,数字递增并且创建新文件,于是就有了`syslog`、`syslog.1`、`syslog.3`的三个文件。后两个作为记录保存,如果规则中需要压缩的话文件也会被压缩保存的~    
这样对于那些`.gz``.数字.gz`结尾的文件是什么意思是不也就理解了?
