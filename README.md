- [安装与配置](#install)
- [框架架构](#arch)
	- [生命周期](#life-circle)
	- [Context](#context)
- [路由](#router)
	- [基本路由](#basic-router)
	- [路由参数](#router-param)
	- [路由群组](#router-group)
- [控制器](#controller)
- [请求](#request)
	- [请求头](#request-header)
	- [Cookies](#request-cookie)
	- [上传文件](#upload)
- [响应](#response)
	- [响应头](#response-header)
	- [附加Cookie](#response-cookie)
	- [字符串响应](#response-string)
	- [JSON响应](#response-json)
	- [视图响应](#response-view)
	- [文件下载](#response-file)
	- [重定向](#response-redirect)
	- [同步异步](#sync-async)
- [视图](#view)
	- [传参](#view-param)
	- [视图组件](#view-unit)
- [中间件](#middleware)
	- [分类使用](#middleware-use)
	- [创建中间件](#middleware-create)
	- [中间件参数](#middleware-param)
- [数据库](#db)
	- [Mongodb](#db-mongodb)
	- [Mysql](#db-mysql)
	- [ORM](#orm)
- [扩展包](#extensions)
- [常用方法](#functions)
	- [gin]()
	- [Context]()

<a name="install"></a>
### 安装与配置
安装：

```sh
$ go get gopkg.in/gin-gonic/gin.v1
```
`
注意：确保 GOPATH GOROOT 已经配置
`

导入：
```go
import "gopkg.in/gin-gonic/gin.v1"
```


<a name="arch"></a>

### 框架架构
<a name="http"></a>

- HTTP 服务器

**1.默认服务器**

```
router.Run()
```

**2.HTTP 服务器**

除了默认服务器中 `router.Run()` 的方式外，还可以用 `http.ListenAndServe()`，比如

```go
func main() {
	router := gin.Default()
	http.ListenAndServe(":8080", router)
}
```
或者自定义 HTTP 服务器的配置：

```go
func main() {
	router := gin.Default()

	s := &http.Server{
		Addr:           ":8080",
		Handler:        router,
		ReadTimeout:    10 * time.Second,
		WriteTimeout:   10 * time.Second,
		MaxHeaderBytes: 1 << 20,
	}
	s.ListenAndServe()
}
```

**3.HTTP 服务器替换方案**
想无缝重启、停机吗? 以下有几种方式：

我们可以使用 [fvbock/endless](https://github.com/fvbock/endless) 来替换默认的 `ListenAndServe`。但是 windows 不能使用。

```go
router := gin.Default()
router.GET("/", handler)
// [...]
endless.ListenAndServe(":4242", router)
```

除了 endless 还可以用manners:

[manners](https://github.com/braintree/manners) 兼容windows

```
manners.ListenAndServe(":8888", r)
```

如果你使用的 golang 版本大于 1.8 版本, 那么可以用 http.Server 内置的 Shutdown 方法来实现优雅的关闭服务, 一个简单的示例代码如下:

```
srv := http.Server{
    Addr: ":8080",
    Handler: router,
}

go func() {
    if err :+ srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("listen: %s\n", err)
    }
}

// 其他代码, 等待关闭信号
...

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
    log.Fatal("Server Shutdown: ", err)
}
log.Println("Server exiting")
```

完整的代码见 [graceful-shutdown](https://github.com/gin-gonic/gin/blob/master/examples/graceful-shutdown/graceful-shutdown/server.go).

<a name="life-circle"></a>

- 生命周期

<a name="context"></a>

- Context

<a name="router"></a>

### 路由
<a name="basic-router"></a>

- 基本路由
gin 框架中采用的路由库是 httprouter。


```go
	// 创建带有默认中间件的路由:
	// 日志与恢复中间件
	router := gin.Default()
	//创建不带中间件的路由：
	//r := gin.New()

	router.GET("/someGet", getting)
	router.POST("/somePost", posting)
	router.PUT("/somePut", putting)
	router.DELETE("/someDelete", deleting)
	router.PATCH("/somePatch", patching)
	router.HEAD("/someHead", head)
	router.OPTIONS("/someOptions", options)
```

<a name="router-param"></a>

- 路由参数

api 参数通过Context的Param方法来获取

```
router.GET("/string/:name", func(c *gin.Context) {
    	name := c.Param("name")
    	fmt.Println("Hello %s", name)
    })
```

URL 参数通过 DefaultQuery 或 Query 方法获取

```
// url 为 http://localhost:8080/welcome?name=ningskyer时
// 输出 Hello ningskyer
// url 为 http://localhost:8080/welcome时
// 输出 Hello Guest
router.GET("/welcome", func(c *gin.Context) {
	name := c.DefaultQuery("name", "Guest") //可设置默认值
	// 是 c.Request.URL.Query().Get("lastname") 的简写
	lastname := c.Query("lastname") 
	fmt.Println("Hello %s", name)
})
```
表单参数通过 PostForm 方法获取
```
//form
router.POST("/form", func(c *gin.Context) {
	type := c.DefaultPostForm("type", "alert")//可设置默认值
	msg := c.PostForm("msg")
	title := c.PostForm("title")
	fmt.Println("type is %s, msg is %s, title is %s", type, msg, title)
})
```
<a name="router-group"></a>

- 路由群组

```go
	someGroup := router.Group("/someGroup")
    {
        someGroup.GET("/someGet", getting)
		someGroup.POST("/somePost", posting)
	}
```

<a name="controller"></a>

### 控制器

<a name="binding"></a>

- 数据解析绑定

模型绑定可以将请求体绑定给一个类型，目前支持绑定的类型有 JSON, XML 和标准表单数据 (foo=bar&boo=baz)。
要注意的是绑定时需要给字段设置绑定类型的标签。比如绑定 JSON 数据时，设置 `json:"fieldname"`。
使用绑定方法时，Gin 会根据请求头中  Content-Type  来自动判断需要解析的类型。如果你明确绑定的类型，你可以不用自动推断，而用 BindWith 方法。
你也可以指定某字段是必需的。如果一个字段被 `binding:"required"` 修饰而值却是空的，请求会失败并返回错误。

```go
// Binding from JSON
type Login struct {
	User     string `form:"user" json:"user" binding:"required"`
	Password string `form:"password" json:"password" binding:"required"`
}

func main() {
	router := gin.Default()

	// 绑定JSON的例子 ({"user": "manu", "password": "123"})
	router.POST("/loginJSON", func(c *gin.Context) {
		var json Login

		if c.BindJSON(&json) == nil {
			if json.User == "manu" && json.Password == "123" {
				c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
			} else {
				c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			}
		}
	})

	// 绑定普通表单的例子 (user=manu&password=123)
	router.POST("/loginForm", func(c *gin.Context) {
		var form Login
		// 根据请求头中 content-type 自动推断.
		if c.Bind(&form) == nil {
			if form.User == "manu" && form.Password == "123" {
				c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
			} else {
				c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			}
		}
	})
	// 绑定多媒体表单的例子 (user=manu&password=123)
	router.POST("/login", func(c *gin.Context) {
		var form LoginForm
		// 你可以显式声明来绑定多媒体表单：
		// c.BindWith(&form, binding.Form)
		// 或者使用自动推断:
		if c.Bind(&form) == nil {
			if form.User == "user" && form.Password == "password" {
				c.JSON(200, gin.H{"status": "you are logged in"})
			} else {
				c.JSON(401, gin.H{"status": "unauthorized"})
			}
		}
	})
	// Listen and serve on 0.0.0.0:8080
	router.Run(":8080")
}
```
<a name="request"></a>

### 请求
<a name="request-header"></a>

- 请求头

<a name="request-params"></a>

- 请求参数

<a name="request-cookie"></a>

- Cookies

<a name="upload"></a>

- 上传文件

```
router.POST("/upload", func(c *gin.Context) {

    file, header , err := c.Request.FormFile("upload")
    filename := header.Filename
    fmt.Println(header.Filename)
    out, err := os.Create("./tmp/"+filename+".png")
    if err != nil {
        log.Fatal(err)
    }
    defer out.Close()
    _, err = io.Copy(out, file)
    if err != nil {
        log.Fatal(err)
    }   
})
```
<a name="response"></a>

### 响应
<a name="response-header"></a>

- 响应头

<a name="response-cookie"></a>

- 附加Cookie

<a name="response-string"></a>

- 字符串响应

```
c.String(http.StatusOK, "some string")
```
<a name="response-json"></a>

- JSON/XML/YAML响应

```
r.GET("/moreJSON", func(c *gin.Context) {
	// You also can use a struct
	var msg struct {
		Name    string `json:"user" xml:"user"`
		Message string
		Number  int
	}
	msg.Name = "Lena"
	msg.Message = "hey"
	msg.Number = 123
	// 注意 msg.Name 变成了 "user" 字段
	// 以下方式都会输出 :   {"user": "Lena", "Message": "hey", "Number": 123}
	c.JSON(http.StatusOK, gin.H{"user": "Lena", "Message": "hey", "Number": 123})
	c.XML(http.StatusOK, gin.H{"user": "Lena", "Message": "hey", "Number": 123})
	c.YAML(http.StatusOK, gin.H{"user": "Lena", "Message": "hey", "Number": 123})
	c.JSON(http.StatusOK, msg)
	c.XML(http.StatusOK, msg)
	c.YAML(http.StatusOK, msg)
})
		
```
<a name="response-view"></a>

- 视图响应

先要使用 LoadHTMLTemplates() 方法来加载模板文件

```go
func main() {
	router := gin.Default()
	//加载模板
	router.LoadHTMLGlob("templates/*")
	//router.LoadHTMLFiles("templates/template1.html", "templates/template2.html")
	//定义路由
	router.GET("/index", func(c *gin.Context) {
		//根据完整文件名渲染模板，并传递参数
		c.HTML(http.StatusOK, "index.tmpl", gin.H{
			"title": "Main website",
		})
	})
	router.Run(":8080")
}
```

模板结构定义

```html
<html>
	<h1>
		{{ .title }}
	</h1>
</html>
```
不同文件夹下模板名字可以相同，此时需要 LoadHTMLGlob() 加载两层模板路径

```go
router.LoadHTMLGlob("templates/**/*")
router.GET("/posts/index", func(c *gin.Context) {
	c.HTML(http.StatusOK, "posts/index.tmpl", gin.H{
		"title": "Posts",
	})
	c.HTML(http.StatusOK, "users/index.tmpl", gin.H{
		"title": "Users",
	})
	
}
```

templates/posts/index.tmpl
```html
<!-- 注意开头 define 与结尾 end 不可少 -->
{{ define "posts/index.tmpl" }}
<html><h1>
	{{ .title }}
</h1>
</html>
{{ end }}

gin也可以使用自定义的模板引擎，如下

```go
import "html/template"

func main() {
	router := gin.Default()
	html := template.Must(template.ParseFiles("file1", "file2"))
	router.SetHTMLTemplate(html)
	router.Run(":8080")
}
```
<a name="response-file"></a>

- 文件响应

```
//获取当前文件的相对路径
router.Static("/assets", "./assets")
//
router.StaticFS("/more_static", http.Dir("my_file_system"))
//获取相对路径下的文件
router.StaticFile("/favicon.ico", "./resources/favicon.ico")

```
<a name="response-redirect"></a>

- 重定向

```
r.GET("/redirect", func(c *gin.Context) {
	//支持内部和外部的重定向
    c.Redirect(http.StatusMovedPermanently, "http://www.baidu.com/")
})
```
<a name="sync-async"></a>

- 同步异步

goroutine 机制可以方便地实现异步处理

```go
func main() {
	r := gin.Default()
	//1. 异步
	r.GET("/long_async", func(c *gin.Context) {
		// goroutine 中只能使用只读的上下文 c.Copy()
		cCp := c.Copy()
		go func() {
			time.Sleep(5 * time.Second)

			// 注意使用只读上下文
			log.Println("Done! in path " + cCp.Request.URL.Path)
		}()
	})
	//2. 同步
	r.GET("/long_sync", func(c *gin.Context) {
		time.Sleep(5 * time.Second)

		// 注意可以使用原始上下文
		log.Println("Done! in path " + c.Request.URL.Path)
	})

	// Listen and serve on 0.0.0.0:8080
	r.Run(":8080")
}
```
<a name="view"></a>

### 视图
<a name="view-param"></a>

- 传参

<a name="view-unit"></a>

- 视图组件

<a name="middleware"></a>

### 中间件
<a name="middleware-use"></a>

- 分类使用方式
```
// 1.全局中间件
router.Use(gin.Logger())
router.Use(gin.Recovery())

// 2.单路由的中间件，可以加任意多个
router.GET("/benchmark", MyMiddelware(), benchEndpoint)

// 3.群组路由的中间件
authorized := router.Group("/", MyMiddelware())
// 或者这样用：
authorized := router.Group("/")
authorized.Use(MyMiddelware())
{
	authorized.POST("/login", loginEndpoint)
}
```
<a name="middleware-create"></a>

- 自定义中间件

```go
//定义
func Logger() gin.HandlerFunc {
	return func(c *gin.Context) {
		t := time.Now()

		// 在gin上下文中定义变量
		c.Set("example", "12345")

		// 请求前

		c.Next()//处理请求

		// 请求后
		latency := time.Since(t)
		log.Print(latency)

		// access the status we are sending
		status := c.Writer.Status()
		log.Println(status)
	}
}
//使用
func main() {
	r := gin.New()
	r.Use(Logger())

	r.GET("/test", func(c *gin.Context) {
		//获取gin上下文中的变量
		example := c.MustGet("example").(string)

		// 会打印: "12345"
		log.Println(example)
	})

	// 监听运行于 0.0.0.0:8080
	r.Run(":8080")
}
```

<a name="middleware-param"></a>

- 中间件参数

- 内置中间件
1.简单认证BasicAuth

```go
// 模拟私有数据
var secrets = gin.H{
	"foo":    gin.H{"email": "foo@bar.com", "phone": "123433"},
	"austin": gin.H{"email": "austin@example.com", "phone": "666"},
	"lena":   gin.H{"email": "lena@guapa.com", "phone": "523443"},
}

func main() {
	r := gin.Default()

	// 使用 gin.BasicAuth 中间件，设置授权用户
	authorized := r.Group("/admin", gin.BasicAuth(gin.Accounts{
		"foo":    "bar",
		"austin": "1234",
		"lena":   "hello2",
		"manu":   "4321",
	}))

	// 定义路由
	authorized.GET("/secrets", func(c *gin.Context) {
		// 获取提交的用户名（AuthUserKey）
		user := c.MustGet(gin.AuthUserKey).(string)
		if secret, ok := secrets[user]; ok {
			c.JSON(http.StatusOK, gin.H{"user": user, "secret": secret})
		} else {
			c.JSON(http.StatusOK, gin.H{"user": user, "secret": "NO SECRET :("})
		}
	})

	// Listen and serve on 0.0.0.0:8080
	r.Run(":8080")
}
```

2.
<a name="db"></a>

## 数据库
<a name="db-mongodb"></a>

- Mongodb

Golang常用的Mongodb驱动为 mgo.v2, [查看文档](http://godoc.org/gopkg.in/mgo.v2)

mgo 使用方式如下：

```
//定义 Person 结构，字段须为首字母大写
type Person struct {
	Name string
	Phone string
}

router.GET("/mongo", func(context *gin.Context){
	//可本地可远程，不指定协议时默认为http协议访问，此时需要设置 mongodb 的nohttpinterface=false来打开httpinterface。
	//也可以指定mongodb协议，如 "mongodb://127.0.0.1:27017"
	var MOGODB_URI = "127.0.0.1:27017"
	//连接
	session, err := mgo.Dial(MOGODB_URI)
	//连接失败时终止
	if err != nil {
        panic(err)
    }
	//延迟关闭，释放资源
	defer session.Close()
	//设置模式
    session.SetMode(mgo.Monotonic, true)
	//选择数据库与集合
    c := session.DB("adatabase").C("acollection")
    //插入文档
    err = c.Insert(&Person{Name:"Ale", Phone:"+55 53 8116 9639"},
               &Person{Name:"Cla",  Phone:"+55 53 8402 8510"})
	//出错判断
    if err != nil {
            log.Fatal(err)
    }
	//查询文档
    result := Person{}
    //注意mongodb存储后的字段大小写问题
    err = c.Find(bson.M{"name": "Ale"}).One(&result)
    //出错判断
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Phone:", result.Phone)
})
```
<a name="db-mysql"></a>

- Mysql

<a name="ORM"></a>

- ORM

<a name="extensions"></a>

## 扩展包

<a name="functions"></a>

### 常用方法
- gin
- Context
