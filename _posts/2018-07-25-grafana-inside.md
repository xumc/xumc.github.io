---
layout: post
title: "grafana http部分源码解析"
date: 2018-07-25
---
作为一名web开发者，笔者使用Golang作为后端语言在开发Web服务的时候遇到过很多问题，
1. Golang作为静态语言，在业务逻辑开发速度方面，相较于Ruby等动态语言并不占有太大优势。使用Golang开发Web系统难免会使得代码可读性，代码整洁不如Ruby等语言来的直观，简洁。但是当我们面对海量用户请求的时候，相同的硬件配置下，使用golang会让我们开发的服务的性能更加好。
2. 相较于Ruby语言中的Rails框架，Golang生态里面的Web开发框架并没有集大成者，把Routing，ORM等帮我们处理好。也就没有Rails生态圈中best practice.很多时候，使用Golang开发，意味着我们需要自己写一些粘合代码，让Golang生态中的优秀第三方框架可以各司其职，协同工作。
3. 相较于Rails生态圈，Golang提供的单元测试工具真可谓简陋至极。

grafana 作为一个典型的Web系统，为我们使用Golang开发web系统提供了良好的范例。本文旨在通过解读grafana 在Http service 方面的代码，帮助Web开发者“写一些粘合代码”。

首先，我们先看看granfana HTTP service 所使用的第三方开源框架。
1. https://github.com/go-macaron/macaron https://go-macaron.com/
2. https://github.com/go-macaron/binding
3. github.com/go-xorm/xorm  http://xorm.io/
4. github.com/go-xorm/builder
5. https://github.com/hashicorp/go-hclog
6. github.com/inconshreveable/log15
7. github.com/opentracing/opentracing-go
8. github.com/patrickmn/go-cache
9. github.com/smartystreets/goconvey
10. github.com/smartystreets/assertions

## 1. 启动流程

http server 包含http service所有需要的基础组件。

```go
// github.com/grafana/grafana/pkg/api/http_server.go:38
type HTTPServer struct {
	log           log.Logger
	macaron       *macaron.Macaron
	context       context.Context
	streamManager *live.StreamManager
	cache         *gocache.Cache
	httpSrv       *http.Server

	RouteRegister RouteRegister     `inject:""`
	Bus           bus.Bus           `inject:""`
	RenderService rendering.Service `inject:""`
	Cfg           *setting.Cfg      `inject:""`
}
```

granfana 启动的时候，会通过grafana 的registry机制自动注册http server. HTTPServer实现了registry中定义的Service interface，所以启动granfana的时候，会调用 HTTPServer 的Init 方法。Init方法主要初始化HTTPServer struct定义的log以及cache组件。 HTTPServer 实现了registry中定义的BackgroundService方法，所以启动granfana的时候，会调用HTTPServer 的Run方法。Run方法会创建Macaron实例，注册路由，监听http端口，并且处理shutdown逻辑。至此，http server启动完毕。

```go
// github.com/grafana/grafana/pkg/api/http_server.go:59
func (hs *HTTPServer) Run(ctx context.Context) error {
	var err error

	hs.context = ctx
	hs.streamManager = live.NewStreamManager()
	hs.macaron = hs.newMacaron()
	hs.registerRoutes()

	hs.streamManager.Run(ctx)

	listenAddr := fmt.Sprintf("%s:%s", setting.HttpAddr, setting.HttpPort)
	hs.log.Info("HTTP Server Listen", "address", listenAddr, "protocol", setting.Protocol, "subUrl", setting.AppSubUrl, "socket", setting.SocketPath)

	hs.httpSrv = &http.Server{Addr: listenAddr, Handler: hs.macaron}

	// handle http shutdown on server context done
	go func() {
		<-ctx.Done()
		// Hacky fix for race condition between ListenAndServe and Shutdown
		time.Sleep(time.Millisecond * 100)
		if err := hs.httpSrv.Shutdown(context.Background()); err != nil {
			hs.log.Error("Failed to shutdown server", "error", err)
		}
	}()

	switch setting.Protocol {
	case setting.HTTP:
		err = hs.httpSrv.ListenAndServe()
		if err == http.ErrServerClosed {
			hs.log.Debug("server was shutdown gracefully")
			return nil
		}
	case setting.HTTPS:
		err = hs.listenAndServeTLS(setting.CertFile, setting.KeyFile)
		if err == http.ErrServerClosed {
			hs.log.Debug("server was shutdown gracefully")
			return nil
		}
	case setting.SOCKET:
		ln, err := net.ListenUnix("unix", &net.UnixAddr{Name: setting.SocketPath, Net: "unix"})
		if err != nil {
			hs.log.Debug("server was shutdown gracefully")
			return nil
		}

		// Make socket writable by group
		os.Chmod(setting.SocketPath, 0660)

		err = hs.httpSrv.Serve(ln)
		if err != nil {
			hs.log.Debug("server was shutdown gracefully")
			return nil
		}
	default:
		hs.log.Error("Invalid protocol", "protocol", setting.Protocol)
		err = errors.New("Invalid Protocol")
	}

	return err
}
```

