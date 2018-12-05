# Go 简易版协程池

近今天在研究goroutine协程，于是乎就想实现一个可控制协程大小的协程池，
在网上看了文章:
[百万级并发协程池](https://mp.weixin.qq.com/s?__biz=MjM5OTcxMzE0MQ==&mid=2653369770&idx=1&sn=044be64c577a11a9a13447b373e80082&chksm=bce4d5b08b935ca6ad59abb5cc733a341a5126fefc0e6600bd61c959969c5f77c95fbfb909e3&mpshare=1&scene=1&srcid=1010dpu0DlPHi6y1YmrixifX#rd) 
，从中学到了些东西，于是乎自己写写练练手。

## 整体思路
把任务task丢到taskQueue队列里面，再由这个队列去分发任务，从池子里面取出一个worker来消费
该task,待task消费完毕后，再把worker重新放到池子里面，等待下一次的消费调用。

## Task interface
```
type Task interface {
	Consume() error
}
```
先定义一个任务接口类型，类型有一个方法：```consume```,用来处理task的消费。

## Worker模型
```
type worker struct {
	workerPool  chan chan Task // worker pool
	taskChannel chan Task      // task chan
	quit        chan bool
}

func NewWorker(workerPool chan chan Task) worker {
	return worker{
		workerPool:  workerPool,
		taskChannel: make(chan Task),
		quit:        make(chan bool),
	}
}
```
worker定义为一个结构体，属性有 ```workerPool```，这个来表示当前```Worker```是属于哪个
协程池的；```taskChannel```是一个Task类型的通道chan,用来传递消息task的。
worker消费task逻辑：
```
func (w *worker) Start() {
	go func() {
		for {
			// consume done ,then worker reenter workerPool
			w.workerPool <- w.taskChannel
			select {
			case task := <-w.taskChannel:
				// received a work request and consume it
				if err := task.Consume(); err != nil {
					log.Printf("Task Consume fail: %v", err.Error())
				}
			case <-w.quit:
				return
			}
		}
	}()
}
```
每个worker启动一个协程(goroutine)，然后再协程里面做个死循环，```worker```
把自己的```taskChannel```放到协程池里面，然后从自己的task任务通道里面取出
消息，进行消费，也就是执行```task.Consume()```方法。

### Pool模型
```
type pool struct {
	workerPool chan chan Task // worker pool
	taskQueue  chan Task      // work queue
	maxWorkers int            // worker pool max worker count
}

func NewPool(maxWorkers, maxQueues int) (*pool, error) {
	p := &pool{
		workerPool: make(chan chan Task, maxWorkers),
		taskQueue:  make(chan Task, maxQueues),
		maxWorkers: maxWorkers,
	}

	return p, nil
}
```
pool模型主要有三个字段：```workerPool```用来存放```worker```的```taskChannel```
;```taskQueue```是个task类型的chann,用来接收业务逻辑的任务task;```maxWorkers```表示当前
pool能控制的worker最大数。
创建Pool的时候，需要给```workerPool```指定大小的workers;任务队列```taskQueue```
也需要指定队列缓冲大小。
启动Pool的逻辑就比较简单了，先初始化该Pool的所有workers,然后分发消息，也就是从任务
队列```taskQueue```里面取出消息，然后从协程池里面取出一个```worker```,然后当前没有
可用的```worker```的话，会堵塞，直到有可用的worker为止，把该任务丢给当前取出的worker
进行消费处理，当worker消费完毕后，worker会从新放进池子里，等待下一次消费，代码如下：
```
// init pool workers
func (p *pool) initWorkers() {
	for i := 0; i < p.maxWorkers; i++ {
		w := NewWorker(p.workerPool)
		w.Start()
	}
}

// dispatch task
func (p *pool) dispatch() {
	for {
		select {
		case task := <-p.taskQueue:
			// a task request has been received
			go func(task Task) {
				// pop a task channel from pool
				// this will block until a worker is idle
				taskChannel := <-p.workerPool
				// push task to task channel
				taskChannel <- task
			}(task)
		}
	}
}
```

## 使用实例代码如下
```
type MyTask struct {
	Name string
}

func (mt *MyTask) Consume() error {
	fmt.Printf("[Task: %s] process done\n", mt.Name)
	return nil
}

func main() {
   // 控制协程池大小为100万个worker
	maxWorkers := 1000000
	// 任务队列缓冲为1000
	maxQueues := 1000
	pool, _ := gopool.NewPool(maxWorkers, maxQueues)

	pool.Run()
	fmt.Println("Starting Worker Pool test...")

	for i := 0; i <= 1000000; i++ {
		myTask := &MyTask{Name: fmt.Sprintf("Test: %d", i)}
		pool.Push(myTask)
	}

	time.Sleep(time.Second * 30)
}
```
以上就是该协程池的所有逻辑了，上面也就是大概实现了一个协程池，基本可以支持百万级的并发
，逻辑还是比较简单的。该协程池还有很多待改进的方面：
- worker数据动态可控制，支持扩容和缩容。
- 支持worker的有效期，worker可以根据有消费自动销毁
- pool能够动态的停止退出等等。

有兴趣的小伙伴可以在这个Demo的基础之上继续改造实现下，逻辑应该不是很难
，可以在pool里面定义一个workers切片来动态存储workers,设置一个chan sig struct{}
来标记pool退出信号。
整个Demo的完整代码：
[Demo完整代码](https://github.com/binkzhao/gopool) 






