---
title: Beego源码解析(三)-HTTP请求处理流程
date: 2016-08-02 14:34:33
tags:
  - Beego Framework
---

>关于上一篇文章[Beego源码解析(二)-路由机制](https://riccoqu.github.io/2016/08/01/Beego%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%BA%8C-%E8%B7%AF%E7%94%B1%E6%9C%BA%E5%88%B6/)中介绍了在 Beego中如注册路由以及如何将我们自定义的路由加入到 Beego的 App实例中  
我们知道关于注册的路由都是会加入到路由树的节点中,那么在 HTTP的请求中,查找路由树就是非常关键的一部分了  

这篇文章将梳理一遍在 Beego启动后,处理 HTTP请求的流程

<!--more-->

关于 Beego的源码注释可以见我的[**Github**](https://github.com/riccoqu/Beego-Comments)

## beego.Run()入口
在第一篇文章中就曾提到过,在启动 Beego应用时都是通过调用 beego.Run()方法启动.而这个方法在设置了6个回到调函数后会进入 app.Run()方法(app就是我们 Beego应用程序的一个实例,在 app.go的 init()函数中被初始化)  

### app.Run()函数
app.Run()函数就是在一切都准备好之后进入 HTTP请求循环的一个启动函数了,函数并不是很长而且逻辑比较简单,结合注释看下即可
beego/app.go:
```Go
// beego程序启动函数
func (app *App) Run() {
	//获得地址
	addr := BConfig.Listen.HTTPAddr
	if BConfig.Listen.HTTPPort != 0 {
		addr = fmt.Sprintf("%s:%d", BConfig.Listen.HTTPAddr, BConfig.Listen.HTTPPort)
	}

	var (
		err        error
		l          net.Listener
		endRunning = make(chan bool, 1)
	)
	// 运行 CGI服务器
	if BConfig.Listen.EnableFcgi {}
	//这里设置标准包中的 http.Server.handler为 app.ServerHandler
	//也就是 ContollerRegister类型,因为 ControllerRegister实现了 ServeHTTP()函数
	app.Server.Handler = app.Handlers
	//设置超时
	app.Server.ReadTimeout = time.Duration(BConfig.Listen.ServerTimeOut) * time.Second
	app.Server.WriteTimeout = time.Duration(BConfig.Listen.ServerTimeOut) * time.Second
	// 运行热编译模式
	if BConfig.Listen.Graceful {
		httpsAddr := BConfig.Listen.HTTPSAddr
		app.Server.Addr = httpsAddr
		// 热编译模式下的 HTTPS
		if BConfig.Listen.EnableHTTPS {
			go func() {
				time.Sleep(20 * time.Microsecond)
				if BConfig.Listen.HTTPSPort != 0 {
					httpsAddr = fmt.Sprintf("%s:%d", BConfig.Listen.HTTPSAddr, BConfig.Listen.HTTPSPort)
					app.Server.Addr = httpsAddr
				}
				server := grace.NewServer(httpsAddr, app.Handlers)
				server.Server.ReadTimeout = app.Server.ReadTimeout
				server.Server.WriteTimeout = app.Server.WriteTimeout
				//执行 HTTPS的 ListenAndServerTLS()
				if err := server.ListenAndServeTLS(BConfig.Listen.HTTPSCertFile, BConfig.Listen.HTTPSKeyFile); err != nil {
					BeeLogger.Critical("ListenAndServeTLS: ", err, fmt.Sprintf("%d", os.Getpid()))
					time.Sleep(100 * time.Microsecond)
					endRunning <- true
				}
			}()
		}
		// 热编译模式下的 HTTP
		if BConfig.Listen.EnableHTTP {
			go func() {
				server := grace.NewServer(addr, app.Handlers)
				server.Server.ReadTimeout = app.Server.ReadTimeout
				server.Server.WriteTimeout = app.Server.WriteTimeout
				if BConfig.Listen.ListenTCP4 {
					server.Network = "tcp4"
				}
				// 执行 HTTP的 ListenAndServe()
				if err := server.ListenAndServe(); err != nil {
					BeeLogger.Critical("ListenAndServe: ", err, fmt.Sprintf("%d", os.Getpid()))
					time.Sleep(100 * time.Microsecond)
					endRunning <- true
				}
			}()
		}
		<-endRunning
		return
	}
	// 运行普通模式
	app.Server.Addr = addr
	// HTTPS
	if BConfig.Listen.EnableHTTPS {
		go func() {
			time.Sleep(20 * time.Microsecond)
			if BConfig.Listen.HTTPSPort != 0 {
				app.Server.Addr = fmt.Sprintf("%s:%d", BConfig.Listen.HTTPSAddr, BConfig.Listen.HTTPSPort)
			}
			BeeLogger.Info("https server Running on %s", app.Server.Addr)
			// 运行 ListenAndServeTLS
			if err := app.Server.ListenAndServeTLS(BConfig.Listen.HTTPSCertFile, BConfig.Listen.HTTPSKeyFile); err != nil {
				BeeLogger.Critical("ListenAndServeTLS: ", err)
				time.Sleep(100 * time.Microsecond)
				endRunning <- true
			}
		}()
	}
	// HTTP
	if BConfig.Listen.EnableHTTP {
		go func() {
			app.Server.Addr = addr
			BeeLogger.Info("http server Running on %s", app.Server.Addr)
			if BConfig.Listen.ListenTCP4 {
				//TCP4
				ln, err := net.Listen("tcp4", app.Server.Addr)
				if err != nil {
					BeeLogger.Critical("ListenAndServe: ", err)
					time.Sleep(100 * time.Microsecond)
					endRunning <- true
					return
				}
				if err = app.Server.Serve(ln); err != nil {
					BeeLogger.Critical("ListenAndServe: ", err)
					time.Sleep(100 * time.Microsecond)
					endRunning <- true
					return
				}
			} else {
				// 运行 ListenAndServe()
				if err := app.Server.ListenAndServe(); err != nil {
					BeeLogger.Critical("ListenAndServe: ", err)
					time.Sleep(100 * time.Microsecond)
					endRunning <- true
				}
			}
		}()
	}
	<-endRunning
}
```
这里需要注意的因为主要讲 HTTP的请求处理,所以为了代码更短我将 CGI部分的代码删掉了  
想看的可以去看 app.Run()的源码 :D

这个函数的逻辑也比较简单,因为在运行到这个函数之前该初始化的地方已经都做好了
所以这个函数只需要根据不同的配置启动不同的服务就行,不同的是根据不同的服务需要调用  
* serve.Server()			对应TCP4
* server.ListenAndServe()	对应HTTP
* serve.ListenAndServeLTS()	对应HTTPS

另外我们也注意到在 热编译的模式中, app.server被 grace.NewServer()函数的返回值给**重写**了  
因为主要研究 HTTP的处理,所以这里对 grace包不做深究  

## ServeHTTP()
了解 http包的都应该知道,在 http.Server.ListenAndServe()被调用后,每当有 HTTP请求来到时就会开启一个新的 goroutine并且调用对应 Server的 Handler接口处理(Handler接口包含了 ServeHTTP()方法).而且我们看到在 app.Run()方法中有一句
```Go
app.Server.Handler = app.Handlers
```
而这个 app.Handlers也就是我们在第二篇文章中介绍到的 ControllerRegister类型的结构体了  
在 beego/router.go:622行我们也可以发现 ControllerRegister类型实现了 ServeHTTP()方法,那么这个方法就是我们的重中之重了!  
这个函数的实现有258行,所以就不贴代码了,让我们可以一点一点看
```Go
func (p *ControllerRegister) ServeHTTP(rw http.ResponseWriter, r *http.Request) {

startTime := time.Now()
var (
	runRouter  reflect.Type
	findRouter bool//是否找到路由
	runMethod  string
	routerInfo *controllerInfo
)
	context := p.pool.Get().(*beecontext.Context)
	context.Reset(rw, r)

	defer p.pool.Put(context)
	defer p.recoverPanic(context)
```
函数的开始就是先获得时间,然后从 ControllerRegister中的pool对象池获得一个　Context对象  
Context对象是对HTTP连接的封装,我们暂时可以理解为 Context中的输入就是客户端发来的信息,输出就是服务器向客户端要发送的信息,所以获得 Context对象后调用 context.Reset()传入当前的 http连接初始化它,可以方便以后的使用  

```Go
// 路由大小写敏感
	if !BConfig.RouterCaseSensitive {
		urlPath = strings.ToLower(r.URL.Path)
	} else {
		urlPath = r.URL.Path
	}
	// 检查请求中的方法是否支持
	if _, ok := HTTPMETHOD[r.Method]; !ok {
		// 不支持则返回405状态
		http.Error(rw, "Method Not Allowed", 405)
		// 记录
		goto Admin
	}
	// 检查对应 url的的 filter
	if p.execFilter(context, BeforeStatic, urlPath) {
		goto Admin
	}
	// 查找静态文件
	serverStaticRouter(context)//只对于 "GET"和"HEAD"请求方法有效
	if context.ResponseWriter.Started {
		//路由找到并且发送了相应的文件则修改 findRouter标志位并且跳转到记录
		findRouter = true
		goto Admin
	}
```
这里可以看到先根据大小写敏感配置设定 URL的大小写,然后检查传过来的方法是否支持  
HTTPMETHOD是一个全局类型的 map[string]string类型,主要保存了支持的方法  
然后调用 p.execFilter()方法,执行对应的Filter,注意这里的 p.execFilter()会在每个阶段被调用能够,这里的阶段有5个,被定义在 beego/router.go:36
```Go
const (
	BeforeStatic = iota
	BeforeRouter
	BeforeExec
	AfterExec
	FinishRouter
)
```
这里的5个阶段同时也对应着 ControllerRegister.filters中的 key,在　p.execFilters()方法中也正是用了不同阶段作为参数而在 ControllerRegister.filters成员总寻找相应的 Filter的  

接下来
就是调用 serverStaticRouter(方法查找静态文件,如果找到,并且 context.ResponseWriter.Started标志位被置位,表示已经开始发送文件,这时只需要跳转到 Admin处记录下这次连接的信息就可以结束了,否则继续执行)

```Go
	//对于非 "GET"和"HEAD"的请求方法开始解析参数
	if r.Method != "GET" && r.Method != "HEAD" {
		if BConfig.CopyRequestBody && !context.Input.IsUpload() {
			context.Input.CopyBody(BConfig.MaxMemory)
		}
		context.Input.ParseFormOrMulitForm(BConfig.MaxMemory)
	}
	// session初始化
	if BConfig.WebConfig.Session.SessionOn { //...省略}
	//检查 Filter,与659行相似,但是会查找不同的 filters数组
	if p.execFilter(context, BeforeRouter, urlPath) {
		goto Admin
	}
	//前面的静态文件未找到,开始查找路由的过程
	if !findRouter {
		httpMethod := r.Method
		if t, ok := p.routers[httpMethod]; ok {
			//这里通过 p.routers[httpMethod]进入路由树
			runObject := t.Match(urlPath, context)//找到路由树中对应的 runObject变量
			if r, ok := runObject.(*controllerInfo); ok {
				//将 runObject变量还原回 controllerInfo结构体
				routerInfo = r
				findRouter = true
				if splat := context.Input.Param(":splat"); splat != "" {
					//如果为全匹配方式,再进行分割
					for k, v := range strings.Split(splat, "/") {
						context.Input.SetParam(strconv.Itoa(k), v)
					}
				}
			}
		}
	}
  // 如果程序执行到这里还没有找到路由则抛出一个异常并跳转到记录
	if !findRouter {
		exception("404", context)
		goto Admin
	}
```
如果前面请求的不是静态文件,则开始解析请求的参数  
然后初始化 session,也就是利用 Session管理器开始一个 Session会话,并且用 defer关键字关闭会话  
接下来开始执行 BeforeRouter阶段的 Filter  
可以看到,在 findRouter标志位未被置位的情况下开始路由的查找,这里的逻辑是先通过 ControllerRegister的 routers成员(也就是第二篇文章中向里面注册路由的变量)通过请求的方法名找到对应的路由树,然后通过 Tree.Match()方法找到对应URL的Tree节点中的 runnObject对象(忘掉 runObject的话可以去看[Beego源码解析(二)-路由机制](https://riccoqu.github.io/2016/08/01/Beego%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E4%BA%8C-%E8%B7%AF%E7%94%B1%E6%9C%BA%E5%88%B6/)中那篇文章)  

只要找到了 runObject,我们就获得了当初注册路由时的自定义 Controller对象的信息,当然如果查找路由还找不到就向客户端返回"404"错误


当完成路由的查找后,接下来就应该是执行对应的方法了,这也是比较重要的一段代码
```Go
	//找到了路由,进行动作
	if findRouter {
		//查找对应的 Filter
		if p.execFilter(context, BeforeExec, urlPath) {
			goto Admin
		}
		isRunnable := false
		//找到了 routerInfo
		if routerInfo != nil {
			//RESTFul路由,执行对应的方法
			if routerInfo.routerType == routerTypeRESTFul {
				//查找到了对应的方法,所以调用回调函数
				if _, ok := routerInfo.methods[r.Method]; ok {
					isRunnable = true
					routerInfo.runFunction(context)//执行对应的回调方法
				} else {
					exception("405", context)
					goto Admin
				}
			} else if routerInfo.routerType == routerTypeHandler {
				//Handler类型的路由
				isRunnable = true
				routerInfo.handler.ServeHTTP(rw, r)//执行　Handler类型的路由回调方法
			} else {
				//其他类型的请求
				runRouter = routerInfo.controllerType
				method := r.Method
				if r.Method == "POST" && context.Input.Query("_method") == "PUT" {
					method = "PUT"
				}
				if r.Method == "POST" && context.Input.Query("_method") == "DELETE" {
					method = "DELETE"
				}
				if m, ok := routerInfo.methods[method]; ok {
					//如果找到注册过的方法,则赋值
					runMethod = m
				} else if m, ok = routerInfo.methods["*"]; ok {
					//如果支持通配符'*',则赋值
					runMethod = m
				} else {
					runMethod = method
				}
			}
		}

		// also defined runRouter & runMethod from filter
		//没有正在运行的情况,即在732行的两个 if都没有匹配到
		if !isRunnable {
			//Invoke the request handler
			vc := reflect.New(runRouter)
			execController, ok := vc.Interface().(ControllerInterface)//这是为了还原出注册时的 Controller结构体变量
			if !ok {
				panic("controller is not ControllerInterface")
			}

			//call the controller init function
			//调用 Controller的 init()函数
			execController.Init(context, runRouter.Name(), runMethod, vc.Interface())

			//call prepare function
			//调用 Controller的 Prepare()函数
			execController.Prepare()

			//if XSRF is Enable then check cookie where there has any cookie in the  request's cookie _csrf
			if BConfig.WebConfig.EnableXSRF {
				execController.XSRFToken()
				if r.Method == "POST" || r.Method == "DELETE" || r.Method == "PUT" ||
					(r.Method == "POST" && (context.Input.Query("_method") == "DELETE" || context.Input.Query("_method") == "PUT")) {
					execController.CheckXSRFCookie()
				}
			}

			execController.URLMapping()
			//根据方法调用不同的处理函数
			if !context.ResponseWriter.Started {
				//exec main logic
				switch runMethod {
				case "GET":
					execController.Get()
				case "POST":
					execController.Post()
				case "DELETE":
					execController.Delete()
				case "PUT":
					execController.Put()
				case "HEAD":
					execController.Head()
				case "PATCH":
					execController.Patch()
				case "OPTIONS":
					execController.Options()
				default:
					if !execController.HandlerFunc(runMethod) {
						var in []reflect.Value
						method := vc.MethodByName(runMethod)
						method.Call(in)
					}
				}

				//render template
				if !context.ResponseWriter.Started && context.Output.Status == 0 {
					if BConfig.WebConfig.AutoRender {
						if err := execController.Render(); err != nil {
							panic(err)
						}
					}
				}
			}

			// finish all runRouter. release resource
			//处理完,进行资源的释放
			execController.Finish()
		}

		//execute middleware filters
		//执行对应的 Filters
		if p.execFilter(context, AfterExec, urlPath) {
			goto Admin
		}
	}
	//在处理完请求后,执行对应的 Filters
	p.execFilter(context, FinishRouter, urlPath)
```
可以看到中间有几个 if else用来判断注册的路由是什么类型
* 如果类型是 routerTypeRESTFul并且支持请求的方法,则执行 ControllerInfo的runFunction()方法,否则返回"405"
* 如果类型是 routerTypeHandler,则执行 Controller.handler的ServeHTTP()方法
* 其他类型的路由,则继续往下执行,注意有两行 "POST"方法的判断,但是 method又改为其他的方法,这是为了用同一个接口实现不同的请求,既将真实的请求放在参数 _method里面

下面的 if !isRunnable{...} 是当上面匹配到其他类型路由时而执行的流程  
可以发现通过反射获得了当初注册路由时的 Controller结构体,并依次调用 ControllerInterface接口所定义的方法(这些方法正在在自定义 Controller时需要实现的方法)  
整个 ServeHTTP()函数的最后调用了 AfterExec阶段和 FinishRouter阶段两个阶段的 Filter  

在后面就是整个请求结束后的记录过程了,逻辑比较简单只贴上代码就好了~
```Go
/*
 * 记录此次请求的一些信息
 */
Admin:
	timeDur := time.Since(startTime)
	//admin module record QPS
	if BConfig.Listen.EnableAdmin {
		if FilterMonitorFunc(r.Method, r.URL.Path, timeDur) {
			if runRouter != nil {
				go toolbox.StatisticsMap.AddStatistics(r.Method, r.URL.Path, runRouter.Name(), timeDur)
			} else {
				go toolbox.StatisticsMap.AddStatistics(r.Method, r.URL.Path, "", timeDur)
			}
		}
	}

	if BConfig.RunMode == DEV || BConfig.Log.AccessLogs {
		var devInfo string
		if findRouter {
			if routerInfo != nil {
				devInfo = fmt.Sprintf("| % -10s | % -40s | % -16s | % -10s | % -40s |", r.Method, r.URL.Path, timeDur.String(), "match", routerInfo.pattern)
			} else {
				devInfo = fmt.Sprintf("| % -10s | % -40s | % -16s | % -10s |", r.Method, r.URL.Path, timeDur.String(), "match")
			}
		} else {
			devInfo = fmt.Sprintf("| % -10s | % -40s | % -16s | % -10s |", r.Method, r.URL.Path, timeDur.String(), "notmatch")
		}
		if DefaultAccessLogFilter == nil || !DefaultAccessLogFilter.Filter(context) {
			Debug(devInfo)
		}
	}

	// Call WriteHeader if status code has been set changed
	if context.Output.Status != 0 {
		context.ResponseWriter.WriteHeader(context.Output.Status)
	}
}
```

一个HTTP请求的处理流程大概就是这个样子,先查找静态路由让后再查找用户定义的路由规则,调用相应规则的处理函数


如有错误,非常希望您能联系我并让我加以改进 :D
