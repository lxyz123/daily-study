## Net/http
### 举例
```go
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello World"))
    })

    if err := http.ListenAndServe(":8000", nil); err != nil {
        fmt.Println("start http server fail:", err)
    }
}
```
### 注册
```go
// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

// HandleFunc registers the handler function for the given pattern.
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}

// 注册
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()

    if pattern == "" {
        panic("http: invalid pattern")
    }
    if handler == nil {
        panic("http: nil handler")
    }
    if _, exist := mux.m[pattern]; exist {
        panic("http: multiple registrations for " + pattern)
    }

    if mux.m == nil {
        mux.m = make(map[string]muxEntry)
    }
    mux.m[pattern] = muxEntry{h: handler, pattern: pattern}

    if pattern[0] != '/' {
        mux.hosts = true
    }
}
```
### 执行
```go
// 创建socket
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}

func (srv *Server) ListenAndServe() error {
    // ... 省略代码
    ln, err := net.Listen("tcp", addr) // <-----看这里listen
    if err != nil {
      return err
    }
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}

// Accept等待客户端连接
func (srv *Server) Serve(l net.Listener) error {
    // ... 省略代码
    for {
      rw, e := l.Accept() // <----- 看这里accept
      if e != nil {
        select {
        case <-srv.getDoneChan():
          return ErrServerClosed
        default:
        }
        if ne, ok := e.(net.Error); ok && ne.Temporary() {
          if tempDelay == 0 {
            tempDelay = 5 * time.Millisecond
          } else {
            tempDelay *= 2
          }
          if max := 1 * time.Second; tempDelay > max {
            tempDelay = max
          }
          srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
          time.Sleep(tempDelay)
          continue
        }
        return e
      }
      tempDelay = 0
      c := srv.newConn(rw)
      c.setState(c.rwc, StateNew) // before Serve can return
      go c.serve(ctx) // <--- 看这里
    }
}

func (c *conn) serve(ctx context.Context) {
    // ... 省略代码
    serverHandler{c.server}.ServeHTTP(w, w.req)
    w.cancelCtx()
    if c.hijacked() {
      return
    }
    w.finishRequest()
    // ... 省略代码
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
      handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
      handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}

// net/http/server.go:L2352-2362
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
      if r.ProtoAtLeast(1, 1) {
        w.Header().Set("Connection", "close")
      }
      w.WriteHeader(StatusBadRequest)
      return
    }
    h, _ := mux.Handler(r) // <--- 看这里
    h.ServeHTTP(w, r)
}

func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {

	// CONNECT requests are not canonicalized.
	if r.Method == "CONNECT" {
		// If r.URL.Path is /tree and its handler is not registered,
		// the /tree -> /tree/ redirect applies to CONNECT requests
		// but the path canonicalization does not.
		if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
			return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
		}

		return mux.handler(r.Host, r.URL.Path)
	}

	// All other requests have any port stripped and path cleaned
	// before passing to mux.handler.
	host := stripHostPort(r.Host)
	path := cleanPath(r.URL.Path)

	// If the given path is /tree and its handler is not registered,
	// redirect for /tree/.
	if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
		return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
	}

	if path != r.URL.Path {
		_, pattern = mux.handler(host, path)
		u := &url.URL{Path: path, RawQuery: r.URL.RawQuery}
		return RedirectHandler(u.String(), StatusMovedPermanently), pattern
	}

	return mux.handler(host, r.URL.Path)
}

// handler is the main implementation of Handler.
// The path is known to be in canonical form, except for CONNECT methods.
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// Host-specific pattern takes precedence over generic ones
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path)
	}
	if h == nil {
		h, pattern = NotFoundHandler(), ""
	}
	return
}

// Find a handler on a handler map given a path string.
// Most-specific (longest) pattern wins.
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// Check for exact match first.
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	// Check for longest valid match.  mux.es contains all patterns
	// that end in / sorted from longest to shortest.
	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}
```
## 比较
### 性能比较
| Benchmark name | (1) | (2) | (3) | (4) |
| --- | --- | --- | --- | --- |
| BenchmarkGin_GithubAll | **43550** | **27364 ns/op** | **0 B/op** | **0 allocs/op** |
| BenchmarkAce_GithubAll | 40543 | 29670 ns/op | 0 B/op | 0 allocs/op |
| BenchmarkAero_GithubAll | 57632 | 20648 ns/op | 0 B/op | 0 allocs/op |
| BenchmarkBear_GithubAll | 9234 | 216179 ns/op | 86448 B/op | 943 allocs/op |
| BenchmarkBeego_GithubAll | 7407 | 243496 ns/op | 71456 B/op | 609 allocs/op |
| BenchmarkBone_GithubAll | 420 | 2922835 ns/op | 720160 B/op | 8620 allocs/op |
| BenchmarkChi_GithubAll | 7620 | 238331 ns/op | 87696 B/op | 609 allocs/op |
| BenchmarkDenco_GithubAll | 18355 | 64494 ns/op | 20224 B/op | 167 allocs/op |
| BenchmarkEcho_GithubAll | 31251 | 38479 ns/op | 0 B/op | 0 allocs/op |
| BenchmarkGocraftWeb_GithubAll | 4117 | 300062 ns/op | 131656 B/op | 1686 allocs/op |
| BenchmarkGoji_GithubAll | 3274 | 416158 ns/op | 56112 B/op | 334 allocs/op |
| BenchmarkGojiv2_GithubAll | 1402 | 870518 ns/op | 352720 B/op | 4321 allocs/op |
| BenchmarkGoJsonRest_GithubAll | 2976 | 401507 ns/op | 134371 B/op | 2737 allocs/op |
| BenchmarkGoRestful_GithubAll | 410 | 2913158 ns/op | 910144 B/op | 2938 allocs/op |
| BenchmarkGorillaMux_GithubAll | 346 | 3384987 ns/op | 251650 B/op | 1994 allocs/op |
| BenchmarkGowwwRouter_GithubAll | 10000 | 143025 ns/op | 72144 B/op | 501 allocs/op |
| BenchmarkHttpRouter_GithubAll | 55938 | 21360 ns/op | 0 B/op | 0 allocs/op |
| BenchmarkHttpTreeMux_GithubAll | 10000 | 153944 ns/op | 65856 B/op | 671 allocs/op |
| BenchmarkKocha_GithubAll | 10000 | 106315 ns/op | 23304 B/op | 843 allocs/op |
| BenchmarkLARS_GithubAll | 47779 | 25084 ns/op | 0 B/op | 0 allocs/op |
| BenchmarkMacaron_GithubAll | 3266 | 371907 ns/op | 149409 B/op | 1624 allocs/op |
| BenchmarkMartini_GithubAll | 331 | 3444706 ns/op | 226551 B/op | 2325 allocs/op |
| BenchmarkPat_GithubAll | 273 | 4381818 ns/op | 1483152 B/op | 26963 allocs/op |
| BenchmarkPossum_GithubAll | 10000 | 164367 ns/op | 84448 B/op | 609 allocs/op |
| BenchmarkR2router_GithubAll | 10000 | 160220 ns/op | 77328 B/op | 979 allocs/op |
| BenchmarkRivet_GithubAll | 14625 | 82453 ns/op | 16272 B/op | 167 allocs/op |
| BenchmarkTango_GithubAll | 6255 | 279611 ns/op | 63826 B/op | 1618 allocs/op |
| BenchmarkTigerTonic_GithubAll | 2008 | 687874 ns/op | 193856 B/op | 4474 allocs/op |
| BenchmarkTraffic_GithubAll | 355 | 3478508 ns/op | 820744 B/op | 14114 allocs/op |
| BenchmarkVulcan_GithubAll | 6885 | 193333 ns/op | 19894 B/op | 609 allocs/op |

