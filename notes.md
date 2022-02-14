# 源码阅读笔记

## 一些点

- `cmd` 模块  也有一个 `main.go`

- `github.com/minio/cli` fork 别人的项目

- `"net/http"` 用的标准库的 http 框架

- `http` 模块 -> `server.go` ->  `func NewServer`  创建一个新的 http 服务

- `cmd` -> `storage-errors.go` 自定义的错误， 用`errors.As` 或 `errors.Is` 来检查

- 当程序出现需要终止的错误时使用`logger.Fatal()`

- `logger.FatalIf()` 是一个自定义的函数，如果有错误的话，就 `Fatal`这样可以简化`错误检查`代码

- `cmd` -> `storage-errors.go` -> `osErrToFileErr` 系统地把文件系统的错误转换成 minio 的错误 

- `func initDataScanner` ： 相当于后台定时运行一个协程

- 使用`ioutil.WriteFile`写入文件

## 待解疑惑

#### 1. 为什么读取目录下的所有文件要用 `syscall.ReadDirent`而并不是`ioutil.ReadDir`?

#### 2. 并发上传时，采用了锁，但是如何保证性能，让其他没抢到锁的线程任然可以上传？

  **测试的结果是:**
  两个客户端可以同时并发上传文件到同一个 key，两个都会成功，都没有阻塞，后完成上传的会覆盖先完成上传的。
  跟谁先开始上传没关系。

  上传文件时， `minio` 会将文件内容保存到临时目录`drive/.minio.sys/tmp/[fsUUID]/[tempObjUUID]`。再重命名到目标路径。

  `fsUUID`每次启动程序的时候生产一个， 可以理解为对应一个单独的实例

  `tempObjUUID` 每调用一次`putObject`函数生成一个， 可以立即为一次请求生产一个。

  putObject函数执行完毕后， 不管出错还是成功，都会删除这个临时文件。

## 搭建测试环境

### 使用 GUI 工具测试

工具：cyberduck 或者 自带 web 界面

### 打开 debug 模式

```shell
export _MINIO_SERVER_DEBUG = on
```

## 项目架构

- minio
  
  - cmd  # 具体的读写本地文件都在这个模块，看起来像是 control + view
    
    - ***-hangdlers.go  # 处理器，类似于 views
    - api-errors.go  # 自定义 http 错误，利用了常量 iota
  
  - internal
    
    - http  # http 服务
    
    - logger  # 封装的自定义日志组件
    
    - dsync  # 锁？

## 函数级别

`cmd` -> `FSObjects` 是 `ObjectLayer` 接口的实现

`serverMain` -> `configureServerHandler` -> `registerAPIRouter` 进行s3 接口的路由配置 

`drives` ： 存储路径

```go
// Config represents cache config settings
type Config struct {
    Enabled         bool     `json:"-"`  // - 表示序列化的时候忽略该字段
    Drives          []string `json:"drives"`
    Expiry          int      `json:"expiry"`
    ...
}
```

`fsMetaV1`  结构体的 `WriteTo()` 方法用来保存 json 文件， 也就是元数据

### 处理流程

#### 删除对象

 `routers` -> `objectAPIHandlers.DeleteObjectHandler()` -> `objects.DeleteObject()`

#### 上传对象

`PutObjectHandler`-> `FSObjects{}.PutObject()`-> `FSObjects{}.putObject` -> `fsCreateFile()`