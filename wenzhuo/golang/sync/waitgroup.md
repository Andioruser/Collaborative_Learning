# WaitGroup 


## WaitGroup的基本用法
```golang
package main
import (
	"errors"
	"fmt"
	"sync"
)

func worker(id int,	wg *sync.WaitGroup, err *error) {
	defer wg.Done()

	if id == 2 {
		*err = errors.New("发送了一个错误")	//模拟可能发生错误的工作
		return
	}
	
	fmt.Println("Worker", id, "已完成")
}

func main() {
	numWorkers := 3
	var wg sync.WaitGroup
	var err error

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go worker(i, &wg, &err)
	}

	wg.Wait()	//等待所有工作完成

	if err != nil {
		fmt.Println("发送了一个错误:", err)
	} else {
		fmt.Println("所有工作完成")
	}
}

``` 


[博客链接](https://blog.frognew.com/2023/07/concurrency-in-go-channels-and-waitgroups.html)