- 在一定的时间内实现的总调用数，越高越好
- 单次操作耗时（ns/op），越低越好
- 堆内存分配 （B/op）, 越低越好
- 每次操作的平均内存分配次数（allocs/op），越低越好
### 比较总结
|  | GoFrame | Beego | Iris | Gin |
| --- | --- | --- | --- | --- |
| 比较版本 | v1.15.2 | v1.12.3 | v12.0.2 | v1.6.3 |
| 项目类型 | 开源（国内） | 开源（国内） | 开源（海外） | 开源（海外） |
| 开源协议 | MIT | Apache-2 | BSD-3-Clause | MIT |
| 框架类型 | 模块化框架 | Web框架 | Web"框架" | Web"框架" |
| 基本介绍 | 工程完备、简单易用，模块化、高质量、高性能、企业级开发框架。 | 最简单易用的企业级Go应用开发框架。 | 目前发展最快的Go Web框架。提供完整的MVC功能并且面向未来。 | 一个Go语言写的HTTP Web框架。它提供了Martini风格的API并有更好的性能。 |
| 项目地址 | github.com/gogf/gf | github.com/beego/beego | github.com/kataras/iris | github.com/gin-gonic/gin |
| 官网地址 | goframe.org | beego.me | iris-go.com | gin-gonic.com |
| 模块化设计 | 是 | - | - | - |
| 模块完善度 | 10 | 6 | 4 | 2 |
| 使用易用性 | 9 | 9 | 9 | 10 |
| 文档完善度 | 10 | 8 | 6 | 4 |
| 工程化完备 | 10 | 8 | 5 | 1 |
| 社区活跃 | 9 | 10 | 9 | 10 |
| 开发模式 | 模块引入、三层架构、MVC | MVC | MVC | - |
| 工程规范 | 分层设计、对象设计 | 项目结构 | - | - |
| 开发工具链 | gf工具链 | bee工具链 | - | - |
| Web: 性能测试 | 10 | 8 | 8 | 9 |
| Web: HTTPS | HTTPS & TLS | 支持 | CustomHttpConfiguration | 支持 |
| Web: HTTP2 | - | - | 支持 | 支持 |
| Web: WebSocket | WebSocket服务 | 有 | 有 | - |
| Web: 分组路由 | 分组路由 | Namespace | GroupingRoutes | 有 |
| Web: 路由冲突处理 | 有 | - | 有 | - |
| Web: 域名支持 | 域名绑定 | - | - | - |
| Web: 文件服务 | 静态文件服务 | 静态文件处理 | ServingStaticFiles | 有 |
| Web: 多端口/实例 | 多端口监听、多实例监听 | - | RunMultipleServiceUsingIris | - |
| Web: 优雅重启/关闭 | 平滑重启特性 | 热升级 | GracefulShutdownOrRestart | - |
| ORM | ORM文档 | ORM文档 | - | - |
| Session | Session | Session | 有 | - |
| I18N支持 | I18N | I18N | Localization | - |
| 模板引擎 | 模板引擎 | View设计 | TemplateRendering | STD Template |
| 配置管理 | 配置管理 | 参数配置 | - | - |
| 日志组件 | 日志组件 | Logging | - | - |
| 数据校验 | 数据校验 | 表单数据验证 | - | - |
| 缓存管理 | 缓存管理 | Cache | - | - |
| 资源打包 | 资源管理 | bee工具bale命令 | - | - |
| 链路跟踪 | 链路跟踪 | - | - | - |
| 测试框架 | 单元测试 | - | Testing | Testing |
| 突出优点 | goframe主要以工程化和企业级方向为主，特别是模块化设计和工程化设计思想非常棒。针对业务项目而言，提供了开发规范、项目规范、命名规范、设计模式、开发工具链、丰富的模块、高质量代码和文档，社区活跃。作者也是资深的PHP开发者，PHP转Go的小伙伴会倍感亲切。 | beego开源的比较早，最早的一款功能比较全面的Golang开发框架，一直在Golang领域有着比较大的影响力，beego有着比较丰富的开发模块、开箱即用，提供了基于MVC设计模式的项目结构、开发工具链，主要定位为Web开发，当然也可以用于非Web项目开发。 | iris主要侧重于Web开发，提供了Web开发的一系列功能组件，基于MVC开发模式。iris这一年发展比较快，从一个Web Server的组件，也慢慢朝着beego的设计方向努力。 | gin专注于轻量级的Web Server，比较简单，易于理解，路由和中间件设计不错，可以看做替代标准库net/http.Server的路由加强版web server。献给爱造轮子的朋友们。 |
| 突出缺点 | 开源时间较晚，推广过于佛系，目前主要面向国内用户，未推广海外。 | 起步较早，自谢大创业后，近几年发展较慢。非模块化设计，对第三方重量级模块依赖较多。 | 号称性能最强，结果平平。非模块化设计。最近两年开始朝beego方向发展，但整体框架能力还不完备，需要加油。 | 功能简单易用，既是优点，也是缺点。 |

## Gin框架介绍
> [Gin](https://github.com/gin-gonic/gin) is a web framework written in Go (Golang). It features a martini-like API with performance that is up to 40 times faster thanks to [httprouter](https://github.com/julienschmidt/httprouter). If you need performance and good productivity, you will love Gin.

## 特性

- **快速**：路由不使用反射，基于Radix树，内存占用少。
- **中间件**：HTTP请求，可先经过一系列中间件处理，例如：Logger，Authorization等。中间件机制也极大地提高了框架的可扩展性。
- **异常处理**：服务始终可用，不会宕机。Gin 可以捕获 panic，并恢复。而且有极为便利的机制处理HTTP请求过程中发生的错误。
- **JSON**：Gin可以解析并验证请求的JSON。这个特性对Restful API的开发尤其有用。
- **路由分组**：例如将需要授权和不需要授权的API分组，不同版本的API分组。而且分组可嵌套，且性能不受影响。
- **渲染内置**：原生支持JSON，XML和HTML的渲染。
- **可继承**：简单的去创建中间件
## Jsoniter
 Gin 使用 encoding/json 作为默认的 json 包，但是你可以在编译中使用标签将其修改为 [jsoniter](https://github.com/json-iterator/go)
> **$ go build -tags=jsoniter .**

## 代码结构
```go
|-- binding                     将请求的数据对象化并校验
|-- examples                    各种例子
|-- json                        提供了另外一种json实现
|-- render                      响应

|-- gin.go                      gin engine所在
|-- gin_test.go
|-- routes_test.go
|-- context.go                  上下文，将各种功能聚焦到上下文（装饰器模式）
|-- context_test.go
|-- response_writer.go          响应的数据输出
|-- response_writer_test.go
|-- errors.go                   错误处理
|-- errors_test.go
|-- tree.go                     路由的具体实现
|-- tree_test.go
|-- routergroup.go
|-- routergroup_test.go
|-- auth.go                     一个基本的HTTP鉴权的中间件
|-- auth_test.go
|-- logger.go                   一个日志中间件
|-- logger_test.go
|-- recovery.go                 一个崩溃处理插件
|-- recovery_test.go

|-- mode.go                     应用模式
|-- mode_test.go
|-- utils.go                    
|-- utils_test.go
```
## Gin框架安装与使用
> **go get -u github.com/gin-gonic/gin**

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	// 创建一个默认的路由引擎
	r := gin.Default()
	// GET：请求方式；/hello：请求的路径
	// 当客户端以GET方法请求/hello路径时，会执行后面的匿名函数
	r.GET("/hello", func(c *gin.Context) {
		// c.JSON：返回JSON格式的数据
		c.JSON(200, gin.H{
			"message": "Hello world!",
		})
	})
	// 启动HTTP服务，默认在0.0.0.0:8080启动服务
	r.Run()
}
```
![{95c8e821-3e6e-4d66-a118-d0a50e08959c}.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666344157472-c77fad8c-4757-4f14-ad8b-e4fe4b5db018.png#clientId=u9dc2b72d-e312-4&errorMessage=unknown%20error&from=paste&height=90&id=u5156b360&name=%7B95c8e821-3e6e-4d66-a118-d0a50e08959c%7D.png&originHeight=90&originWidth=301&originalType=binary&ratio=1&rotation=0&showTitle=false&size=2874&status=error&style=none&taskId=ubf6d1414-73eb-47a9-84a7-189a3ccd64f&title=&width=301)![{98037a6a-8fc4-4a97-8ceb-d5761fe438a6}.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666344223464-eec3a10a-9a48-463e-b90d-d5d2bf47c6a2.png#clientId=u9dc2b72d-e312-4&errorMessage=unknown%20error&from=paste&height=270&id=u14175635&name=%7B98037a6a-8fc4-4a97-8ceb-d5761fe438a6%7D.png&originHeight=270&originWidth=876&originalType=binary&ratio=1&rotation=0&showTitle=false&size=43374&status=error&style=none&taskId=u1e5014ab-4703-4b2b-9a61-9a73f2ffa00&title=&width=876)
## RESTful API
 	REST与技术无关，代表的是一种软件架构风格，REST是Representational State Transfer的简称，中文翻译为“表征状态转移”或“表现层状态转化”。  
推荐阅读[阮一峰 理解RESTFul架构](http://www.ruanyifeng.com/blog/2011/09/restful.html)
简单来说，REST的含义就是客户端与Web服务器之间进行交互的时候，使用HTTP协议中的4个请求方法代表不同的动作。

- GET用来获取资源
- POST用来新建资源
- PUT用来更新资源
- DELETE用来删除资源。

只要API程序遵循了REST风格，那就可以称其为RESTful API。目前在前后端分离的架构中，前后端基本都是通过RESTful API来进行交互。
例如，我们现在要编写一个管理书籍的系统，我们可以查询对一本书进行查询、创建、更新和删除等操作，我们在编写程序的时候就要设计客户端浏览器与我们Web服务端交互的方式和路径。按照经验我们通常会设计成如下模式：

| 请求方法 | URL | 含义 |
| --- | --- | --- |
| GET | /book | 查询书籍信息 |
| POST | /create_book | 创建书籍信息 |
| POST | /update_book | 更新书籍信息 |
| POST | /detete_book | 删除书籍信息 |

RESTFul API设计

| 请求方法 | URL | 含义 |
| --- | --- | --- |
| GET | /book | 查询书籍信息 |
| POST | /book | 创建书籍信息 |
| PUT | /book | 更新书籍信息 |
| DELETE | /book | 删除书籍信息 |

Gin框架支持开发RESTful API的开发。  
```go
func main() {
	r := gin.Default()
	r.GET("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "GET",
		})
	})

	r.POST("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "POST",
		})
	})

	r.PUT("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "PUT",
		})
	})

	r.DELETE("/book", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "DELETE",
		})
	})
}

