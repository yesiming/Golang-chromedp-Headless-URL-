## Golang+chromedp+Headless实现百万级URL有效性的测试

### 一、项目背景

从TOP站点中抓取了最常访问的100万个URL，要测试是否走DDWI两种情况的URL访问情况，记录Status，如果有异常还需将访问页面进行截图。

目的：走DDWI的情况中，经过DDWI解密后不能访问的URL找出来，分析不能访问的原因，找出DDWI存在的Bug。

### 二、实现方式

1、由于大数据处理以及高并发，同时也为了性能测试做准备，所以选择Golang来做，Golang最大的特点是在语言层面上就支持并发，所以在并发这一块，Go语言拥有很大的优势，其次就是与python比起来，执行速度快，占用内存小，采用协程的方式，能很好地处理大数量级的请求。

2、Golang也有大量的第三方库，本次要用到chrome的Webdriver来模拟URL请求，就像Python里的Selenium一样，Go也提供了一个chromedp的第三方库，来向chrome浏览器进行操作。

3、由于有100万个URL，并且我们是自动化操作，从效率和速度上来考虑，此次选择了Chrome-headless模式，无头浏览器是自动化测试和不需要可见UI外壳的服务器环境的绝佳工具。我们本次只需检测URL的访问情况，所以采用无头浏览器无疑是最好的选择。

### 三、具体过程

1、调用Chromedp库，来实现访问一个URL并将访问结果写入txt文档中。代码为：

```go
func (s *Score) Do() {
	fmt.Printf("num:%d  url:%s\n", s.Num, s.Url)
	// fmt.Println("num:", s.Url)
	// opts := chromedp.ExecPath("C:\\Users\\10049\\AppData\\Local\\Google\\Chrome\\Application\\chrome.exe")

	// allocCtx, cancel := chromedp.NewExecAllocator(context.Background(), opts)
	// defer cancel()

	// taskCtx, cancel := chromedp.NewContext(allocCtx, chromedp.WithLogf(log.Printf))
	// defer cancel()

	var buf []byte
	// ch := make(chan []byte, 1024)
	// create chrome instance
	taskCtx, cancel := chromedp.NewContext(
		context.Background(),
		chromedp.WithLogf(log.Printf),
	)
	defer cancel()

	// create a timeout
	taskCtx, cancel = context.WithTimeout(taskCtx, 30*time.Second)
	defer cancel()

	// ensure that the browser process is started
	if err := chromedp.Run(taskCtx); err != nil {
		panic(err)
		// fmt.Println(err)
	}

	// listen network event
	listenForNetworkEvent(taskCtx) //,ch

	chromedp.Run(taskCtx,
		network.Enable(),
		chromedp.Navigate(s.Url),
		// chromedp.WaitVisible(`#doc-info`), //, chromedp.BySearch
		chromedp.CaptureScreenshot(&buf),
	)

	if flag == true{
		pngname := strings.Replace(s.Url,".","1",-1)
		pngname = strings.Replace(pngname,"http://","1",-1)
		log.Println(buf)
		if err1 := ioutil.WriteFile(pngname+".png", buf, 0644); err1 != nil {
			log.Fatal(err1)
		}
	}

	time.Sleep(30 * time.Second)
}

func listenForNetworkEvent(ctx context.Context) { 
	count := 0 // 因为访问一个URL会有很多的链接，这里我们只打印主页的Status
	chromedp.ListenTarget(ctx, func(ev interface{}) {

		switch ev := ev.(type) {

		case *network.EventResponseReceived:
			count++
			resp := ev.Response
			if count < 2 {
				if resp.Status == 200 {
					log.Printf("Success, url:%s; status:%d; text:%s", resp.URL, resp.Status, resp.StatusText)
					writefile("./success.txt", "Url:"+resp.URL+" Status:200"+" Text:"+resp.StatusText+"\r\n")
					flag = false
				} else {
					log.Printf("Failed, url:%s; status:%d; text:%s", resp.URL, resp.Status, resp.StatusText)
					writefile("./failed.txt", "Url:"+resp.URL+" Status:"+statusMap[int(resp.Status)]+" Text:"+resp.StatusText+"\r\n")
					flag = true //这里定义了一个全局变量来对fail的url进行截图
				}
			}
		}
		// other needed network Event
	})
}

