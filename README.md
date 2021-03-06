# go-docker

以一个项目case，演示如果将一个golang 项目打包到一个docker image

## 项目步骤
- 新建项目目录
``` shell
  $mkdir go-docker
```

- 在目录中新建文件server.go
``` shell
  $touch server.go
```

- 在server.go文件中，用下面的代码搭建一个简单http服务
``` golang
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gorilla/mux"
	"gopkg.in/natefinch/lumberjack.v2"
)

func handler(w http.ResponseWriter, r *http.Request) {
	query := r.URL.Query()
	name := query.Get("name")
	if name == "" {
		name = "Guest"
	}
	log.Printf("Received request for %s\n", name)
	w.Write([]byte(fmt.Sprintf("Hello, %s\n", name)))
}

func main() {
	// Create Server and Route Handlers
	r := mux.NewRouter()

	r.HandleFunc("/", handler)

	srv := &http.Server{
		Handler:      r,
		Addr:         ":8080",
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	// Configure Logging
	LOG_FILE_LOCATION := os.Getenv("LOG_FILE_LOCATION")
	if LOG_FILE_LOCATION != "" {
		log.SetOutput(&lumberjack.Logger{
			Filename:   LOG_FILE_LOCATION,
			MaxSize:    500, // megabytes
			MaxBackups: 3,
			MaxAge:     28,   //days
			Compress:   true, // disabled by default
		})
	}

	// Start Server
	go func() {
		log.Println("Starting Server")
		if err := srv.ListenAndServe(); err != nil {
			log.Fatal(err)
		}
	}()

	// Graceful Shutdown
	waitForShutdown(srv)
}

func waitForShutdown(srv *http.Server) {
	interruptChan := make(chan os.Signal, 1)
	signal.Notify(interruptChan, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

	// Block until we receive our signal.
	<-interruptChan

	// Create a deadline to wait for.
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
	defer cancel()
	srv.Shutdown(ctx)

	log.Println("Shutting down")
	os.Exit(0)
}
``` 

- go version 1.11+ 通过go mod 管理依赖并且编译
``` shell
  $go mod init go-docker
  $go build
  $./go-docker #可以启动服务，8080端口可以进行http服务 curl http://localhost:8080?name=holos
```

- 在项目目录中添加 Dockerfile 文件
``` shell
  $touch Dockerfile
```

- 在 Dockerfile 中写入下面的内容
``` dockerfile
# Dockerfile References: https://docs.docker.com/engine/reference/builder/

# Start from golang v1.11 base image
FROM golang:1.11

# Add Maintainer Info
LABEL maintainer="easierway <easierway@gmail.com>"

# Set the Current Working Directory inside the container
WORKDIR ~/go-docker

# Copy everything from the current directory to the PWD(Present Working Directory) inside the container
COPY . .

# Download all the dependencies
# https://stackoverflow.com/questions/28031603/what-do-three-dots-mean-in-go-command-line-invocations
RUN go get -d -v ./...

# Install the package
RUN go install -v ./...

# This container exposes port 8080 to the outside world
EXPOSE 8080

# Run the executable
CMD ["go-docker"]
```

- 构建 go-docker image文件
```shell
  $docker build -t go-docker .
```

- 构建成功后，可以在image列表中看到go-docker image
``` shell
  $docker image ls
  REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
  go-docker           latest              a1f111d527ad        12 minutes ago      792MB
```

- 在容器中运行 go-docker image
```shell
  $docker run -d -p 8080:8080 --name go-docker go-docker
```

- go-docker 容器实例的一些操作
```shell
  $docker container ls   #查看容器实例
  CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS                    NAMES
  c5ced58aa2d4        go-docker           "go-docker"         About a minute ago   Up About a minute   0.0.0.0:8080->8080/tcp   go-docker

  $docker exec -it go-docker bash #进入运行的容器实例go-docker,并且启用bash交互

  $docker stop go-docker #关闭一个container实例
```

- 通过volume 共享Host OS的目录用于container 应用日志的写入,增加Dockerfile.volume文件，并写入下面的内容
```dockerfile
# Dockerfile References: https://docs.docker.com/engine/reference/builder/

# Start from golang v1.11 base image
FROM golang:1.11

# Add Maintainer Info
LABEL maintainer="easierway <easierway@gmail.com>"

# Build Args
ARG APP_NAME=go-docker
ARG LOG_DIR=/${APP_NAME}/logs

# Create Log Directory
RUN mkdir -p ${LOG_DIR}

# Environment Variables
ENV LOG_FILE_LOCATION=${LOG_DIR}/app.log 


# Set the Current Working Directory inside the container
WORKDIR ~/go-docker

# Copy everything from the current directory to the PWD(Present Working Directory) inside the container
COPY . .

# Download all the dependencies
# https://stackoverflow.com/questions/28031603/what-do-three-dots-mean-in-go-command-line-invocations
RUN go get -d -v ./...

# Install the package
RUN go install -v ./...

# This container exposes port 8080 to the outside world
EXPOSE 8080

# Declare volumes to mount
VOLUME ["/go-docker/logs"]

# Run the executable
CMD ["go-docker"]
```

- 重新 build image, 并且通过挂载 volume方式启动container
```shell
  $docker build -t go-docker-volume -f Dockerfile.volume .  #指定Dockerfile.volume文件构建image go-docker-volume

  $docker image ls  #查看构建成功的镜像
  REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
  go-docker-volume    latest              6c073081c4fe        6 seconds ago       792MB

  $mkdir /tmp/app-logs  #创建host log dir
  $docker run -d -p 8080:8080 -v /tmp/app-logs/:/go-docker/logs go-docker-volume  #挂载volume，启动container实例
  $tailf /tmp/app-logs/app.log #服务可以访问，日志挂载成功

```

- 通过 Multi-stage builds减少应用image的大小，创建 Dockerfile.multistage文件，并填充下面的信息
``` dockerfile
# Dockerfile References: https://docs.docker.com/engine/reference/builder/

# Start from golang v1.11 base image
FROM golang:1.11 as builder

# Add Maintainer Info
LABEL maintainer="easierway <easierway@gmail.com>"

# Set the Current Working Directory inside the container
WORKDIR ~/go-docker

# Copy everything from the current directory to the PWD(Present Working Directory) inside the container
COPY . .

# Download dependencies
RUN go get -d -v ./...

# Build the Go app
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /go/bin/go-docker .


######## Start a new stage from scratch #######
FROM alpine:latest  

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy the Pre-built binary file from the previous stage
COPY --from=builder /go/bin/go-docker .

EXPOSE 8080

CMD ["./go-docker"] 
```

- 通过 Multi-stage重新 build image, 并且通过优化后的image启动container
```shell
  $docker build -t go-docker-optimized -f Dockerfile.multistage .  #指定Dockerfile.multifile文件multi stage build image go-docker-optimized

  $docker image ls  #查看构建成功的镜像
  REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
  go-docker-optimized   latest              1fe23e2d1059        8 seconds ago       12MB

  $docker run -d -p 8080:8080  go-docker-optimized  #使用go-docker-optimized image 启动container实例

```