```
## Gin渲染
### HTML渲染
 	定义一个存放模板文件的templates文件夹，然后在其内部按照业务分别定义一个posts文件夹和一个users文件夹。 posts/index.html文件的内容如下：  
```html
{{define "posts/index.html"}}
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>posts/index</title>
</head>
<body>
    {{.title}}
</body>
</html>
{{end}}
```
```html
{{define "users/index.html"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>users/index</title>
</head>
<body>
    {{.title}}
</body>
</html>
{{end}}
```
 	Gin框架中使用LoadHTMLGlob()或者LoadHTMLFiles()方法进行HTML模板渲染。  
```go
func main() {
    r := gin.Default()
    r.LoadHTMLGlob("templates/**/*")
    //r.LoadHTMLFiles("templates/posts/index.html", "templates/users/index.html")
    r.GET("/posts/index", func(c *gin.Context) {
        c.HTML(http.StatusOK, "posts/index.html", gin.H{
            "title": "posts/index",
        })
    })

    r.GET("users/index", func(c *gin.Context) {
        c.HTML(http.StatusOK, "users/index.html", gin.H{
            "title": "users/index",
        })
    })

    r.Run(":8080")
}
```
### JSON渲染
```go
func main() {
	r := gin.Default()

	// gin.H 是map[string]interface{}的缩写
	r.GET("/someJSON", func(c *gin.Context) {
		// 方式一：自己拼接JSON
		c.JSON(http.StatusOK, gin.H{"message": "Hello world!"})
	})
	r.GET("/moreJSON", func(c *gin.Context) {
		// 方法二：使用结构体
		var msg struct {
			Name    string `json:"user"`
			Message string
			Age     int
		}
		msg.Name = "James"
		msg.Message = "Hello World"
		msg.Age = 38
		c.JSON(http.StatusOK, msg)
	})
	r.Run(":8080")
}
```
### XML渲染
```go
func main() {
	r := gin.Default()
	// gin.H 是map[string]interface{}的缩写
	r.GET("/someXML", func(c *gin.Context) {
		// 方式一：自己拼接JSON
		c.XML(http.StatusOK, gin.H{"message": "Hello world!"})
	})
	r.GET("/moreXML", func(c *gin.Context) {
		// 方法二：使用结构体
		type MessageRecord struct {
			Name    string
			Message string
			Age     int
		}
		var msg MessageRecord
		msg.Name = "小王子"
		msg.Message = "Hello world!"
		msg.Age = 18
		c.XML(http.StatusOK, msg)
	})
	r.Run(":8080")
}
```
### YMAL渲染
```go
r.GET("/someYAML", func(c *gin.Context) {
	c.YAML(http.StatusOK, gin.H{"message": "ok", "status": http.StatusOK})
})

```
### Protobuf渲染
```go
r.GET("/someProtoBuf", func(c *gin.Context) {
	reps := []int64{int64(1), int64(2)}
	label := "test"
	// protobuf 的具体定义写在 testdata/protoexample 文件中。
	data := &protoexample.Test{
		Label: &label,
		Reps:  reps,
	}
	// 请注意，数据在响应中变为二进制数据
	// 将输出被 protoexample.Test protobuf 序列化了的数据
	c.ProtoBuf(http.StatusOK, data)
})
```
## 获取参数
### 获取querystring参数
querystring指的是URL中?后面携带的参数，例如：/user/search?username=***&address=***。 获取请求的querystring参数的方法如下：  
```go
func main() {
	//Default返回一个默认的路由引擎
	r := gin.Default()
	r.GET("/user/search", func(c *gin.Context) {
		username := c.Query("username")
		address := c.Query("address")
		//输出json结果给调用方
		c.JSON(http.StatusOK, gin.H{
			"message":  "ok",
			"username": username,
			"address":  address,
		})
	})
	r.Run()
}
```
### 获取form参数
 	当前端请求的数据通过form表单提交时，例如向/user/search发送一个POST请求，获取请求数据的方式如下：  
```go
func main() {
	//Default返回一个默认的路由引擎
	r := gin.Default()
	r.POST("/user/search", func(c *gin.Context) {
		// DefaultPostForm取不到值时会返回指定的默认值
		username := c.DefaultPostForm("username", "James")
		//username := c.PostForm("username")
		address := c.PostForm("address")
		//输出json结果给调用方
		c.JSON(http.StatusOK, gin.H{
			"message":  "ok",
			"username": username,
			"address":  address,
		})
	})
	r.Run(":8080")
}

```
### 获取json参数
```go
r.POST("/json", func(c *gin.Context) {
	// 注意：下面为了举例子方便，暂时忽略了错误处理
	b, _ := c.GetRawData()  // 从c.Request.Body读取请求数据
	// 定义map或结构体
	var m map[string]interface{}
	// 反序列化
	_ = json.Unmarshal(b, &m)

	c.JSON(http.StatusOK, m)
})
```
### 获取path参数
请求的参数通过URL路径传递，例如：/user/search/小王/北京。 获取请求URL路径中的参数的方式如下
```go
func main() {
	//Default返回一个默认的路由引擎
	r := gin.Default()
	r.GET("/user/search/:username/:address", func(c *gin.Context) {
		username := c.Param("username")
		address := c.Param("address")
		//输出json结果给调用方
		c.JSON(http.StatusOK, gin.H{
			"message":  "ok",
			"username": username,
			"address":  address,
		})
	})

	r.Run(":8080")
}

```
## 文件上传
### 前端页面代码
```html
<!DOCTYPE html>
<html lang="zh-CN">
  <head>
    <title>上传文件示例</title>
  </head>
  <body>
    <form action="/upload" method="post" enctype="multipart/form-data">
      <input type="file" name="f1">
      <input type="submit" value="上传">
    </form>
  </body>
</html>
```
### 后端gin框架部分代码
```go
func main() {
	router := gin.Default()
	// 处理multipart forms提交文件时默认的内存限制是32 MiB
	// 可以通过下面的方式修改
	// router.MaxMultipartMemory = 8 << 20  // 8 MiB
	router.POST("/upload", func(c *gin.Context) {
		// 单个文件
		file, err := c.FormFile("f1")
		if err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{
				"message": err.Error(),
			})
			return
		}

		log.Println(file.Filename)
		dst := fmt.Sprintf("C:/tmp/%s", file.Filename)
		// 上传文件到指定的目录
		c.SaveUploadedFile(file, dst)
		c.JSON(http.StatusOK, gin.H{
			"message": fmt.Sprintf("'%s' uploaded!", file.Filename),
		})
	})
	router.Run()
}