```go
// github.com/grafana/grafana/pkg/api/http_server.go:52
func (hs *HTTPServer) Init() error {
	hs.log = log.New("http.server")
	hs.cache = gocache.New(5*time.Minute, 10*time.Minute)

	return nil
}
```

## 2. log

grafana 采用了github.com/inconshreveable/log15第三方log组件写log，并且做了简单的封装。主要针对config文件中logging相关的配置写了一些逻辑代码。有一点是和大部分系统不一样的， 在grafana中，不同的组件拥有不同的log对象，比如HTTPServer有自己的log对象，xorm（grafana 采用的ORM框架）也有自己的log对象。这些log对象产生的log会合成一个log，输出到stdout, log file。
## 3. grafana采用Macaron 作为Go Web 框架

   作为一款具有高生产力和模块化设计的web框架， Macaron提供了功能丰富的中间件模块，能满足基本Web服务的需求。需要Macaron解决的第一个问题是web系统的routing问题，grafana针对Macaron的routing做了适当的封装。

   首先pkg/api/route_register.go 文件中定义了注册路由的逻辑。grafana把处理request分成namedmiddleware, subfixHandlers,和handler三部分。其中namedmiddleware是一种特殊的middleware，在grafana中， 通过github.com/facebookgo/inject设置了两个namedmiddleware, 分别为ReqeustMetrics 和 RequestTracing.

