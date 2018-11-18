---
title: Beego源码解析(一)——配置项初始化流程
date: 2016-07-30 00:06:44
tags:
  - Beego Framework
---

# 简介
>beego是一个快速开发 Go应用的 HTTP框架,他可以用来快速开发 API、Web以及后端服务等各种应用,是一个 RESTful框架。
  想了解更多关于 Beego的介绍,可以看 [**官方文档**](http://www.beego.me/docs/intro)  
  本文及以后的文章都采用 stable v1.6.1版本  
  Beego项目[**Github地址**](https://github.com/astaxie/beego)

# 动力
最近在看 Beego的源码,我选择 Beego一方面是因为我对 Go语言很感兴趣,另一方面在 GoWeb方面 Beego也做的十分出色。模块化的设计、完善的文档和社区、强大的功能都是我对于 Beego下手的推动力。  
对于一个解析项目的开始,我都想从配置来下手。所以这篇文章也是主要介绍了 Beego在启动过程中配置项的初始化过程。这也是关于 Beego的第一篇文章,日后我们慢慢补坑的。

<!--more-->

关于 **Beego源码的注释**可以见我的 **[Github](https://github.com/riccoqu/Beego-Comments)** 我会很努力的慢慢完善它的 :D

# beego配置项的解析
在开始之前先让我们用 bee工具创建一个 Beego的应用。  项目的结构大概是这样的
```
myproject
|—conf
	 |—app.conf
|—controllers
	 |—default.go
|—main.go
|—modules
|—routers
	 |—router.go
|—static
	 |—css
	 |—img
	 |—js
|—tests
	 |—default_test.go
|—views
	 |—index.tpl
```
### 配置文件
可以看到 那个`conf/app.conf`就是项目的默认配置文件了  
在配置文件中的配置项都是采用 **键值对** 的方式,即 `key = value`

我们看下 beego中对于关于配置的一些文件
beego目前支持 INI、XML、JSON、YAML格式的配置文件解析,默认是 INI格式的解析
`beego/config`这个包内放的就是不同解析器的文件  
`beego/config.go`这个文件就是用来初始化配置项的文件

### 程序中的配置项
在 beego的启动过程中,有两个变量对配置项的初始化很重要,它们都在 beego/config.go:106被声明
```Go
var	(
	BConfig *Config
	AppConfig *beegoAppConfig

	...配置文件的路径等变量
   )
```
BConfig是程序在整个运行过程中都需要用的,而 AppConfig是用来解析配置项的接口,所以整个配置项的解析过程来说就是 **使用 AppConfig去完成 BConfig的初始化**  
(关于 Config这个结构体的定义可以参考 beego/config.go:50行开始的 `Listen`、`WebConfig`、`SessionConfig`、`LogConfig`这四个结构体,注意这里的 Config保存的是程序运行是已经解析好的配置,而与下文提到的 beego/config/Config接口无关)  
### 关于 beegoAppConfig结构体
让我们继续进入 beegoAppConfig结构体,在 beego/config.go:340可以看到定义
```Go
type beegoAppConfig struct{
		innerConfig config.Configer
}
```
明白了吧,其实它就是 config包中 Configer接口的封装.所以在 beego/config.go文件后面部分都是在调用这个接口从而实现了 beegoAppConfig的很多方法  
关于 config.Configer接口我们可以在 config/config.go:50找到,它定义了16种在解析配置项中可能用到的方法
不过正在看源码的同学可能会发现在 Configer接口下还有个 Config接口,那么它又是干什么的呢?
```Go
type Config interface{
		Parse(key string)(Configer,error)
		ParseData(data []byte)(Configer,error)
}
```
看下它的接口定义我们就能猜到,它是为了给我们创建一个 Configer实例的.而 Parse()和 ParseData()两个函数在不同的配置器实现里都有不同的实现,但他们的目的也就只有创建实例了。  

### 保存配置器的 map
看到这里需要打断下,因为又有一个非常重要的变量需要我们认识  
在 beego/config/config.go:85行有这样一行
```Go
var adapters = make(map[strng]Config)
```
为什么说这个变量很重要呢,看看参数便知。map的 key是 string类型,也就是我们对应配置器的名称("ini"、"xml"等）
而第二个就是我们刚才定义了 Parse()和ParseData()方法的 Config接口,在同文件的91行可以看到一个叫 Register的函数。在不同的配置器文件的init()函数中,配置器都会通过 Register函数向 adapters变量进行注册(比如打开 beego/config/json.go拖到文件末尾就可以看到)  
也就是说在 beego/config/包中所有源文件的 init()函数执行完时, adapters就已经保存了所有配置器了

这样在 beego/config.go文件中使用时只需要传入配置器的名字,就能获得对应的 Config接口方法,调用方法后就能获得实现了 Configer接口所有方法的实例了!  
那么接下来就比较简单了,不同的配置器只需要在自己文件中实现 Configer接口的所有方法就能成功的解析文件了,这里只需要根据不同格式进行配置项的处理即可  

### BConfig的初始化
通过这样自顶向下的分析,大家应该能够对这几个接口和函数之间的关系有了大概的思路了吧,既然我们现在已经知道如何得到 beegoAppConfig这样实现了16种解析方法的实例,那么接下来也就是通过这个实例给我们需要的BConfig变量赋值了。
首先在 beego/config.go文件的 init()函数中我们可以看到了一大片关于给 BConfig赋默认值的代码,并且在 init()函数的最后 使用 appConfigPath(保存配置文件路径的string类型变量 beego/config.go:117)检查配置文件是否存在,然后调用 parseConfig()函数开始配置的解析过程  
在 parseConfig()函数中首先是调用了本包内的 newAppConfig()函数给 AppConfig初始化, 然后就可以看到使用了 AppConfig中实现的Congiger接口的方法给BConfig赋了很多值并且在函数末尾初始化了log  
因为给 BConfig初始化的代码在 init()函数中,所以在 beego.App.Run()方法中并不能看到初始化配置项的代码。也就是说在 Run()函数运行前,配置就已经赋值完成了 :D

___
配置项的初始化大概就是这么个流程  
如果有错误非常希望您能指出让我改正 :D