```
### 多文件上传
```go
func main() {
	router := gin.Default()
	// 处理multipart forms提交文件时默认的内存限制是32 MiB
	// 可以通过下面的方式修改
	// router.MaxMultipartMemory = 8 << 20  // 8 MiB
	router.POST("/upload", func(c *gin.Context) {
		// Multipart form
		form, _ := c.MultipartForm()
		files := form.File["file"]

		for index, file := range files {
			log.Println(file.Filename)
			dst := fmt.Sprintf("C:/tmp/%s_%d", file.Filename, index)
			// 上传文件到指定的目录
			c.SaveUploadedFile(file, dst)
		}
		c.JSON(http.StatusOK, gin.H{
			"message": fmt.Sprintf("%d files uploaded!", len(files)),
		})
	})
	router.Run()
}
```
## 重定向
### HTTP重定向
```go
r.GET("/test", func(c *gin.Context) {
	c.Redirect(http.StatusMovedPermanently, "http://www.baidu.com/")
})
```
### 路由重定向
```go
r.GET("/test", func(c *gin.Context) {
    // 指定重定向的URL
    c.Request.URL.Path = "/test2"
    r.HandleContext(c)
})
r.GET("/test2", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"hello": "world"})
})
```
## 路由组
```go
func main() {
	r := gin.Default()
	userGroup := r.Group("/user")
	{
		userGroup.GET("/index", func(c *gin.Context) {...})
		userGroup.GET("/login", func(c *gin.Context) {...})
		userGroup.POST("/login", func(c *gin.Context) {...})

	}
	shopGroup := r.Group("/shop")
	{
		shopGroup.GET("/index", func(c *gin.Context) {...})
		shopGroup.GET("/cart", func(c *gin.Context) {...})
		shopGroup.POST("/checkout", func(c *gin.Context) {...})
	}
	r.Run()
}

// 嵌套路由组
shopGroup := r.Group("/shop")
	{
		shopGroup.GET("/index", func(c *gin.Context) {...})
		shopGroup.GET("/cart", func(c *gin.Context) {...})
		shopGroup.POST("/checkout", func(c *gin.Context) {...})
		// 嵌套路由组
		xx := shopGroup.Group("xx")
		xx.GET("/oo", func(c *gin.Context) {...})
	}
```

## 源码解析
### context
```go
// Context作为一个数据结构在中间件中传递本次请求的各种数据、管理流程，进行响应
// context.go:40
type Context struct {
    // ServeHTTP的第二个参数: request
    Request   *http.Request

    // 用来响应 
    Writer    ResponseWriter
    writermem responseWriter

    // URL里面的参数，比如：/xx/:id  
    Params   Params
    // 参与的处理者（中间件 + 请求处理者列表）
    handlers HandlersChain
    // 当前处理到的handler的下标
    index    int8

    // Engine单例
    engine *Engine

    // 在context可以设置的值
    Keys map[string]interface{}

    // 一系列的错误
    Errors errorMsgs

    // Accepted defines a list of manually accepted formats for content negotiation.
    Accepted []string
}

// response_writer.go:20
type ResponseWriter interface {
    http.ResponseWriter //嵌入接口
    http.Hijacker       //嵌入接口
    http.Flusher        //嵌入接口
    http.CloseNotifier  //嵌入接口

    // 返回当前请求的 response status code
    Status() int

    // 返回写入 http body的字节数
    Size() int

    // 写string
    WriteString(string) (int, error)

    //是否写出
    Written() bool

    // 强制写htp header (状态码 + headers)
    WriteHeaderNow()
}

// response_writer.go:40
// 实现 ResponseWriter 接口
type responseWriter struct {
    http.ResponseWriter
    size   int
    status int
}


type errorMsgs []*Error


// 每当一个请求来到服务器，都会从对象池中拿到一个context。其函数有：
// **** 创建
reset()                 //从对象池中拿出来后需要初始化
Copy() *Context         //克隆，用于goroute中
HandlerName() string    //得到最后那个处理者的名字
Handler()               //得到最后那个Handler

// **** 流程控制
Next()                  // 只能在中间件中使用，依次调用各个处理者
IsAborted() bool    
Abort()                 // 废弃
AbortWithStatusJson(code int, jsonObj interface{})
AbortWithError(code int, err error) *Error

// **** 错误管理
Error(err error) *Error // 给本次请求添加个错误。将错误收集然后用中间件统一处理（打日志|入库）是一个比较好的方案

// **** 元数据管理
Set(key string, value interface{})  //本次请求用户设置各种数据 (Keys 字段)
Get(key string)(value interface{}, existed bool)
MustGet(key string)(value interface{})
GetString(key string) string
GetBool(key string) bool
GetInt(key string) int
GetInt64(key string) int64
GetFloat64(key string) float64
GetTime(key string) time.Time
GetDuration(key string) time.Duration
GetStringSlice(key string) []string
GetStringMap(key string) map[string]interface{}
GetStringMapString(key string) map[string]string
GetStringMapStringSlice(key string) map[string][]string

// **** 输入数据
//从URL中拿值，比如 /user/:id => /user/john
Param(key string) string    

//从GET参数中拿值，比如 /path?id=john
GetQueryArray(key string) ([]string, bool)  
GetQuery(key string)(string, bool)
Query(key string) string
DefaultQuery(key, defaultValue string) string
GetQueryArray(key string) ([]string, bool)
QueryArray(key string) []string

//从POST中拿数据
GetPostFormArray(key string) ([]string, bool)
PostFormArray(key string) []string 
GetPostForm(key string) (string, bool)
PostForm(key string) string
DefaultPostForm(key, defaultValue string) string

// 文件
FormFile(name string) (*multipart.FileHeader, error)
MultipartForm() (*multipart.Form, error)
SaveUploadedFile(file *multipart.FileHeader, dst string) error

// 数据绑定
Bind(obj interface{}) error //根据Content-Type绑定数据
BindJSON(obj interface{}) error
BindQuery(obj interface{}) error

//--- Should ok, else return error
ShouldBindJSON(obj interface{}) error 
ShouldBind(obj interface{}) error
ShouldBindJSON(obj interface{}) error
ShouldBindQuery(obj interface{}) error

//--- Must ok, else SetError
MustBindJSON(obj interface{}) error 

ClientIP() string
ContentType() string
IsWebsocket() bool

// **** 输出数据
Status(code int)            // 设置response code
Header(key, value string)   // 设置header
GetHeader(key string) string

GetRawData() ([]byte, error)

Cookie(name string) (string, error)     // 设置cookie
SetCookie(name, value string, maxAge int, path, domain string, secure, httpOnly bool)

