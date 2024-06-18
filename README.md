Working pool


**Using buffered channel**

Based on buffered channel
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
		time.Sleep(5 * time.Millisecond)
	}
```