func writefile(filename, msg string) {
	f, err := os.OpenFile(filename, os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644) //表示最佳的方式打开文件，如果不存在就创建，打开的模式是可读可写，权限是644
	if err != nil {
		log.Fatal(err)
	}
	f.WriteString(msg)
	f.Close()
}
```

2、接口函数实现了，接下来就要处理高并发了，毕竟有100万个URL，就必须要有足够多的协程才能很好的实现测试。在网上查看大佬们的博客，最终选择了采用job队列+工作池的方法来实现高并发。

3、工作池实现

- 首先，定义一个job的接口，具体内容由具体job实现：

  ```go
  type Job interface {
      Do()
  }
  ```

- 定义Worker类型，给每一个Worker建立一条Job队列，用来接收Job；再定义一个新建Worker的函数；对每一个Worker而言，前一个任务执行完成之后，再读取新的任务并执行。

  ```go
  type Worker struct {
      JobQueue chan Job
  }
  
  func NewWorker() Worker {
      return Worker{JobQueue: make(chan Job)}
  }
  func (w Worker) Run(wq chan chan Job) {
      go func() {
          for {
              wq <- w.JobQueue
              select {
              case job := <-w.JobQueue:
                  job.Do()
              }
          }
      }()
  }
  ```

- 定义一个工作池：新建一个工作池；循环获取可用的worker，往worker中写入job。

  ```go
  type WorkerPool struct {
      workerlen   int
      JobQueue    chan Job
      WorkerQueue chan chan Job
  }
  
  func NewWorkerPool(workerlen int) *WorkerPool {
      return &WorkerPool{
          workerlen:   workerlen,
          JobQueue:    make(chan Job),
          WorkerQueue: make(chan chan Job, workerlen),
      }
  }
  func (wp *WorkerPool) Run() {
      fmt.Println("初始化worker")
      //初始化worker
      for i := 0; i < wp.workerlen; i++ {
          worker := NewWorker()
          worker.Run(wp.WorkerQueue)
      }
      // 循环获取可用的worker,往worker中写job
      go func() {
          for {
              select {
              case job := <-wp.JobQueue:
                  worker := <-wp.WorkerQueue
                  worker <- job
              }
          }
      }()
  }
  ```

- 在主函数中往Job队列送入任务。

  ```go
  datanum := len(urls)
  	// datanum := 100 * 100 //* 100 * 100
  	go func() {
  		for i := 0; i < datanum; i++ {
  			sc := &Score{Num: i, Url: urls[i]}
  			p.JobQueue <- sc
  			time.Sleep(2 * time.Second) // 700 * time.Millisecond
  		}
  	}()
  ```

- 整个job队列+工作池的工作过程为

```go
/*
一个任务的执行过程如下
JobQueue <- work  新任务入队
job := <-JobQueue: 调度中心收到任务
jobChannel := <-d.WorkerPool 从工作者池取到一个工作者
jobChannel <- job 任务给到工作者
job := <-w.JobChannel 工作者取出任务
{{1}} 执行任务
w.WorkerPool <- w.JobChannel 工作者在放回工作者池
*/
```

### 四、源代码

**main.go**

```go
package main

import (
	"context"
	"fmt"
	"log"
	"runtime"
	"time"
	"strings"
	"io/ioutil"
	"os"

	"github.com/chromedp/cdproto/network"
	"github.com/chromedp/chromedp"
)

type Score struct {
	Num int
	Url string
}

var statusMap map[int]string = map[int]string{
	301: "301",
	302: "302",
	304: "304",
	400: "400",
	401: "401",
	403: "403",
	404: "404",
	405: "405",
	408: "408",
	500: "500",
	502: "502",
	503: "503",
	504: "504",
}

var flag bool

func (s *Score) Do() {
	fmt.Printf("num:%d  url:%s\n", s.Num, s.Url)
	// fmt.Println("num:", s.Url)
	// opts := chromedp.ExecPath("C:\\Users\\10049\\AppData\\Local\\Google\\Chrome\\Application\\chrome.exe")

	// allocCtx, cancel := chromedp.NewExecAllocator(context.Background(), opts)
	// defer cancel()

	// taskCtx, cancel := chromedp.NewContext(allocCtx, chromedp.WithLogf(log.Printf))
	// defer cancel()

	var buf []byte
	// ch := make(chan []byte, 1024)
	// create chrome instance
	taskCtx, cancel := chromedp.NewContext(
		context.Background(),
		chromedp.WithLogf(log.Printf),
	)
	defer cancel()

	// create a timeout
	taskCtx, cancel = context.WithTimeout(taskCtx, 30*time.Second)
	defer cancel()

	// ensure that the browser process is started
	if err := chromedp.Run(taskCtx); err != nil {
		panic(err)
		// fmt.Println(err)
	}

	// listen network event
	listenForNetworkEvent(taskCtx) //,ch

	chromedp.Run(taskCtx,
		network.Enable(),
		chromedp.Navigate(s.Url),
		// chromedp.WaitVisible(`#doc-info`), //, chromedp.BySearch
		chromedp.CaptureScreenshot(&buf),
	)

	log.Println("hello")
	if flag == true{
		pngname := strings.Replace(s.Url,".","1",-1)
		pngname = strings.Replace(pngname,"http://","1",-1)
		log.Println(buf)
		if err1 := ioutil.WriteFile(pngname+".png", buf, 0644); err1 != nil {
			log.Fatal(err1)
		}
	}

	time.Sleep(30 * time.Second)
}