Render(code int, r render.Render)      // 数据渲染
HTML(code int, name string, obj interface{})    //HTML
JSON(code int, obj interface{})                 //JSON
IndentedJSON(code int, obj interface{})
SecureJSON(code int, obj interface{})
JSONP(code int, obj interface{})                //jsonp
XML(code int, obj interface{})                  //XML
YAML(code int, obj interface{})                 //YAML
String(code int, format string, values ...interface{})  //string
Redirect(code int, location string)             // 重定向
Data(code int, contentType string, data []byte) // []byte
File(filepath string)                           // file
SSEvent(name string, message interface{})       // Server-Sent Event
Stream(step func(w io.Writer) bool)             // stream
```
#### 扩展

- [官方文档](https://gin-gonic.com/zh-cn/docs/examples/)
- [源码](https://github.com/gin-gonic/gin/blob/master/context.go)
### Radix Tree
gin框架使用的是定制版本的[httprouter](https://github.com/julienschmidt/httprouter)，其路由的原理是大量使用公共前缀的树结构，它基本上是一个紧凑的[Trie tree](https://baike.sogou.com/v66237892.htm)（或者只是[Radix Tree](https://baike.sogou.com/v73626121.htm)）。具有公共前缀的节点也共享一个公共父节点。
基数树（Radix Tree）又称为PAT位树（Patricia Trie or crit bit tree），是一种更节省空间的前缀树（Trie Tree）。对于基数树的每个节点，如果该节点是唯一的子树的话，就和父节点合并。下图为一个基数树示例：
![radix_tree.png](https://cdn.nlark.com/yuque/0/2022/png/1607733/1666346027966-8fc6e299-cdc6-4af4-b1c9-85ab4f4269a9.png#clientId=u9dc2b72d-e312-4&errorMessage=unknown%20error&from=paste&height=500&id=u55b9818b&name=radix_tree.png&originHeight=500&originWidth=800&originalType=binary&ratio=1&rotation=0&showTitle=false&size=19464&status=error&style=none&taskId=u868202dc-3c10-477a-a664-b3518993cf7&title=&width=800)
### 路由树节点
```go
// serarch support
// s earch upport
type node struct {
   // 节点路径，比如上面的s，earch，和upport
	path      string
	// 和children字段对应, 保存的是分裂的分支的第一个字符
	// 例如search和support, 那么s节点的indices对应的"eu"
	// 代表有两个分支, 分支的首字母分别是e和u
	indices   string
	// 儿子节点
	children  []*node
	// 处理函数链条（切片）
	handlers  HandlersChain
	// 优先级，子节点、子子节点等注册的handler数量
	priority  uint32
	// 节点类型，包括static, root, param, catchAll
	// static: 静态节点（默认），比如上面的s，earch等节点
	// root: 树的根节点
	// catchAll: 有*匹配的节点
	// param: 参数节点
	nType     nodeType
	// 路径上最大参数个数
	maxParams uint8
	// 节点是否是参数节点
	wildChild bool
	// 完整路径
	fullPath  string
}
```
### 请求方法树
 	在gin的路由中，每一个HTTP Method(GET、POST、PUT、DELETE…)都对应了一棵 radix tree，我们注册路由的时候会调用下面的addRoute函数：  
```go
// gin.go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}

func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
   // 获取请求方法对应的树
	root := engine.trees.get(method)
	if root == nil {
	
	   // 如果没有就创建一个
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
	root.addRoute(path, handlers)
}
```
```go
type methodTree struct {
	method string
	root   *node
}

type methodTrees []methodTree  // slice
```
```go
func (trees methodTrees) get(method string) *node {
	for _, tree := range trees {
		if tree.method == method {
			return tree.root
		}
	}
	return nil
}
```
**出于内存考虑使用切片而不是使用map来存储 请求方法->树 的结构,HTTP请求方法的数量是固定的，而且常用的只有几种**
```go
func New() *Engine {
	debugPrintWARNINGNew()
	engine := &Engine{
		RouterGroup: RouterGroup{
			Handlers: nil,
			basePath: "/",
			root:     true,
		},
		// 初始化容量为9的切片（HTTP1.1请求方法共9种）
		trees: make(methodTrees, 0, 9),
	}
	engine.RouterGroup.engine = engine
	engine.pool.New = func() interface{} {
		return engine.allocateContext()
	}
	return engine
}

```
### 注册路由
 	注册路由的逻辑主要有addRoute函数和insertChild方法。  
#### addRoute
```go
// tree.go
// addRoute 将具有给定句柄的节点添加到路径中。
// 不是并发安全的
func (n *node) addRoute(path string, handlers HandlersChain) {
	fullPath := path
	n.priority++
	numParams := countParams(path)  // 数一下参数个数

	// 空树就直接插入当前节点
	if len(n.path) == 0 && len(n.children) == 0 {
		n.insertChild(numParams, path, fullPath, handlers)
		n.nType = root
		return
	}

	parentFullPathIndex := 0

walk:
	for {
		// 更新当前节点的最大参数个数
		if numParams > n.maxParams {
			n.maxParams = numParams
		}

		// 找到最长的通用前缀
		// 这也意味着公共前缀不包含“:”"或“*” /
		// 因为现有键不能包含这些字符。
		i := longestCommonPrefix(path, n.path)

		// 分裂边缘（此处分裂的是当前树节点）
		// 例如一开始path是search，新加入support，s是他们通用的最长前缀部分
		// 那么会将s拿出来作为parent节点，增加earch和upport作为child节点
		if i < len(n.path) {
			child := node{
				path:      n.path[i:],  // 公共前缀后的部分作为子节点
				wildChild: n.wildChild,
				indices:   n.indices,
				children:  n.children,
				handlers:  n.handlers,
				priority:  n.priority - 1, //子节点优先级-1
				fullPath:  n.fullPath,
			}

			// Update maxParams (max of all children)
			for _, v := range child.children {
				if v.maxParams > child.maxParams {
					child.maxParams = v.maxParams
				}
			}

			n.children = []*node{&child}
			// []byte for proper unicode char conversion, see #65
			n.indices = string([]byte{n.path[i]})
			n.path = path[:i]
			n.handlers = nil
			n.wildChild = false
			n.fullPath = fullPath[:parentFullPathIndex+i]
		}

		// 将新来的节点插入新的parent节点作为子节点
		if i < len(path) {
			path = path[i:]

			if n.wildChild {  // 如果是参数节点
				parentFullPathIndex += len(n.path)
				n = n.children[0]
				n.priority++

				// Update maxParams of the child node
				if numParams > n.maxParams {
					n.maxParams = numParams
				}
				numParams--

				// 检查通配符是否匹配
				if len(path) >= len(n.path) && n.path == path[:len(n.path)] {
					// 检查更长的通配符, 例如 :name and :names
					if len(n.path) >= len(path) || path[len(n.path)] == '/' {
						continue walk
					}
				}

				pathSeg := path
				if n.nType != catchAll {
					pathSeg = strings.SplitN(path, "/", 2)[0]
				}
				prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
				panic("'" + pathSeg +
					"' in new path '" + fullPath +
					"' conflicts with existing wildcard '" + n.path +
					"' in existing prefix '" + prefix +
					"'")
			}
			// 取path首字母，用来与indices做比较
			c := path[0]

			// 处理参数后加斜线情况
			if n.nType == param && c == '/' && len(n.children) == 1 {
				parentFullPathIndex += len(n.path)
				n = n.children[0]
				n.priority++
				continue walk
			}

			// 检查path下一个字节的子节点是否存在
			// 比如s的子节点现在是earch和upport，indices为eu
			// 如果新加一个路由为super，那么就是和upport有匹配的部分u，将继续分列现在的upport节点
			for i, max := 0, len(n.indices); i < max; i++ {
				if c == n.indices[i] {
					parentFullPathIndex += len(n.path)
					i = n.incrementChildPrio(i)
					n = n.children[i]
					continue walk
				}
			}

			// 否则就插入
			if c != ':' && c != '*' {
				// []byte for proper unicode char conversion, see #65
				// 注意这里是直接拼接第一个字符到n.indices
				n.indices += string([]byte{c})
				child := &node{
					maxParams: numParams,
					fullPath:  fullPath,
				}
				// 追加子节点
				n.children = append(n.children, child)
				n.incrementChildPrio(len(n.indices) - 1)
				n = child
			}
			n.insertChild(numParams, path, fullPath, handlers)
			return
		}

		// 已经注册过的节点
		if n.handlers != nil {
			panic("handlers are already registered for path '" + fullPath + "'")
		}
		n.handlers = handlers
		return
	}
}
```

1. 如果树上不存在任何节点，则把当前节点作为根节点，插入到方法树上，节点路径为传入路径。
2. 否则，遍历树的节点：
   1. 计算当前节点的路径和传入路径的最长公共前缀位置。（比如已存在一个节点路径为 /search/，传入路径 /support/ 与它的最长公共前缀位置是 2，意味着它们具有共同前缀/s）
   2. 如果公共前缀的长度小于当前节点的长度（/s 长度小于 /search ），则在当前节点产生分裂，生成一个路径为 earch/ 的子节点，把它添加到当前节点的 children，并把首字符 e 添加到当前节点的前缀索引 indices 中，将当前节点的路径改为前缀路径（从 /search/ 变为 /s）。
   3. 如果公共前缀的长度小于传入节点的长度（/s 长度小于 /support ），则在传入路径中产生一个新的路径（upport/），插入到当前节点的 children，把首字符 u 添加到当前节点的前缀索引 indices 中。
      1. 这里存在一种情况是，如果当前节点的子节点是一个参数节点（当前节点的wildChild 为 true），那么会检查传入路径是否也是相同的参数节点下的路径，比如当前节点路径为 /user/:user_id，传入节点路径为 /user/:user_id/name，如果满足条件的话，则继续到子节点（:user_id) 下创建新的路径，否则若在参数节点下定义了其他路径，如/user/name，则会直接发生 panic 返回，因为当前路径下存在冲突（一个参数节点不能跟一个非参数节点位于同级）。
      2. 如果当前节点是一个参数节点，（如 :user_id，在此节点下创建路径为 /name），并且路径以 / 开头且当前节点只存在一个子节点，则当前节点指向子节点，继续进行路径分裂。
      3. 如果当前节点存在多个子节点，则从 indices 查找匹配路径首字符的子节点，继续往子节点遍历。
      4. 否则直接往当前节点上创建子节点。（例如：定义路由为 /user/:user_id/，则 :user_id 会存在一个子节点为 /，这时候 /name 就需要跟 / 节点进行路径分裂插入到 :user_id 下，如果定义路由为 /user/:user_id，则直接插入到 :user_id 下就好了）
#### insertChild
```go
// tree.go
func (n *node) insertChild(numParams uint8, path string, fullPath string, handlers HandlersChain) {
  // 找到所有的参数
	for numParams > 0 {
		// 查找前缀直到第一个通配符
		wildcard, i, valid := findWildcard(path)
		if i < 0 { // 没有发现通配符
			break
		}

		// 通配符的名称必须包含':' 和 '*'
		if !valid {
			panic("only one wildcard per path segment is allowed, has: '" +
				wildcard + "' in path '" + fullPath + "'")
		}

		// 检查通配符是否有名称
		if len(wildcard) < 2 {
			panic("wildcards must be named with a non-empty name in path '" + fullPath + "'")
		}

		// 检查这个节点是否有已经存在的子节点
		// 如果我们在这里插入通配符，这些子节点将无法访问
		if len(n.children) > 0 {
			panic("wildcard segment '" + wildcard +
				"' conflicts with existing children in path '" + fullPath + "'")
		}

		if wildcard[0] == ':' { // param
			if i > 0 {
				// 在当前通配符之前插入前缀
				n.path = path[:i]
				path = path[i:]
			}

			n.wildChild = true
			child := &node{
				nType:     param,
				path:      wildcard,
				maxParams: numParams,
				fullPath:  fullPath,
			}
			n.children = []*node{child}
			n = child
			n.priority++
			numParams--

			// 如果路径没有以通配符结束
			// 那么将有另一个以'/'开始的非通配符子路径。
			if len(wildcard) < len(path) {
				path = path[len(wildcard):]

				child := &node{
					maxParams: numParams,
					priority:  1,
					fullPath:  fullPath,
				}
				n.children = []*node{child}
				n = child  // 继续下一轮循环
				continue
			}

			// 否则我们就完成了。将处理函数插入新叶子中
			n.handlers = handlers
			return
		}

		// catchAll
		if i+len(wildcard) != len(path) || numParams > 1 {
			panic("catch-all routes are only allowed at the end of the path in path '" + fullPath + "'")
		}

		if len(n.path) > 0 && n.path[len(n.path)-1] == '/' {
			panic("catch-all conflicts with existing handle for the path segment root in path '" + fullPath + "'")
		}

		// currently fixed width 1 for '/'
		i--
		if path[i] != '/' {
			panic("no / before catch-all in path '" + fullPath + "'")
		}

		n.path = path[:i]
		
		// 第一个节点:路径为空的catchAll节点
		child := &node{
			wildChild: true,
			nType:     catchAll,
			maxParams: 1,
			fullPath:  fullPath,
		}
		// 更新父节点的maxParams
		if n.maxParams < 1 {
			n.maxParams = 1
		}
		n.children = []*node{child}
		n.indices = string('/')
		n = child
		n.priority++

		// 第二个节点:保存变量的节点
		child = &node{
			path:      path[i:],
			nType:     catchAll,
			maxParams: 1,
			handlers:  handlers,
			priority:  1,
			fullPath:  fullPath,
		}
		n.children = []*node{child}

		return
	}

	// 如果没有找到通配符，只需插入路径和句柄
	n.path = path
	n.handlers = handlers
	n.fullPath = fullPath
}

