**Working pool**


**Using buffered channel**

Based on buffered channel.
The solution is based on the Go feature: writing to buffered channel is blocked once it's full.

So, given the limit of simultaneously working tasks **numWorkers** we create a buffer limited by the number (**numWorkers**).

```
	numWorkers := 10
	totalTasksNum := 1_000
	
	tasks := make(chan Task) // , 1_000)
	res := make(chan string, totalTasksNum)
	for i := 0; i < numWorkers; i++ {
		go func(workerNum int) {
			for task := range tasks {
				res <- fmt.Sprintf("worker #%d, task #%d\n", workerNum, task.data)
			}
		}(i)
	}
	for i := 0; i < totalTasksNum; i++ {
		tasks <- Task{i}
	}
	close(tasks)
	
	for i := 0; i < totalTasksNum; i++ {
		fmt.Println(<-res)
	}
```

**Using Semaphore**

As the previous one, the solution is based on the same Go feature: writing to buffered channel is blocked once it's full.
But this time the buffered channel acts as Semaphore: buffer contains number of "locks".
```
type Semaphore struct {
	ch chan struct{}
}

func (s *Semaphore) Acquire() {
	s.ch <- struct{}{}
}

func (s *Semaphore) Release() {
	<-s.ch
}

	numTasks := 1_000
	throttle := 10
	sem := &Semaphore{ch: make(chan struct{}, throttle)}
	in := make(chan int)
	out := make(chan int)
	go func() {
		for i := 0; i < numTasks; i++ {
			in <- i
		}
		close(in)
	}()
	wg := sync.WaitGroup{}
	go func() {
		for i := 0; i < numTasks; i++ {
			wg.Add(1)
			go func() {
				defer wg.Done()
				for t := range in {
					sem.Acquire()
					out <- t
					sem.Release()
				}
			}()
		}
		wg.Wait()
		close(out)
	}()
	for r := range out {
		s := fmt.Sprintf("*%d*\n", r)
		fmt.Print(s)
	}
```


**Timeouts**

The design template with timeout engages **case** operator.
```
 c1 := make(chan string, 1)
    go func() {
        time.Sleep(2 * time.Second)
        c1 <- "result 1"
    }()
```





