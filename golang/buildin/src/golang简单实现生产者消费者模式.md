## golang简单实现生产者消费者模式

生产者/消费者模型主要通过平衡生产线程和消费线程的工作能力来提高程序的整体处理数据的速度。简单来说，就是生产者生产一些数据，放到成果队列中，同时消费者从成果队列中来取数据。这样生产和消费就变成了异步过程了。当成果队列中没有数据时，消费者就进入饥饿的等待状态中；而当成果队列中数据已满时，生产者则面临因产品挤压导致CPU被剥夺的问题。

## golang实现

通过带缓冲的channel实现，channel的缓冲大小，可作为队列长度的缓冲大小。

**demo**

```
package main

import (
	"fmt"
	"time"
)

type Event struct {
	queue   chan int      // channel
	timeout time.Duration // the timeout of produce send data
	handle  func(v int)   // consume callback
}

// NewEvent new queue event
func NewEvent(buffer int, t time.Duration, f func(v int)) *Event {
	return &Event{
		queue:   make(chan int, buffer),
		timeout: t,
		handle:  f,
	}
}

// Producer produce to queue
func (e Event) Producer(num int) {
	select {
	case e.queue <- num:
	case <-time.After(e.timeout):
	}
}

// Consumer consume from queue
func (e Event) Consumer() {
	for v := range e.queue {
		e.handle(v)
	}
}

func main() {
	e := NewEvent(100, time.Second, func(v int) {
		fmt.Println("this is num : ", v)
	})
	go func() {
		for i := 0; i < 100; i++ {
			e.Producer(i)
		}
	}()
	go func() {
		e.Consumer()
	}()
	time.Sleep(5 * time.Second)
}
```

## 参考
* 《Go语言高级编程》
