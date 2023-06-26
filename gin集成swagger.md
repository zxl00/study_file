#### swagger参数介绍

- @Title

  > 这个API所表达的含义，是一个文本，空格之后的内容全部解析为title

- @Description

  > 这个API为详细的描述，是一个文本，空格之后的内容全部解析为Description

- @Param

  > 参数，表述需要传递到服务器端的参数，有五列参数，使用空格或者tab分割，五个参数分别表示的含义：
  >
  > 1. 参数名
  > 2. 参数类型，可以有的值是formData、query、path、body、header
  >    - formData：表示是post请求的数据
  >    - query：表示是带在url之后的参数
  >    - path：表示请求路径上的参数
  >    - body：表示请求体
  >    - header：表示带在header信息中的参数
  > 3. 参数类型
  > 4. 是否必须
  > 5. 注释

- @Success

  > 成功返回给客户端的信息，三个参数，第一个是status code，第二个是参数返回类型，必须使用{}包含，第三个是返回的对象或者字符串信息，如果是{object}类型，那个bee工具在生产docs的时候会扫描对应的对象，这个填写的是相对你项目的目录名和对象。三个参数必须使用空格隔开

- @Failure

  > 失败返回信息，包含两个参数，使用空格分割，第一个表示status code，第二个表示错误信息

- @Router

  > 路由信息，包含两个参数，使用空格分割，第一个是请求的路由地址，支持正则和自定义路由，和之前的路由规则一样，第二个参数是支持的请求方法，放在[]之中，如果有多个方法，那么使用，分隔。

------

#### swagger在gin中的使用

#### 工程结构

```
D:.
├─.idea
├─config
├─controller
│  └─router
├─dao
├─db
├─docs
├─middle
├─model
├─service
├─tmp
└─utils
```

> .idea：goland加载该项目的索引，可以删除
>
> config：配置文件文件夹
>
> controller：控制层，主要是接收请求使用
>
> controller.router：路由信息
>
> dao：数据库操作，包含数据库的增删改查
>
> db：用于初始化数据库连接及配置
>
> docs：swagger相关文件
>
> middel：中间件存放目录
>
> model：定义数据库的表字段
>
> service：处理接口逻辑
>
> tmp：运行go生成临时文件，可以删除
>
> utils：常用的工具

ping接口集成swagger

​	无请求参数

```golang
package controller

import (
	"auto-dingtalk/utils"
	"github.com/gin-gonic/gin"
)

type ping struct {
}

var Ping ping

// @Summary  ping接口
// @Tags 测试相关接口
// @version 1.0
// @Accept application/json
// @Success 200 {object} utils.ReturnMsg
// @Router /ping [post]
func (p *ping) Ping(ctx *gin.Context) {
	utils.ReturnContext(ctx).Success("ping ok", "pong")
}

```

> utils.ReturnContext(ctx).Success("ping ok", "pong")，这里是封装了一下结果返回的格式，其具体代码为
>
> utils-->response.go
>
> ```
> package utils
> 
> import (
> "github.com/gin-gonic/gin"
> "net/http"
> )
> 
> type BaseContext struct {
> ctx *gin.Context
> }
> 
> // 返回格式
> 
> type ReturnMsg struct {
> Code int         `json:"code"`
> Msg  string      `json:"msg"`
> Data interface{} `json:"data"`
> }
> 
> // 成功返回
> 
> func ReturnContext(ctx *gin.Context) *BaseContext {
> return &BaseContext{ctx: ctx}
> }
> func (BaseContext *BaseContext) Success(msg string, data interface{}) {
> resp := &ReturnMsg{
>    Code: 20000,
>    Msg:  msg,
>    Data: data,
> }
> BaseContext.ctx.JSON(http.StatusOK, resp)
> }
> 
> // 失败返回
> 
> func (BaseContext *BaseContext) Fail(msg string, data interface{}) {
> resp := &ReturnMsg{
>    Code: 50000,
>    Msg:  msg,
>    Data: data,
> }
> BaseContext.ctx.JSON(http.StatusOK, resp)
> }
> ```

> 注意： 
>
> // @Summary  ping接口
> // @Tags 测试相关接口
> // @version 1.0
> // @Accept application/json
> // @Success 200 {object} utils.ReturnMsg
> // @Router /ping [post]
>
> **这里对swagger的说明要和下边的方法紧挨着，不要空行**

##### main.go中对swagger的说明，不可少