// Search for a wildcard segment and check the name for invalid characters.
// Returns -1 as index, if no wildcard was found.
func findWildcard(path string) (wildcard string, i int, valid bool) {
	// Find start
	for start, c := range []byte(path) {
		// A wildcard starts with ':' (param) or '*' (catch-all)
		if c != ':' && c != '*' {
			continue
		}

		// Find end and check for invalid characters
		valid = true
		for end, c := range []byte(path[start+1:]) {
			switch c {
			case '/':
				return path[start : start+1+end], start, valid
			case ':', '*':
				valid = false
			}
		}
		return path[start:], start, valid
	}
	return "", -1, false
}
```
insertChild函数是根据path本身进行分割，将/分开的部分分别作为节点保存，形成一棵树结构。参数匹配中的 **: **和** * **的区别是，前者是匹配一个字段而后者是匹配后面所有的路径。  
### 路由匹配
```go
// gin.go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  // 这里使用了对象池
	c := engine.pool.Get().(*Context)
  // 这里有一个细节就是Get对象后做初始化
	c.writermem.reset(w)
	c.Request = req
	c.reset()
	engine.handleHTTPRequest(c)  // 我们要找的处理HTTP请求的函数
	engine.pool.Put(c)  // 处理完请求后将对象放回池子
}
```
```go
// gin.go
func (engine *Engine) handleHTTPRequest(c *Context) {
	// 根据请求方法找到对应的路由树
	t := engine.trees
	for i, tl := 0, len(t); i < tl; i++ {
		if t[i].method != httpMethod {
			continue
		}
		root := t[i].root
		// 在路由树中根据path查找
		value := root.getValue(rPath, c.Params, unescape)
		if value.handlers != nil {
			c.handlers = value.handlers
			c.Params = value.params
			c.fullPath = value.fullPath
			c.Next()  // 执行函数链条
			c.writermem.WriteHeaderNow()
			return
		}
	c.handlers = engine.allNoRoute
	serveError(c, http.StatusNotFound, default404Body)
}

```
路由匹配是由节点的 getValue方法实现的。getValue根据给定的路径(键)返回nodeValue值，保存注册的处理函数和匹配到的路径参数数据。
如果找不到任何处理函数，则会尝试TSR(尾随斜杠重定向)。
```go
// tree.go
type nodeValue struct {
	handlers HandlersChain
	params   Params  // []Param
	tsr      bool
	fullPath string
}

func (n *node) getValue(path string, po Params, unescape bool) (value nodeValue) {
	value.params = po
walk: // Outer loop for walking the tree
	for {
		prefix := n.path
		if path == prefix {
			// 我们应该已经到达包含处理函数的节点。
			// 检查该节点是否注册有处理函数
			if value.handlers = n.handlers; value.handlers != nil {
				value.fullPath = n.fullPath
				return
			}

			if path == "/" && n.wildChild && n.nType != root {
				value.tsr = true
				return
			}

			// 没有找到处理函数 检查这个路径末尾+/ 是否存在注册函数
			indices := n.indices
			for i, max := 0, len(indices); i < max; i++ {
				if indices[i] == '/' {
					n = n.children[i]
					value.tsr = (len(n.path) == 1 && n.handlers != nil) ||
						(n.nType == catchAll && n.children[0].handlers != nil)
					return
				}
			}

			return
		}

		if len(path) > len(prefix) && path[:len(prefix)] == prefix {
			path = path[len(prefix):]
			// 如果该节点没有通配符(param或catchAll)子节点
			// 我们可以继续查找下一个子节点
			if !n.wildChild {
				c := path[0]
				indices := n.indices
				for i, max := 0, len(indices); i < max; i++ {
					if c == indices[i] {
						n = n.children[i] // 遍历树
						continue walk
					}
				}

				// 没找到
				// 如果存在一个相同的URL但没有末尾/的叶子节点
				// 我们可以建议重定向到那里
				value.tsr = path == "/" && n.handlers != nil
				return
			}

			// 根据节点类型处理通配符子节点
			n = n.children[0]
			switch n.nType {
			case param:
				// find param end (either '/' or path end)
				end := 0
				for end < len(path) && path[end] != '/' {
					end++
				}

				// 保存通配符的值
				if cap(value.params) < int(n.maxParams) {
					value.params = make(Params, 0, n.maxParams)
				}
				i := len(value.params)
				value.params = value.params[:i+1] // 在预先分配的容量内扩展slice
				value.params[i].Key = n.path[1:]
				val := path[:end]
				if unescape {
					var err error
					if value.params[i].Value, err = url.QueryUnescape(val); err != nil {
						value.params[i].Value = val // fallback, in case of error
					}
				} else {
					value.params[i].Value = val
				}

				// 继续向下查询
				if end < len(path) {
					if len(n.children) > 0 {
						path = path[end:]
						n = n.children[0]
						continue walk
					}

					// ... but we can't
					value.tsr = len(path) == end+1
					return
				}

				if value.handlers = n.handlers; value.handlers != nil {
					value.fullPath = n.fullPath
					return
				}
				if len(n.children) == 1 {
					// 没有找到处理函数. 检查此路径末尾加/的路由是否存在注册函数
					// 用于 TSR 推荐
					n = n.children[0]
					value.tsr = n.path == "/" && n.handlers != nil
				}
				return

			case catchAll:
				// 保存通配符的值
				if cap(value.params) < int(n.maxParams) {
					value.params = make(Params, 0, n.maxParams)
				}
				i := len(value.params)
				value.params = value.params[:i+1] // 在预先分配的容量内扩展slice
				value.params[i].Key = n.path[2:]
				if unescape {
					var err error
					if value.params[i].Value, err = url.QueryUnescape(path); err != nil {
						value.params[i].Value = path // fallback, in case of error
					}
				} else {
					value.params[i].Value = path
				}

				value.handlers = n.handlers
				value.fullPath = n.fullPath
				return

			default:
				panic("invalid node type")
			}
		}

		// 找不到，如果存在一个在当前路径最后添加/的路由
		// 我们会建议重定向到那里
		value.tsr = (path == "/") ||
			(len(prefix) == len(path)+1 && prefix[len(path)] == '/' &&
				path == prefix[:len(prefix)-1] && n.handlers != nil)
		return
	}
}

```

1. 如果当前节点路径与传入路径相等
   1. 如果当前节点的处理函数不为空，结束并返回路由信息。
   2. 如果当前节点不存在处理函数，则尝试寻找路径+'/'上是否注册了处理函数，如果有则尝试进行重定向。
2. 如果当前节点路径与传入路径具有相同前缀
   1. 如果当前节点不存在一个带参数的子节点，则查找并遍历到下一个匹配的子节点。
   2. 否则如果当前节点存在一个带参数的子节点，则解析请求参数并记录到返回值的 params 里，如果路径后还有子路径（如：/user/123/name），则继续尝试匹配当前节点下的子节点，直至完全匹配返回。
3. 如果当前节点路径与传入路径不匹配，则尝试去寻找路径+"/"的节点是否存在，不存在则返回 HTTP 404 或 405 错误。
### 基数树构建
```go
// 最初的数据时：engine.roots = nil
// step1: engine.Use(f1)
// step2：添加{method: POST, path: /p1, handler:f2}
/p1

node_/p1 = {
    path:"/p1"
    indices:""
    handlers:[f1, f2]
    priority:1
    nType:root
    maxParams:0
    wildChild:false
}
engine.roots = [{
    method: POST,
    root: node_/p1
}]

// step3：添加{method: POST, path: /p, handler:f3}
/p
/p1

node_/p = {
    path:"/p"
    indices:"1"
    handlers:[f1, f3]
    priority:2
    nType:root
    maxParams:0
    wildChild:false
    children: [
        {
            path:"1"
            indices:""
            children:[]
            handlers: [f1, f2]
            priority:1
            nType:static
            maxParams:0
            wildChild:false
        }
    ]
}

engine.roots = [{
    method: POST,
    root: node_/p
}]

// step4：添加{method: POST, path: /, handler:f4}
/
/p
/p1

node_/ = {
    path:"/"
    indices:"p"
    handlers:[f1, f4]
    priority:3
    nType:root
    maxParams:0
    wildChild:false
    children:[
        {
            path:"p"
            indices:"1"
            handlers:[f1, f3]
            priority:2
            nType:static
            maxParams:0
            wildChild:false
            children:[
                {
                    path:"1"
                    indices:""
                    children:[]
                    handlers:[f1, f2]
                    priority:1
                    nType:static
                    maxParams:0
                    wildChild:false
                }
            ]
        }
    ]
}

engine.roots = [{
    method: POST,
    root: node_/
}]

// step5：添加{method: POST, path: /p1/p34, handler:f5}
/
/p
/p1
/p1/p34

node_/ = {
    path:"/"
    indices:"p"
    handlers:[f1, f4]
    priority:4
    nType:root
    maxParams:0
    wildChild:false
    children:[
        {
            path:"p"
            indices:"1"
            handlers:[f1, f3]
            priority:3
            nType:static
            maxParams:0
            wildChild:false
            children:[
                {
                    path:"1"
                    indices:""
                    handlers:[f1, f2]
                    priority:2
                    nType:static
                    maxParams:0
                    wildChild:false
                    children:[
                        {
                            path:"/p34"
                            indices:""
                            handlers:[f1, f5]
                            priority:1
                            nType:static
                            maxParams:0
                            wildChild:false
                            children:[]
                        }
                    ]
                }
            ]
        }
    ]
}

engine.roots = [{
    method: POST,
    root: node_/
}]

// step6：添加{method: POST, path: /p12, handler:f6}
/
/p
/p1
/p1/p34
/p12

node_/ = {
    path:"/"
    indices:"p"
    handlers:[f1, f4]
    priority:5
    nType:root
    maxParams:0
    wildChild:false
    children:[
        {
            path:"p"
            indices:"1"
            handlers:[f1, f3]
            priority:4
            nType:static
            maxParams:0
            wildChild:false
            children:[
                {
                    path:"1"
                    indices:"/2"
                    handlers:[f1, f2]
                    priority:3
                    nType:static
                    maxParams:0
                    wildChild:false
                    children:[
                        {
                            path:"/p34"
                            indices:""
                            handlers:[f1, f5]
                            priority:1
                            nType:static
                            maxParams:0
                            wildChild:false
                            children:[]
                        }
                        {
                            path:"2"
                            indices:""
                            handlers:[f1, f6]
                            priority:1
                            nType:static
                            maxParams:0
                            wildChild:false
                            children:[]
                        }
                    ]
                }
            ]
        }
    ]
}

// step7：添加{method: POST, path: /p12/p56, handler:f7}
/
/p
/p1
/p1/p34
/p12
/p12/p56

node_/ = {
    path:"/"
    indices:"p"
    handlers:[f1, f4]
    priority:5 + 1
    nType:root
    maxParams:0
    wildChild:false
    children:[
        {
            path:"p"
            indices:"1"
            handlers:[f1, f3]
            priority:4 + 1
            nType:static
            maxParams:0
            wildChild:false
            children:[
                {
                    path:"1"
                    indices:"2/"
                    handlers:[f1, f2]
                    priority:3 + 1
                    nType:static
                    maxParams:0
                    wildChild:false
                    children:[
                        {
                            path:"2"
                            indices:""
                            handlers:[f1, f6]
                            priority:2
                            nType:static
                            maxParams:0
                            wildChild:false
                            children:[
                                {
                                    path:"/p56"
                                    indices:""
                                    handlers:[f1, f7]
                                    priority:1
                                    nType:static
                                    maxParams:0
                                    wildChild:false
                                    children:[]
                                }
                            ]
                        }
                        {
                            path:"/p34"
                            indices:""
                            handlers:[f1, f5]
                            priority:1
                            nType:static
                            maxParams:0
                            wildChild:false
                            children:[]
                        }
                    ]
                }
            ]
        }
    ]
}

// step8：添加{method: POST, path: /p12/p56/:id, handler:f8}
/
/p
/p1
/p1/p34
/p12
/p12/p56
/p12/p56/:id

node_/ = {
    path:"/"
    indices:"p"
    handlers:[f1, f4]
    priority:6 + 1
    nType:root
    maxParams:0 + 1
    wildChild:false
    children:[
        {
            path:"p"
            indices:"1"
            handlers:[f1, f3]
            priority:5 + 1
            nType:static
            maxParams:1
            wildChild:false
            children:[
                {
                    path:"1"
                    indices:"2/"
                    handlers:[f1, f2]
                    priority:4 + 1
                    nType:static
                    maxParams:1
                    wildChild:false
                    children:[
                        {
                            path:"2"
                            indices:""
                            handlers:[f1, f6]
                            priority:2 + 1
                            nType:static
                            maxParams:1
                            wildChild:false
                            children:[
                                {
                                    path:"/p56"
                                    indices:""
                                    handlers:[f1, f7]
                                    priority:1 + 1
                                    nType:static
                                    maxParams:1
                                    wildChild:false
                                    children:[
                                        {
                                            path:"/"
                                            indices:""
                                            handlers:[]
                                            priority:1
                                            nType:static
                                            maxParams:1
                                            wildChild:false
                                            children:[
                                                {
                                                    path:":id"
                                                    indices:""
                                                    handlers:[f1, f8]
                                                    priority:1
                                                    nType:param
                                                    maxParams:1
                                                    wildChild:false
                                                    children:[]
                                                }
                                            ]
                                        }
                                    ]
                                }
                            ]
                        }
                        {
                            path:"/p34"
                            indices:""
                            handlers:[f1, f5]
                            priority:1
                            nType:static
                            maxParams:0
                            wildChild:false
                            children:[]
                        }
                    ]
                }
            ]
        }
    ]
}

// step9：添加{method: POST, path: /p12/p56/:id/p78, handler:f9}
/
/p
/p1
/p1/p34
/p12
/p12/p56
/p12/p56/:id

node_/ = {
    path:"/"
    indices:"p"
    handlers:[f1, f4]
    priority:7 + 1
    nType:root
    maxParams:0 + 1
    wildChild:false
    children:[
        {
            path:"p"
            indices:"1"
            handlers:[f1, f3]
            priority:6 + 1
            nType:static
            maxParams:1
            wildChild:false
            children:[
                {
                    path:"1"
                    indices:"2/"
                    handlers:[f1, f2]
                    priority:5 + 1
                    nType:static
                    maxParams:1
                    wildChild:false
                    children:[
                        {
                            path:"2"
                            indices:"/"
                            handlers:[f1, f6]
                            priority:3 + 1
                            nType:static
                            maxParams:1
                            wildChild:false
                            children:[
                                {
                                    path:"/p56"
                                    indices:""
                                    handlers:[f1, f7]
                                    priority:2 + 1
                                    nType:static
                                    maxParams:1
                                    wildChild:false
                                    children:[
                                        {
                                            path:"/"
                                            indices:""
                                            handlers:[]
                                            priority:1 + 1
                                            nType:static
                                            maxParams:1
                                            wildChild:true
                                            children:[
                                                {
                                                    path:":id"
                                                    indices:""
                                                    handlers:[f1, f8]
                                                    priority:1 + 1
                                                    nType:param
                                                    maxParams:1
                                                    wildChild:false
                                                    children:[
                                                        {
                                                            path:"/p78"
                                                            indices:""
                                                            handlers:[f1, f9]
                                                            priority:1
                                                            nType:static
                                                            maxParams:0
                                                            wildChild:false
                                                            children:[]
                                                        }
                                                    ]
                                                }
                                            ]
                                        }
                                    ]
                                }
                            ]
                        }
                        {
                            path:"/p34"
                            indices:""
                            handlers:[f1, f5]
                            priority:1
                            nType:static
                            maxParams:0
                            wildChild:false
                            children:[]
                        }
                    ]
                }
            ]
        }
    ]
}
```
## gin框架中间件
### 中间件API
```go
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery())  // 默认注册的两个中间件
	return engine
}
```
```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.Use(middleware...)  // 实际上还是调用的RouterGroup的Use函数
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}
```
```go
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```
 	注册路由时会将对应路由的函数和之前的中间件函数结合到一起  
```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)  // 将处理请求的函数与中间件函数结合
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}
```
```go
const abortIndex int8 = math.MaxInt8 / 2

