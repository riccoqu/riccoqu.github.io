---
title: Beego源码解析(二)-路由机制
date: 2016-08-01 00:22:08
tags:
  - Beego Framework
---

>上一篇文章[Beego源码解析(一)-配置项初始化流程](https://riccoqu.github.io/2016/07/30/Beego%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%B8%80-%E2%80%94%E9%85%8D%E7%BD%AE%E9%A1%B9%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B5%81%E7%A8%8B/)介绍了 Beego关于配置项初始化的流程。那么今天就来说说在 Beego中非常重要的**路由机制**.  
Beego到现在 v1.6.1版本为止支持了:**固定路由**、**正则路由**、**自动路由**这三种路由方法.  
关于这三种路由的详细用法可以参考官方给出的[**开发文档**](),这里面已经记录的很全面了.  

所以我们今天这篇文章就是要介绍这三种路由是如何在 Beego内部实现的.  

<!--more-->

关于 Beego的源码注释可以见我的[**Github**](https://github.com/riccoqu/Beego-Comments)

## 一个简单的示例
让我们先从官网给出的示例开始,下面是会在浏览器中打印"HelloWorld"的一个Beego程序.
```Go
package main

import(
	"github.com/astaxie/beego"
)

type MainController struct {
	beego.Controller
}

func (this *MainController) Get() {
	this.Ctx.WriteString("Hello World")
}

func main() {
		beego.Router("/",&MainController{})
		beego.Run()
}
```

我们需要先知道它干了什么:
1. 自定义了一个内含 beego.Controller(这个类型后面会讲到)控制器的 MainController
2. 重写了 MainController的 Get()方法,熟悉 Go语言的应该知道这个方法来自 Controller
3. 在 main()函数中调用了 beego.Router()方法注册了路由"/"和一个 MainController实例
4. 执行了 beego.Run()方法启用了 beego程序

## 重要的类型和接口
为了不在接下来的流程中打断,在介绍流程之前需要先了解 beego中关于路由的一些东西

### ControllerInterface 接口

源文件中的位置: beego/controller.go:90
```Go
type ControllerInterface interface {
		Init(ct *context.Context,controllerName,actionName string,app interface{})
		Prepare()
		Get()
		Post()
		Delete()
		Put()
		Head()
		Patch()
		Options()
		Finish()
		Render() error
		XSRFToken() string
		CheckXSRFCookie() bool
		HandlerFunc(fn string) bool
		URLMapping()
}
```

这个接口定义了 15个方法,看名字就能够知道这是每个 Controller都需要实现的接口

### Controller结构体

位置 beego/controller.go:60
```Go
type Controller struct {

//context data
Ctx  *context.Context
Data map[interface{}]interface{}

//route controller info
controllerName string
actionName     string
methodMapping  map[string]func() //method:routertree
gotofunc       string
AppController  interface{}

// template data
TplName        string
Layout         string
LayoutSections map[string]string // the key is the section name and the value is the template name
TplExt         string
EnableRender   bool

// xsrf data
_xsrfToken string
XSRFExpire int
EnableXSRF bool

// session
CruSession session.Store
}
```

这个结构体保存了作为 Controller的一些必要的信息,一些基础的字段看名字就比较好理解  
在这里的 context.Context(Beego中的上下文,封装了 HTTP的输入和输出)和 Session.Store(用于存储 Session)在以后的文章中会再提到  

在这个源文件中的后面部分都是对 Controller的一些方法实现,我们会注意到 Controller实现了 ControllerInterface的方法,但是在一些方法实现中却是用 Ctx成员向客户端进行错误输出(例如 Get()方法)。  
因为就像例子中给的一样,当我们需要自己定义 Controller,并且使用 Get()函数来完成对客户端 Get请求的处理时,我们就需要自己实现处理逻辑,这样就**覆盖**了本身输出错误的方法.而对于我们没有实现的方法(比如例子中的 Post()方法)没有重写,则对于客户端的 Post请求就会输出错误了

### ControllerRegister结构体
这是一个非常关键的数据结构,为什么说他关键呢?我们可以先看下 Beego中 App结构体的定义
位置: beego/app.go
```Go
type App struct {
	Handlers *ControllerRegister
	Server *http.Server
}
```

关于 App需要说下,在程序中 App类型的变量**BeeApp**(beego/app.go:32)在 init()函数中会调用 NewApp()创建出唯一的一个Beego程序实例  
可以看到在例子中 main()函数最后调用了 beego.Run()函数,这个函数会在设置完hooks(关于回调方法以后也会介绍)后进入 BeeApp.Run()函数并且在进入这个函数后就会根据配置项开始不同的　HTTP请求的处理(在 ControllerRegister实现的 ServeHTTP()方法中)
App中一共就两个变量,一个**Server**(标准包中 http.Server类型,这个不做介绍,需要的可以看 Go语言文档).  
另外一个就是 **ControllerRegister**,这个 ControllerRegister顾名思义就是注册 Controller的管理器,那么如何管理的呢?接下来看定义

位置: beego/router.go:115
```Go
type ControllerRegister struct {
	routers map[string][]*Tree
	enableFilter bool
	filters map[int][]*FilterRouter
	pool sync.Pool
}
```

可以看到这篇文章的主角已经出现了, routers就是我们程序运行时所需要的路由表, routers的 key是我们注册的方法名(例如"get"、"post"等),而 value就是由注册的路由构建出来的路由树了(关于路由树,后面也会讲到).

### ControllerInfo结构体
这个结构体是用来保存我们自定义的控制器信息的,看下定义便知道  
位置: beego/router.go:104
```Go
type ControllerInfo struct {
	pattern string	//模式
	controllerType reflect.Type//类型
	methods map[string]string//支持的方法
	handler http.Handler//http.Handler接口
	runFunction FilterFunc
	routerType int//路由类型
}
```

### Tree结构体
```Go
type Tree struct {
	//路由前缀
	prefix string
	//不带正则的路由
	fixrouters []*Tree
	//通配符,如果设置并且查找 fixrouters失败时会来查找 wildcard
	wildcard *Tree
	//叶子节点,如果设置并且查找 wildcard失败后会查找 leaves,里面保存了一些正则的信息
	leaves []*leafInfo
}
```

### leafInfo结构体
```Go
type leafInfo struct {
	wildcards []string//通配符
	regexps *regexp.Regexp//正则对象
	runObject interface{}//一般保存得到的 ControllerInfo对象,在处理请求时会返回该对象,并调用处理方法
}
```
这两个结构体就会构成一颗用来查找路由的路由树,通过结构体的定义我们应该不难看出关于正则的部分都会存放在叶子节点中.所以这个路由树的大致样子也比较好理解

## 路由的注册过程
在前面的实例中可以看到需要注册自己的 Controller时使用的是 beego.Router()函数(在官方开发文档中的基础路由部分也可以使用 beego.Get()方法注册路由,不过内部与用 beego.Router()注册方法相比都会使用 addToRouter()函数,所以也是比较相似的)  

看下 beego.Router的原型:
```Go
beego/app.go:211
func Router(rootpath string,c ControllerInterface,mappingMethods ...string *App) {
		BeeApp.Handlers.Add(rootpath,c,mappingMethods...)
		return BeeApp
}
```

看到第一个参数是需要注册路由,而第二个参数是我们自定义实现了 ControllerInterface接口的控制器,第三个就是自定义路由中方法和处理函数的映射关系  
函数内部实际调用了 App.ControllerRegister的Add()方法来注册  
接下来看看 Add()方法做了什么:  
位置: beego/router.go:144
```Go
func (p *ControllerRegister) Add(pattern string, c ControllerInterface, mappingMethods ...string) {
	reflectVal := reflect.ValueOf(c)	//反射获得 value
	t := reflect.Indirect(reflectVal).Type()//反射获得 type
	methods := make(map[string]string)
	if len(mappingMethods) > 0 {
		semi := strings.Split(mappingMethods[0], ";")//切分出每个以';'分隔的自定义方法和对应的函数
		for _, v := range semi {
			colon := strings.Split(v, ":")//切分出以':'分隔的方法名和对应的函数,colon[1]为处理的函数名
			if len(colon) != 2 {
				panic("method mapping format is invalid")
			}
			comma := strings.Split(colon[0], ",")//切分出以','分隔的方法名, comma包含了当前需要注册的所有方法名
			for _, m := range comma {
				if _, ok := HTTPMETHOD[strings.ToUpper(m)]; m == "*" || ok {
					//如果方法名为通配符'*'或者在支持的方法列表中.并使用反射包中的方法获得一个绑定对应函数的　Value类型
					//如果返回的值有效,就将当前方法加入到 methods中
					if val := reflectVal.MethodByName(colon[1]); val.IsValid() {
						methods[strings.ToUpper(m)] = colon[1]
					} else {
					//不支持方法时报错
						panic("'" + colon[1] + "' method doesn't exist in the controller " + t.Name())
					}
				} else {
						panic(v + " is an invalid method mapping. Method doesn't exist " + m)
				}
			}
		}
	}
	//添加 ControllerInfo类型来保存此项路由规则
	route := &controllerInfo{}
	route.pattern = pattern
	route.methods = methods
	route.routerType = routerTypeBeego
	route.controllerType = t
	//当传入的方法名为空时,给当前模式加入所有支持的方法
	if len(methods) == 0 {
		for _, m := range HTTPMETHOD {
			p.addToRouter(m, pattern, route)
		}
	} else {
		//方法名不为空时,判断是否含有通配符 "*"
		for k := range methods {
			if k == "*" {
				for _, m := range HTTPMETHOD {
					//含有通配符,加入所有方法
					p.addToRouter(m, pattern, route)
					}
			} else {
					//只加入指定的方法
					p.addToRouter(k, pattern, route)
			}
		}
	}
}
```
这是一个稍微长点的函数,不过通过注释可以看出这个函数做了几个工作:
1. 解析了传入的 mappingMethods,得到其中包含的全部方法
2. 用传入的4个参数构造出一个 ControllerInfo的实例,而这个实例中就保存了我们自定的控制器的 reflct.Type类型(可参考[ControllerInfo](#ControllerInfo结构体))  

在函数的最后调用了 ControllerRegister的 addToRouter()方法

位置: beego/router.go:199
```Go
func (p *ControllerRegister) addToRouter(method, pattern string, r *controllerInfo) {
	if !BConfig.RouterCaseSensitive {
		pattern = strings.ToLower(pattern)
	}
	if t, ok := p.routers[method]; ok {
		//如果方法对应的路由树存在就直接添加
		t.AddRouter(pattern, r)
	} else {
		//方法不存在这新创建一个路由树
		t := NewTree()
		t.AddRouter(pattern, r)
		//设定新方法的路由树
		p.routers[method] = t
	}
}
```
这个方法比较短,主要是判断当前的方法是否在 ControllerRegister的成员 routers所支持的方法中  
* 存在就直接插入对应的路由树  
* 否则创建一个新的路由树  

路由树节点的插入操作就是 Tree.AddRouter()方法

位置: beego/tree.go:206
```Go
func (t *Tree) AddRouter(pattern string,runObject interface{}) {
	t.addseg(splitPath(pattern),runObject,nil,"")
}
```
可以看出它只是把 pattern中的路径进行了切割(例如"/admin/users"切割成"["admin","users"]"),并返回一个 string类型的数组切片  
那么接下来的目的就很明确了,我们需要使用 Tree提供的 addseg方法给路由树**添加节点**

这个函数也是最终的一个函数了,函数的逻辑可以看注释

```Go
func (t *Tree) addseg(segments []string, route interface{}, wildcards []string, reg string) {
	if len(segments) == 0 {
		if reg != "" {
			//添加 leaves节点,并给 leaves添加正则规则
			t.leaves = append(t.leaves, &leafInfo{runObject: route, wildcards: wildcards, regexps: regexp.MustCompile("^" + reg + "$")})
		} else {
			t.leaves = append(t.leaves, &leafInfo{runObject: route, wildcards: wildcards})
		}
	} else {
		seg := segments[0]
		iswild, params, regexpStr := splitSegment(seg)//splitSegment函数在后面介绍
		// if it's ? meaning can igone this, so add one more rule for it
		if len(params) > 0 && params[0] == ":" {
			//当　params[0]为':'时,代表参数为空,开始解析下一个
			t.addseg(segments[1:], route, wildcards, reg)//递归调用
			params = params[1:]
		}
		//Rule: /login/*/access match /login/2009/11/access
		//if already has *, and when loop the access, should as a regexpStr
		//全匹配方式,可参考　http://beego.me/docs/mvc/controller/router.md 的正则路由->全匹配方式
		// utils.InSlice()检查":solat"是否在wildcards中
		if !iswild && utils.InSlice(":splat", wildcards) {
			//如果使用了全匹配方式则继续使用正则解析
			iswild = true
			regexpStr = seg
		}
		//Rule: /user/:id/*
		if seg == "*" && len(wildcards) > 0 && reg == "" {
			regexpStr = "(.+)"
		}
		//包含有正则表达式　
		if iswild {
			if t.wildcard == nil {
				t.wildcard = NewTree()
			}
			if regexpStr != "" {
				if reg == "" {
					rr := ""
					for _, w := range wildcards {
						if w == ":splat" {
							rr = rr + "(.+)/"
						} else {
							rr = rr + "([^/]+)/"
						}
					}
					regexpStr = rr + regexpStr
				} else {
					regexpStr = "/" + regexpStr
				}
			} else if reg != "" {
				if seg == "*.*" {
					regexpStr = "/([^.]+).(.+)"
					params = params[1:]
				} else {
					for range params {
						regexpStr = "/([^/]+)" + regexpStr
					}
				}
			} else {
				if seg == "*.*" {
					params = params[1:]
				}
			}
			t.wildcard.addseg(segments[1:], route, append(wildcards, params...), reg+regexpStr)//递归调用
		} else {
			var subTree *Tree
			for _, sub := range t.fixrouters {
				if sub.prefix == seg {
					subTree = sub
					break
				}
			}
			if subTree == nil {
				subTree = NewTree()
				subTree.prefix = seg
				t.fixrouters = append(t.fixrouters, subTree)
			}
			subTree.addseg(segments[1:], route, wildcards, reg)//递归调用
		}
	}
}
```
至此路由树节点添加完成  

这里需要提一下 splitSegment这个函数
位置: beego/tree.go:489
```Go
// "admin" -> false, nil, ""
// ":id" -> true, [:id], ""
// "?:id" -> true, [: :id], ""        : meaning can empty
// ":id:int" -> true, [:id], ([0-9]+)
// ":name:string" -> true, [:name], ([\w]+)
// ":id([0-9]+)" -> true, [:id], ([0-9]+)
// ":id([0-9]+)_:name" -> true, [:id :name], ([0-9]+)_(.+)
// "cms_:id_:page.html" -> true, [:id_ :page], cms_(.+)(.+).html
// "cms_:id(.+)_:page.html" -> true, [:id :page], cms_(.+)_(.+).html
// "*" -> true, [:splat], ""
// "*.*" -> true,[. :path :ext], ""      . meaning separator
//正则路由,用于对正则的 Segment进行解析
//当 key中包含正则 返回true，否则返回false
//返回值第二个为不同的参数
//第三个为正则的规则
func splitSegment(key string) (bool, []string, string)
```

## Final
最终我们从调用beego.Router()到最后给 ControllerRegister.router成功添加路由树节点的过程就完成了  
总结一下就是注册路由的过程就是在添加 ControllerRegister中的路由树的节点,而在 HTTP执行的过程中对这棵树进行搜索(这就到树的搜索方法了),从而判断接受到的请求应该怎么样的处理(对应的根据 Controller不同的类型调用不同的方法)  
完成 HTTP请求的正常处理过程:D


如果文章有误,非常希望能给我提出,好让我更正 :D
