**Working pool**


**Using buffered channel with known number of tasks**

Based on buffered channel.
The solution is based on the Go feature: writing to buffered channel is blocked once it's full.

So, given the limit of simultaneously working tasks **numWorkers** we create a buffer limited by the number (**numWorkers**).

```
	numWorkers := 10
	totalTasksNum := 1_000

	tasks := make(chan Task, totalTasksNum) // wide input
	res := make(chan string, totalTasksNum) // wide output
	for i := 0; i < numWorkers; i++ {       // narrow processors
		go func(workerNum int) {
			for task := range tasks {
				res <- fmt.Sprintf("worker #%d, task #%d\n", workerNum, task.data)
				time.Sleep(10 * time.Millisecond) // emulate long running task
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

Or, we could engage WorkGroup and eliminate knowledge of number of tasks at received end:
```
 	numWorkers := 10
	totalTasksNum := 1_000
	wg := sync.WaitGroup{}

	tasks := make(chan Task, totalTasksNum) // wide input
	res := make(chan string, totalTasksNum) // wide output
	go func() {
		for i := 0; i < numWorkers; i++ { // narrow processors
			wg.Add(1)
			go func(workerNum int) {
				defer wg.Done()
				for task := range tasks {
					res <- fmt.Sprintf("worker #%d, task #%d\n", workerNum, task.data)
					time.Sleep(10 * time.Millisecond) // emulate long running task
				}
			}(i)
		}
		wg.Wait()
		close(res)
	}()

	for i := 0; i < totalTasksNum; i++ {
		tasks <- Task{i}
	}
	close(tasks)

	for r := range res {
		fmt.Println(r)
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

func main() {
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
					time.Sleep(10 * time.Millisecond) // emulate long running task
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


**Check if channel is closed**

Reading from a closed channel succeeds immediately, returning the zero value of the underlying type. The optional second return value is true if the value received was delivered by a successful send operation to the channel, or false if it was a zero value generated because the channel is closed and empty.

```
_, ok := <-jobs
    fmt.Println("received more jobs:", ok)
```


**Timers**
Timers represent a single event in the future.

```
    timer1 := time.NewTimer(2 * time.Second)
    <-timer1.C
    timer1.Stop()
```

**Tickers**
Tickers are for when you want to do something repeatedly at regular intervals.
```
ticker := time.NewTicker(500 * time.Millisecond)
for {
            select {
            case <-done:
                return
            case t := <-ticker.C:
                fmt.Println("Tick at", t)
            }
	    ticker.Stop()
}
```

**Rate limiting**

```
    requests := make(chan int, 5)
    for i := 1; i <= 5; i++ {
        requests <- i
    }
    close(requests)

    limiter := time.Tick(200 * time.Millisecond)

    for req := range requests {
        <-limiter
        fmt.Println("request", req, time.Now())
    }
```


**Goroutine memary pattern**

Channel-based approach aligns with Go’s ideas of sharing memory by communicating and having each piece of data owned by exactly 1 goroutine.