func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
	finalSize := len(group.Handlers) + len(handlers)
	if finalSize >= int(abortIndex) {  // 这里有一个最大限制
		panic("too many handlers")
	}
	mergedHandlers := make(HandlersChain, finalSize)
	copy(mergedHandlers, group.Handlers)
	copy(mergedHandlers[len(group.Handlers):], handlers)
	return mergedHandlers
}

// 将一个路由的中间件函数和处理函数结合到一起组成一条处理函数链条HandlersChain，
// 而它本质上就是一个由HandlerFunc组成的切片
type HandlersChain []HandlerFunc

```
### 中间件的执行
```go
value := root.getValue(rPath, c.Params, unescape)
if value.handlers != nil {
  c.handlers = value.handlers
  c.Params = value.params
  c.fullPath = value.fullPath
  c.Next()  // 执行函数链条
  c.writermem.WriteHeaderNow()
  return
}

func (c *Context) Next() {
	c.index++
	for c.index < int8(len(c.handlers)) {
		c.handlers[c.index](c)
		c.index++
	}
}

func (c *Context) Abort() {
	c.index = abortIndex  // 直接将索引置为最大限制值，从而退出循环
}

```
### 示例
#### 定义中间件
```go
// StatCost 是一个统计耗时请求耗时的中间件
func StatCost() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		c.Set("name", "小王") // 可以通过c.Set在请求上下文中设置值，后续的处理函数能够取到该值
		// 调用该请求的剩余处理程序
		c.Next()
		// 不调用该请求的剩余处理程序
		// c.Abort()
		// 计算耗时
		cost := time.Since(start)
		log.Println(cost)
	}
}
```
#### 注册中间件
为全局路由注册
```go
func main() {
	// 新建一个没有任何默认中间件的路由
	r := gin.New()
	// 注册一个全局中间件
	r.Use(StatCost())
	
	r.GET("/test", func(c *gin.Context) {
		name := c.MustGet("name").(string) // 从上下文取值
		log.Println(name)
		c.JSON(http.StatusOK, gin.H{
			"message": "Hello world!",
		})
	})
	r.Run()
}
```
为某单个路由注册
```go
// 给/test2路由单独注册中间件（可注册多个）
	r.GET("/test2", StatCost(), func(c *gin.Context) {
		name := c.MustGet("name").(string) // 从上下文取值
		log.Println(name)
		c.JSON(http.StatusOK, gin.H{
			"message": "Hello world!",
		})
	})
