# Docker入门教程101: 基于Docker部署Go项目

## 环境准备

1. Docker安装：参考我的上一篇文章[Docker入门教程101：用途，架构，安装和使用](https://github.com/jincheng9/disributed-system-notes/tree/main/docker/01)。
2. Go项目代码准备

​	基于golang最流行的Web框架Gin，搭建一个最简单的Web服务，大家可以下载[zip包](https://github.com/jincheng9/disributed-system-notes/archive/refs/heads/main.zip)，或者使用git下载源码：

```bash
$ git clone git@github.com:jincheng9/disributed-system-notes.git
```

​	下载后，用VSCode，Goland或其它IDE打开`disributed-system-notes/docker/02/go-docker-demo`目录。

​	在该目录下执行如下2条命令，服务正常启动后，会监听8080端口

```bash
$ go build main.go
$ ./main
```

​	在浏览器上输入[http://localhost:8080/hello](http://localhost:8080/hello)，如果有输出如下结果，就表示一切准备就绪了。

```markdown
{
	"msg": "world"
}
```



## 创建Dockerfile

在`go-docker-demo`目录下，创建文件`Dockerfile`，文件名全称就叫`Dockerfile`，没有后缀。

`Dockerfile`文件内容为：

```dockerfile
FROM golang:latest

WORKDIR /app/demo
COPY . .

RUN go build main.go

EXPOSE 8080
ENTRYPOINT ["./main"]
```

`FROM`: 指定基础镜像。我们的项目需要用到Go，所以指定golang的最新版本为基础镜像

`WORKDIR`：指定本项目在容器里的工作目录或者说存储位置。设置了`WORKDIR`后，`Dockerfile`里后续的指令如果要使用容器里的路径，就可以根据`WORKDIR`来使用相对路径了。

`COPY`：把执行`docker build`指定的目录下的某些文件或目录拷贝到容器的指定路径下。例子里的第一个`.`表示`docker build`指定的当前目录，第二个`.`表示容器的当前工作目录`WORKDIR`，该指令表示把`docker build`指定的目录下的所有内容(包括子目录下的文件)全部拷贝到容器的`/app/demo`目录下。

`RUN`：在指定的容器工作目录执行命令。例子表示在`WORKDIR`下执行`go build main.go`，会生成`main`二进制文件。

`EXPOSE`：声明容器要使用的端口。

`ENTRYPOINT`：指定容器的启动程序和参数。

Dockerfile文件语法指引：https://docs.docker.com/engine/reference/builder/



## 构建镜像

在`go-docker-demo`目录下，执行如下命令来构建镜像：

```bash
$ docker build -t go-docker-demo .
```

执行完成后，使用`docker image ls`可以查看到`REPOSITORY`为`go-docker-demo`的镜像文件。



## 运行容器

执行如下命令，启动容器：

```bash
$ docker run -d -p 8080:8080 go-docker-demo
```

成功执行后，该命令会返回类似`2b7a47d1e24265e638a2b931561a303f97463fac9d9f5fa5a9f9b77b2212fa24`这样的字符串，这个是运行的容器的ID，也叫container id。

在浏览器上输入[http://localhost:8080/hello](http://localhost:8080/hello)，如果有输出如下结果，就表示大功告成了。

```markdown
{
	"msg": "world"
}
```



## 容器长啥样

通过Docker Desktop里的`Containers/Apps`这个Tab页找到运行的容器`go-docker-demo`，点击右边第2个CLI图标，就可以进去到容器里了。

分别执行`pwd`, `ls`命令，就能以容器的视角看到当前容器里的文件目录结构。

```bash
# pwd
/app/demo
# ls
Dockerfile  go.mod  go.sum  main  main.go
# curl http://127.0.0.1:8080/hello            
{"msg":"world"}
# ls /
app  boot  etc	home  lib64  mnt  proc	run   srv  tmp	var
bin  dev   go	lib   media  opt  root	sbin  sys  usr
```



## 开源地址

文章和示例代码开源地址在GitHub: https://github.com/jincheng9/disributed-system-notes

公众号：coding进阶

个人网站：https://jincheng9.github.io/



## References

*  https://docs.docker.com/engine/reference/builder/ 