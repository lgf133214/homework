# homework
go训练营作业

## 第三周 基于 errgroup 实现一个 http server 的启动和关闭 ，以及 linux signal 信号的注册和处理，要保证能够一个退出，全部注销退出

```golang
package main

import (
	"context"
	"github.com/gin-gonic/gin"
	"golang.org/x/sync/errgroup"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
)

var sigs chan os.Signal
var closed = make(chan struct{})

func init() {
	gin.SetMode(gin.ReleaseMode)

	// 注册 && 初始化 信号
	sigs = make(chan os.Signal)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	
	// 信号的后台监听
	go func() {
		sig := <-sigs
		log.Println("sig:", sig, "received")
		close(closed)
	}()
}

func main() {
	defer log.Println("Main is exited.")

	// errgroup 初始化
	g, ctx := errgroup.WithContext(context.Background())

	g.Go(func() error {
		s := HelloServer()

		go func() {
			// 其他 errgroup 组内成员失败 或 收到信号 就执行 Shutdown
			select {
			case <-ctx.Done():
			case <-closed:
			}

			ctx, cancel := context.WithTimeout(context.Background(), 1e9)
			defer cancel()

			log.Println("HelloServer will shutdown ...")
			err := s.Shutdown(ctx)
			if err != nil {
				log.Println(err)
			}
			// 这里其实可能时间太短（基本不可能），输出不了，小问题
		}()

		// 监听是主线任务，所以不能放到后台（即不能和上面的 go func(){ ... } 调换位置）
		return s.ListenAndServe()
	})

	g.Go(func() error {
		s := NotAllowedServer()

		go func() {
			// 其他 errgroup 组内成员失败 或 收到信号 就执行 Shutdown
			select {
			case <-ctx.Done():
			case <-closed:
			}

			ctx, cancel := context.WithTimeout(context.Background(), 1e9)
			defer cancel()

			log.Println("NotAllowedServer will shutdown ...")
			err := s.Shutdown(ctx)
			if err != nil {
				log.Println(err)
			}
			// 这里其实可能时间太短（基本不可能），输出不了，小问题
		}()

		// 监听是主线任务，所以不能放到后台（即不能和上面的 go func(){ ... } 调换位置）
		return s.ListenAndServe()
	})

	err := g.Wait()
	if err != nil {
		log.Println("errgroup.Group.err:", err.Error())
	}

	log.Println("Main done.")
}

// Gin 注册 及 返回 Server 指针，因为要用 http.Server 的 Shutdown 方法
func HelloServer() *http.Server {
	r := gin.New()
	r.Use(gin.Logger())
	r.GET("/", func(c *gin.Context) {
		c.String(200, "hello,world!")
	})

	s := &http.Server{
		Addr:    ":8000",
		Handler: r,
	}
	return s
}

// Gin 注册 及 返回 Server 指针，因为要用 http.Server 的 Shutdown 方法
func NotAllowedServer() *http.Server {
	r := gin.New()
	r.Use(gin.Logger())
	r.GET("/", func(c *gin.Context) {
		c.String(403, "You are not allowed for this page.")
	})

	s := &http.Server{
		Addr:    ":8001",
		Handler: r,
	}
	return s
}

```
```golang
// 输出：

2021/05/08 20:23:56 sig: interrupt received
2021/05/08 20:23:56 HelloServer will shutdown ...
2021/05/08 20:23:56 NotAllowedServer will shutdown ...
2021/05/08 20:23:56 errgroup.Group.err: http: Server closed
2021/05/08 20:23:56 Main done.
2021/05/08 20:23:56 Main is exited.

```


### 感悟？
  - gin 源码需要阅读
  - signal 知识点
  
### 疑问
  - 输出的时候发现，stderr 和 stdout 是随机选择顺序的，如
  ```golang
  package main

import "os"

func main() {
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")
	os.Stderr.WriteString("aaaaaaaaaaaaa\n")


	os.Stdout.WriteString("aaaaaaaaaaaaa\n")
	os.Stdout.WriteString("aaaaaaaaaaaaa\n")
	os.Stdout.WriteString("aaaaaaaaaaaaa\n")
	os.Stdout.WriteString("aaaaaaaaaaaaa\n")
	os.Stdout.WriteString("aaaaaaaaaaaaa\n")
	os.Stdout.WriteString("aaaaaaaaaaaaa\n")
	os.Stdout.WriteString("aaaaaaaaaaaaa\n")
	os.Stdout.WriteString("aaaaaaaaaaaaa\n")
}
  ```
  stdout 和 stderr 每次出现的顺序不一样，解释解释？🤭 :)