```go
// github.com/grafana/grafana/pkg/cmd/grafana-server/server.go:78
	serviceGraph.Provide(&inject.Object{Value: api.NewRouteRegister(middleware.RequestMetrics, middleware.RequestTracing)})

```

  subfixHandler则是应用于Group类型路由的一类handler。 最后的handler(s)才是处理request核心逻辑的函数。route_register.go 定义了注册路由的逻辑， 而pkg/api/api.go则主要定义了具体的路由。我们以api.go中的admin api为例解释这套逻辑。 reqGrafanaAdmin函数是说凡是访问/api/admin/*的request都必须要具有admin role。AdminGetSetting 是处理核心业务的函数,它接收一个ReqContext对象作为参数， ReqContext是grafana基于macaron.Context定义的request对象。 其中bind(dtos.AdminCreateUserFrom{})以及wrap(...)等也是handler，只是使用了一些bind 以及wrap方法做了一些处理。通过这种分门别类的路由， granfana减少了重复代码的产生，而且代码更简洁，易懂。


```go
    // github.com/grafana/grafana/pkg/api/api.go:368
	// admin api
	r.Group("/api/admin", func(adminRoute RouteRegister) {
		adminRoute.Get("/settings", AdminGetSettings)
		adminRoute.Post("/users", bind(dtos.AdminCreateUserForm{}), AdminCreateUser)
		adminRoute.Put("/users/:id/password", bind(dtos.AdminUpdateUserPasswordForm{}), AdminUpdateUserPassword)
		adminRoute.Put("/users/:id/permissions", bind(dtos.AdminUpdateUserPermissionsForm{}), AdminUpdateUserPermissions)
		adminRoute.Delete("/users/:id", AdminDeleteUser)
		adminRoute.Get("/users/:id/quotas", wrap(GetUserQuotas))
		adminRoute.Put("/users/:id/quotas/:target", bind(m.UpdateUserQuotaCmd{}), wrap(UpdateUserQuota))
		adminRoute.Get("/stats", AdminGetStats)
		adminRoute.Post("/pause-all-alerts", bind(dtos.PauseAllAlertsCommand{}), wrap(PauseAllAlerts))
	}, reqGrafanaAdmin)

```

```go
// github.com/grafana/grafana/pkg/api/admin.go:12
func AdminGetSettings(c *m.ReqContext) {
	settings := make(map[string]interface{})

	for _, section := range setting.Raw.Sections() {
		jsonSec := make(map[string]interface{})
		settings[section.Name()] = jsonSec

		for _, key := range section.Keys() {
			keyName := key.Name()
			value := key.Value()
			if strings.Contains(keyName, "secret") || strings.Contains(keyName, "password") || (strings.Contains(keyName, "provider_config")) {
				value = "************"
			}
			if strings.Contains(keyName, "url") {
				var rgx = regexp.MustCompile(`.*:\/\/([^:]*):([^@]*)@.*?$`)
				var subs = rgx.FindAllSubmatch([]byte(value), -1)
				if subs != nil && len(subs[0]) == 3 {
					value = strings.Replace(value, string(subs[0][1]), "******", 1)
					value = strings.Replace(value, string(subs[0][2]), "******", 1)
				}
			}

			jsonSec[keyName] = value
		}
	}

	c.JSON(200, settings)
}
```

```go
// github.com/grafana/grafana/pkg/models/context.go:14
type ReqContext struct {
	*macaron.Context
	*SignedInUser

	Session session.SessionStore

	IsSignedIn     bool
	IsRenderCall   bool
	AllowAnonymous bool
	Logger         log.Logger
}

```

## 4. bus机制

  在pkg/bus/bus.go中，定义了bus机制。bus主要利用供refect解决了接口依赖问题。在web系统中，我们往往把系统进行分层， 在grafana中， pkg/api package 属于接入层， pkg/api层会调用service层提供的业务逻辑代码。在api layer，我们为了能够调用service提供的函数并且方便些Unit test，传统的做法是在service提供的业务逻辑函数抽象出interface，这种做法虽然可行，但是很啰嗦。而且在写unit test的时候，会更加的啰嗦。 grafana中，提供bus机制来解决这个问题。首先，系统启动或者运行时， 可以通过bus.AddHandler函数注册HandlerFunc处理函数到globalBus对象， 然后在api层调用bus.Dispatch的时候，我们会提供一个strcut 对象作为msg，Dispatch会调用注册的HandlerFunc函数处理strcut对象，并且将处理结果写入到strcut对象中。
我们以api/admin/stats 为例，在AdminGetStats handler函数中，首先生成GetAdminStatsQuery struct 实例，然后利用bus.Dispatch 方法将query dispatch出去。最后由init函数注册的GetAdminStats函数处理，并且更新GetAdminStatsQuery strcut的Result的值。最后，AdminGetStats 会把statsQuery.Result以json形式返回给客户端。
这种机制在我们写unit test的时候mock service 函数非常实用，用起来也非常方便。


```go
// github.com/grafana/grafana/pkg/services/sqlstore/stats.go:11
func init() {
	bus.AddHandler("sql", GetSystemStats)
	bus.AddHandler("sql", GetDataSourceStats)
	bus.AddHandler("sql", GetDataSourceAccessStats)
	bus.AddHandler("sql", GetAdminStats)
	bus.AddHandlerCtx("sql", GetSystemUserCountStats)
}
```

```go
// github.com/grafana/grafana/pkg/api/api.go:376
		adminRoute.Get("/stats", AdminGetStats)
```

```go
// github.com/grafana/grafana/pkg/api/admin.go:12
func AdminGetSettings(c *m.ReqContext) {
	settings := make(map[string]interface{})

	for _, section := range setting.Raw.Sections() {
		jsonSec := make(map[string]interface{})
		settings[section.Name()] = jsonSec

		for _, key := range section.Keys() {
			keyName := key.Name()
			value := key.Value()
			if strings.Contains(keyName, "secret") || strings.Contains(keyName, "password") || (strings.Contains(keyName, "provider_config")) {
				value = "************"
			}
			if strings.Contains(keyName, "url") {
				var rgx = regexp.MustCompile(`.*:\/\/([^:]*):([^@]*)@.*?$`)
				var subs = rgx.FindAllSubmatch([]byte(value), -1)
				if subs != nil && len(subs[0]) == 3 {
					value = strings.Replace(value, string(subs[0][1]), "******", 1)
					value = strings.Replace(value, string(subs[0][2]), "******", 1)
				}
			}

			jsonSec[keyName] = value
		}
	}

	c.JSON(200, settings)
}
```


  dispatch机制中，每个msg由一个HandlerFunc处理。 除了dispatch 模式外， bus还提供了Publish机制， 每个msg可以由多个handlerFunc处理。实现机制和dispatch类似，只是可以注册多个handlerFunc处理对应的event。（publish机制中，把msg成为event）。我们以signup 为例解释publish机制。在用户signup成功时， pkg/api/signup.go 会调用bus.Publish方法产生SignUpCompleted event。在用户注册完成后，我们往往会发送注册成功邮件，在grafana中，发送注册成功邮件是由NotificationService完成的。NotificationService 会监听SignUpCompleted event, 如果有SignUpCompleted event产生，NotificationService就会执行发送邮件的逻辑。

```go
// github.com/grafana/grafana/pkg/services/notifications/notifications.go:52
	ns.Bus.AddEventListener(ns.signUpCompletedHandler)

```

```go
// github.com/grafana/grafana/pkg/api/signup.go:87
	// publish signup event
	user := &createUserCmd.Result
	bus.Publish(&events.SignUpCompleted{
		Email: user.Email,
		Name:  user.NameOrFallback(),
	})

	// mark temp user as completed
	if ok, rsp := updateTempUserStatus(form.Code, m.TmpUserCompleted); !ok {
		return rsp
	}
```

  需要注意的是，不论dispatch还是publish机制， handlerFunc 都是同步完成的，dispatch函数会调用对应的handlerFunc, 然后执行handlerFunc， handlerFunc处理过程中，产生错误的话，错误会作为dispatch的返回值返回。在publish机制中， 一个event会被多个HandlerFunc处理。假设有三个HandlerFunc,分别为HandlerFunc1， HandlerFunc2， HandlerFunc3. bus.Publish会按照AddEventListener的顺序执行三个HandlerFunc， 假设HandlerFunc1执行成功， HandlerFunc2由于种种原因执行失败了，这个时候Publish就会返回，HandlerFunc3就不会被执行。 
这一点和MQ产品中的异步机制是完全不同的。

## 5. Unit test
  在golang的世界里面，相对于ruby等语言来说，写单元测试的确是一个繁琐无趣的活。一般来说，我们会通过golang的interface来mock一些方法。这种mock方法又衍生出两种写法。 比如，我们有一个待测方法。 func1， 里面有一个已经被测试过无需写单元测试的方法func2, 那么我们有两种方式来写单元测试。
  方法1， 利用“type func"为func2抽象出一个接口interface_for_func2, 然后这个接口作为func1的一个参数。在运行的代码中，把func2传入func1作为参数。 在单元测试中，则用一个实现了interface_for_func2的mock方法传入func1.这样就实现了mock方法的目的。
  方法2， 原理上和方法1是类似的，只是写法上有一些不同。 我们不在使用type func 抽象接口，而是为func2专门定义一个接口， 然后用一个struct来实现这些接口，在运行的代码中，使用实现了接口的struct来调用func2， 而在单元测试中，则mock一个实现了接口的mock的struct来调用mock的func2.
这两种方法在https://github.com/DATA-DOG/go-sqlmock和https://github.com/vektra/mockery都有体现。sqlmock的缺点是太过于负责，对鞋单元测试来说，需要构造大量的sql返回值。 mockery则需要生成大量mock的函数。

  然而在grafana中， 使用了bus机制来解耦不通层面的代码。 以达到方便些单元测试的目的。 以下图里面的一个case为例，我们只需要使用bus.AddHandler就可以构造mock函数了。 不需要生成代码，简单方便。

```go
// github.com/grafana/grafana/pkg/api/dashboard_permission_test.go:15
func TestDashboardPermissionApiEndpoint(t *testing.T) {
	Convey("Dashboard permissions test", t, func() {
		Convey("Given dashboard not exists", func() {
			bus.AddHandler("test", func(query *m.GetDashboardQuery) error {
				return m.ErrDashboardNotFound
			})

			loggedInUserScenarioWithRole("When calling GET on", "GET", "/api/dashboards/id/1/permissions", "/api/dashboards/id/:id/permissions", m.ROLE_EDITOR, func(sc *scenarioContext) {
				callGetDashboardPermissions(sc)
				So(sc.resp.Code, ShouldEqual, 404)
			})

			cmd := dtos.UpdateDashboardAclCommand{
				Items: []dtos.DashboardAclUpdateItem{
					{UserId: 1000, Permission: m.PERMISSION_ADMIN},
				},
			}

			updateDashboardPermissionScenario("When calling POST on", "/api/dashboards/id/1/permissions", "/api/dashboards/id/:id/permissions", cmd, func(sc *scenarioContext) {
				callUpdateDashboardPermissions(sc)
				So(sc.resp.Code, ShouldEqual, 404)
			})
		})

```

  本文仅仅解析了grafana中的http相关的部分的代码，还有很多代码没有touch到。以后再予以解析。

