

在这个示例中，我们有多个workder，由worker函数表示。 

每个worker执行一些工作，并将结果发送到Channel ch。   

主Goroutine然后使用sync.WaitGroup等待所有worker完成，并关闭Channel。    

最后，主Goroutine从Channel接收结果并进行处理。
```golang
package main

import (
	"fmt"
	"sync"
)

func worker(id int, ch chan int, wg *sync.WaitGroup) {
	defer wg.Done()

	result := id * 2	//执行一些工作

	ch <- result	//将结果发送到channel
}

func main() {
	numWorkers := 3
	ch := make(chan int, numWorkers)	//创建一个带缓冲带channel
	var wg sync.WaitGroup

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go worker(i, ch, &wg)
	}

	go func() {
		wg.Wait()
		close(ch)	//在所有的worker完成后关闭channel
	}()

	for result := range ch {
		fmt.Println(result)	//处理结果	
	}
}

``` 


Channel的主要优势之一是它们能够同步Goroutines。通过使用Channel，您可以协调多个Goroutines的执行，确保它们完成其工作。

例如，考虑一个场景，您有多个Goroutines执行独立的任务，但是只有在所有Goroutines都完成后才希望处理结果。您可以使用Channel通过创建一个Channel，并让每个Goroutine在完成其工作时向Channel发送一个值来实现此同步。然后，您可以使用一个单独的Goroutine或主Goroutine来等待从Channel接收到的所有预期值。

```go
package main

import (
	"fmt"
	"sync"
)

func squreWorker(id int, input <- chan int, output chan <- int, wg *sync.WaitGroup) {
	defer wg.Done()
	for num := range input {
		squre := num * num
		output <- squre
	}
}

func main() {
	input := make(chan int)
	output := make(chan int)

	var wg sync.WaitGroup
	wg.Add(2)

	go squreWorker(1, input, output, &wg)
	go squreWorker(2, input, output, &wg)

	go func() {
		defer close(input)
		for i := 1; i <= 5; i++ {
			input <- i
		}
	}()

	go func() {
		defer close(output)
		wg.Wait()
	}()

	for square := range output {
		fmt.Println(square)
	}

	fmt.Println("All values processed")
}

```

[博客链接](https://blog.frognew.com/2023/07/concurrency-in-go-channels-and-waitgroups.html)