```
为路由组注册
```go
// func 1
shopGroup := r.Group("/shop", StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}

// func2
shopGroup := r.Group("/shop")
shopGroup.Use(StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}

```
### c.Set()/c.Get()
c.Set()和c.Get()这两个方法多用于在多个函数之间通过c传递数据的，比如我们可以在认证中间件中获取当前请求的相关信息（userID等）通过c.Set()存入c，然后在后续处理业务逻辑的函数中通过c.Get()
## 运行多个服务
```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"golang.org/x/sync/errgroup"
)

var (
	g errgroup.Group
)

func router01() http.Handler {
	e := gin.New()
	e.Use(gin.Recovery())
	e.GET("/", func(c *gin.Context) {
		c.JSON(
			http.StatusOK,
			gin.H{
				"code":  http.StatusOK,
				"error": "Welcome server 01",
			},
		)
	})

	return e
}

func router02() http.Handler {
	e := gin.New()
	e.Use(gin.Recovery())
	e.GET("/", func(c *gin.Context) {
		c.JSON(
			http.StatusOK,
			gin.H{
				"code":  http.StatusOK,
				"error": "Welcome server 02",
			},
		)
	})

	return e
}

func main() {
	server01 := &http.Server{
		Addr:         ":8080",
		Handler:      router01(),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	server02 := &http.Server{
		Addr:         ":8081",
		Handler:      router02(),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}
   // 借助errgroup.Group或者自行开启两个goroutine分别启动两个服务
	g.Go(func() error {
		return server01.ListenAndServe()
	})

	g.Go(func() error {
		return server02.ListenAndServe()
	})

	if err := g.Wait(); err != nil {
		log.Fatal(err)
	}
}

```