func listenForNetworkEvent(ctx context.Context) { //, ch chan []byte
	count := 0
	chromedp.ListenTarget(ctx, func(ev interface{}) {

		switch ev := ev.(type) {

		case *network.EventResponseReceived:
			count++
			// log.Println("count=", count)
			resp := ev.Response
			if count < 2 {
				if resp.Status == 200 {
					log.Printf("Success, url:%s; status:%d; text:%s", resp.URL, resp.Status, resp.StatusText)
					writefile("./success.txt", "Url:"+resp.URL+" Status:200"+" Text:"+resp.StatusText+"\r\n")
					flag = false
				} else {
					log.Printf("Failed, url:%s; status:%d; text:%s", resp.URL, resp.Status, resp.StatusText)
					writefile("./failed.txt", "Url:"+resp.URL+" Status:"+statusMap[int(resp.Status)]+" Text:"+resp.StatusText+"\r\n")
					flag = true
				}
			}
		}
		// other needed network Event
	})
}

func writefile(filename, msg string) {
	f, err := os.OpenFile(filename, os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644) //表示最佳的方式打开文件，如果不存在就创建，打开的模式是可读可写，权限是644
	if err != nil {
		log.Fatal(err)
	}
	f.WriteString(msg)
	f.Close()
}

func main() {
	num := 500
	// debug.SetMaxThreads(num + 1000) //设置最大线程数
	// 注册工作池，传入任务
	// 参数1 worker并发个数
	p := NewWorkerPool(num)
	p.Run()

	// 读取文件，以切片的形式返回 URLS
	txtpath := "F:\\GO_code\\test_xlsx\\top-2m.xlsx"
	urls := loadfile(txtpath)
	log.Printf("urls[0]:%s  type:%T   urls'length:%d", urls[0], urls[0], len(urls)) // print the content as a 'string'
	datanum := len(urls)
	// datanum := 100 * 100 //* 100 * 100
	go func() {
		for i := 0; i < datanum; i++ {
			sc := &Score{Num: i, Url: urls[i]}
			p.JobQueue <- sc
			time.Sleep(100 * time.Millisecond) // 700 * time.Millisecond
		}
	}()

	for {
		fmt.Println("runtime.NumGoroutine() :", runtime.NumGoroutine())
		time.Sleep(2 * time.Second)
	}

}
```

**job.go**

```go
package main

type Job interface {
    Do()
}
```

**worker.go**

```go
package main

type Worker struct {
    JobQueue chan Job
}

func NewWorker() Worker {
    return Worker{JobQueue: make(chan Job)}
}
func (w Worker) Run(wq chan chan Job) {
    go func() {
        for {
            wq <- w.JobQueue
            select {
            case job := <-w.JobQueue:
                job.Do()
            }
        }
    }()
}
```

**workerpool.go**

```go
package main

import "fmt"

type WorkerPool struct {
    workerlen   int
    JobQueue    chan Job
    WorkerQueue chan chan Job
}

func NewWorkerPool(workerlen int) *WorkerPool {
    return &WorkerPool{
        workerlen:   workerlen,
        JobQueue:    make(chan Job),
        WorkerQueue: make(chan chan Job, workerlen),
    }
}
func (wp *WorkerPool) Run() {
    fmt.Println("初始化worker")
    //初始化worker
    for i := 0; i < wp.workerlen; i++ {
        worker := NewWorker()
        worker.Run(wp.WorkerQueue)
    }
    // 循环获取可用的worker,往worker中写job
    go func() {
        for {
            select {
            case job := <-wp.JobQueue:
                worker := <-wp.WorkerQueue
                worker <- job
            }
        }
    }()
}
```

**readfile.go**

```go
package main


import (
    "fmt"
    "github.com/360EntSecGroup-Skylar/excelize"
)

func loadfile(txtpath string) []string {
    // f, err := excelize.OpenFile("F:\\GO_code\\test_xlsx\\top-2m.xlsx")
    f, err := excelize.OpenFile(txtpath)
    if err != nil {
        fmt.Println(err)
    }
    
    rows, err := f.GetRows("top-1m")
    var str []string
    // fmt.Printf(string(rows[0][0]))
    for _, row := range rows {
        s := "http://" + row[1]
        str = append(str, s)
    }
    return str
}
```

### 五、下一步工作

当前虽然在代码里设置了worker的数目，但是当请求太多，在worker的上一个任务还没执行完的时候，下一个请求已经到了，这样就会引起阻塞，我们当前是在请求里加了延时，来控制请求的速度，但是这样显然还是能再优化的。