```
package main

import (
	"auto-dingtalk/config"
	"auto-dingtalk/controller/router"
	"auto-dingtalk/utils"
	"context"
	"fmt"
	"github.com/wonderivan/logger"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// @title outgoing钉钉机器人
// @version 1.0
// @description 自动应答钉钉机器人
// @BasePath /api/v1
func main() {
	// 日志设置
	err := logger.SetLogger("./config/setlog.json")
	if err != nil {
		panic("日志设置出错了")
	}
	// 优雅关闭服务
	routers := router.Router.Baserouters()

	srv := &http.Server{
		Addr:           fmt.Sprintf("%s:%d", config.ListenAddr, config.ListenPort),
		Handler:        routers,
		MaxHeaderBytes: 1 << 20,
	}
	go func() {
		utils.Crontab()
	}()
	// 关闭服务
	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Info("listen: %s\n", err)

		}
	}()
	quit := make(chan os.Signal)
	// 获取停止服务信号，kill  -9 获取不到
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	logger.Info("shutdown server...")
	// 执行延迟停止
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		logger.Error("server shutdown:", err)
	}
	logger.Info("server exiting...")
}

```

> // @version 1.0  版本
>
> // @BasePath /api/v1  基础路由

##### 路由设置

controller-->router-->router.go

```
package router

import (
	"auto-dingtalk/controller"
	"auto-dingtalk/db"
	_ "auto-dingtalk/docs"
	"auto-dingtalk/model"
	"auto-dingtalk/utils"
	"github.com/gin-gonic/gin"
	swaggerFiles "github.com/swaggo/files"
	ginSwagger "github.com/swaggo/gin-swagger"
)

var Router router

type router struct{}

func (router *router) Baserouters() *gin.Engine {
	// 初始化数据库
	db.MysqlInit()
	// 同步表
	_ = db.GORM.AutoMigrate(&model.CategoryList{}, &model.AnswerList{})
	// 初始化gin
	r := gin.Default()
	r.GET("/health", func(ctx *gin.Context) {
		utils.ReturnContext(ctx).Success("health ok", "ok")
	})
	// swagger
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
	baseRouter := r.Group("/api/v1")
	{
		baseRouter.POST("/ping", controller.Ping.Ping)
		baseRouter.POST("/back_call", controller.DingBack.BackCall)
	}
	dingTalkRouter := baseRouter.Group("/ding_talk")
	{
		dingTalkRouter.POST("/category", controller.OutGoing.AddCategoryList)
		dingTalkRouter.POST("/answer", controller.OutGoing.AddAnswerList)
	}
	r.NoRoute(func(ctx *gin.Context) {
		utils.ReturnContext(ctx).Fail("网络请求错误", nil)
	})
	return r
}

```

##### 生成接口文档

- 编写完注释后，使用以下命令安装swag工具：

```
go get -u github.com/swaggo/swag/cmd/swag
```

> 当然，也可以提前安装，swag就是一个命令行工具

- 在项目根目录执行以下命令，使用swag工具生成接口文档数据。

```
swag init
```

> 注意：要在当前工程目录下，每次完成注释，需要执行该命令，生成接口文档

##### 效果展示

![](https://gitee.com/root_007/md_file_image/raw/master/202302071242703.png)

#### Gin中post请求集成swagger

​	带有请求参数

##### controller层中post请求接口代码示例

```
package controller

import (
	"auto-dingtalk/model"
	"auto-dingtalk/service"
	"auto-dingtalk/utils"
	"github.com/gin-gonic/gin"
	"github.com/wonderivan/logger"
)

type outgoing struct{}

var OutGoing outgoing

// 创建授权问题类别

// @Summary 机器人问题分类
// @Description 自动应答机器人回答问题类别
// @Tags 机器人
// @Accept application/json
// @Produce  application/json
// @Param data body  model.CategoryListRes true "post请求参数"
// @Success 200 {object} utils.ReturnMsg
// @Router /ding_talk/category [post]
func (o *outgoing) AddCategoryList(ctx *gin.Context) {
	params := new(model.CategoryList)
	if err := ctx.Bind(params); err != nil {
		logger.Error("Bind请求参数失败, " + err.Error())
		utils.ReturnContext(ctx).Fail("Bind请求参数失败", err.Error())
		return
	}
	err := service.OutGoing.AddCategoryList(params)
	if err != nil {
		logger.Error("" + err.Error())
		utils.ReturnContext(ctx).Fail("Fail", err.Error())
		return
	}
	utils.ReturnContext(ctx).Success("ok", "创建授权问题类别成功")
}
```

> @Param data body  model.CategoryListRes true "post请求参数"  **说明**：
>
> data：可以自定义，body参数的变量名称
>
> body：参数类别
>
> model.CategoryListRes：参数模型
>
> true：是否为必填
>
> post请求参数：参数说明

##### 传入参数模型

```
type CategoryListRes struct {
	Species     string `json:"species"`
	NumberOrder int    `json:"number_order"`
	IsActive    bool   `json:"is_active"`
}
```

> 需要注意的是，集成swagger的时，不要使用定义数据库的model，其中的模型的tag，不要使用gorm这个tag

##### 效果展示

![image-20230111135100621](https://gitee.com/root_007/md_file_image/raw/master/202302071242097.png)

