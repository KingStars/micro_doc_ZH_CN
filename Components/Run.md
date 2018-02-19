# Micro Run
micro run命令管理微服务的生命周期。它获取源代码，构建二进制文件并执行它。这是一个可用于本地开发的简单工具。如果没有指定参数，则微运行作为可以管理其他服务的服务来运行。

*注意：默认运行时（Go）需要设置PATH和GOPATH中的Go二进制文件。*

## 概述

### Run

```
micro run github.com/service/foo
```

### Status

```
micro run -s github.com/service/foo
```

### Kill

```
micro run -k github.com/service/foo
```

### 运行服务管理

```
micro run
```

### 推迟运行服务管理

```
micro run -x github.com/service/foo
```

### 运行并重新启动

```
micro run -r github.com/service/foo
```

### 运行并更新源代码

```
micro run -u github.com/service/foo
```

## 使用帮助

```
NAME:
   micro run - Run the micro runtime

USAGE:
   micro run [command options] [arguments...]

OPTIONS:
   -k	Kill service
   -r	Restart if dies. Default: false
   -u	Update the source. Default: false
   -x	Defer run to service. Default: false
   -s	Get service status
```

## TODO
- [ ]支持接受args和env变量的服务
- [ ]添加服务接口go-run
- [ ]支持Go以外的可配置运行时
- [ ]插件支持重建
- [ ]守护进程？
- [ ]监控内存消耗并kill？
- [ ]chroot的进程？
