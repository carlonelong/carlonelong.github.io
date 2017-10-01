---
layout: article-detail
category: nsq
author: 龙飞 
title: NSQ Channel 
---
在[NSQ Topic](../../09/17/nsq-topic)一文中提到了，一个topic下面可以注册多个channel，每个channel都有完整的数据拷贝，但是每个channel的单条信息只能被某一个用户接收。现在就来看看channel是如何实现的。
## channel的定义
```go
type Channel struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms
	requeueCount uint64
	messageCount uint64
	timeoutCount uint64

	sync.RWMutex

	topicName string
	name      string
	ctx       *context

	backend BackendQueue

	memoryMsgChan chan *Message
	exitFlag      int32
	exitMutex     sync.RWMutex

	// state tracking
	clients        map[int64]Consumer
	paused         int32
	ephemeral      bool
	deleteCallback func(*Channel)
	deleter        sync.Once

	// Stats tracking
	e2eProcessingLatencyStream *quantile.Quantile

	// TODO: these can be DRYd up
	deferredMessages map[MessageID]*pqueue.Item
	deferredPQ       pqueue.PriorityQueue
	deferredMutex    sync.Mutex
	inFlightMessages map[MessageID]*Message
	inFlightPQ       inFlightPqueue
	inFlightMutex    sync.Mutex
}
```
除了一些居家必备良药，如topicName，name等之外，channel跟topic一样，也有一个backend的磁盘队列。为什么topic和channel都需要队列呢？想象一下当topic下没有channel和topic下有多个channel的情况下消息都堆积在哪就豁然开朗了。

与topic相似，channel也有一些用来控制生命周期的变量，比如paused, ephemeral, deleteCallback。此外因为channel是直接连接到消费者，所以有一个clients字典来记录这些consumer。

然而这些跟消息分发半毛钱关系都没有，我们重点关注的是最后几个成员：inFlight\*和deferred\*。顾名思义，前者是正在分发过程中的消息的相关信息，后者则是被延迟发送消息的信息。

接下来来看看整个流程是怎么样的。
## Channel消息分发
### Consumer注册
```go
func (c *Channel) AddClient(clientID int64, client Consumer) {
	c.Lock()
	defer c.Unlock()

	_, ok := c.clients[clientID]
	if ok {
		return
	}
	c.clients[clientID] = client
}
```
很简单，就是把新来的client加入到前面的clients中。不过有个问题，就是如果用户注册的channel不存在呢？回头看看topic就知道了，topic会为不存在的channel_name新创建一个channel。

```go
func NewChannel(topicName string, channelName string, ctx *context,
	deleteCallback func(*Channel)) *Channel {

	c := &Channel{
		topicName:      topicName,
		name:           channelName,
		memoryMsgChan:  make(chan *Message, ctx.nsqd.getOpts().MemQueueSize),
		clients:        make(map[int64]Consumer),
		deleteCallback: deleteCallback,
		ctx:            ctx,
	}
	if len(ctx.nsqd.getOpts().E2EProcessingLatencyPercentiles) > 0 {
		c.e2eProcessingLatencyStream = quantile.New(
			ctx.nsqd.getOpts().E2EProcessingLatencyWindowTime,
			ctx.nsqd.getOpts().E2EProcessingLatencyPercentiles,
		)
	}

	c.initPQ()

	if strings.HasSuffix(channelName, "#ephemeral") {
		c.ephemeral = true
		c.backend = newDummyBackendQueue()
	} else {
		dqLogf := func(level diskqueue.LogLevel, f string, args ...interface{}) {
			opts := ctx.nsqd.getOpts()
			lg.Logf(opts.Logger, opts.logLevel, lg.LogLevel(level), f, args...)
		}
		// backend names, for uniqueness, automatically include the topic...
		backendName := getBackendName(topicName, channelName)
		c.backend = diskqueue.New(
			backendName,
			ctx.nsqd.getOpts().DataPath,
			ctx.nsqd.getOpts().MaxBytesPerFile,
			int32(minValidMsgLength),
			int32(ctx.nsqd.getOpts().MaxMsgSize)+minValidMsgLength,
			ctx.nsqd.getOpts().SyncEvery,
			ctx.nsqd.getOpts().SyncTimeout,
			dqLogf,
		)
	}

	c.ctx.nsqd.Notify(c)

	return c
}
```
类似于topic，channel也会根据是否ephemeral来指定对应的backend。

值得注意的是，NSQ在初始化channel的时候也初始化了前面提到的inFlight*和deferred*变量。
```go
func (c *Channel) initPQ() {
	pqSize := int(math.Max(1, float64(c.ctx.nsqd.getOpts().MemQueueSize)/10))

	c.inFlightMessages = make(map[MessageID]*Message)
	c.deferredMessages = make(map[MessageID]*pqueue.Item)

	c.inFlightMutex.Lock()
	c.inFlightPQ = newInFlightPqueue(pqSize)
	c.inFlightMutex.Unlock()

	c.deferredMutex.Lock()
	c.deferredPQ = pqueue.New(pqSize)
	c.deferredMutex.Unlock()
}
```
代码很清晰，但有两个问题：
1. 为什么需要两个变量*Messages和*PQ呢？
2. 为什么inFlightPQ和deferredPQ使用了不一样的数据结构，区别是什么？

我们接着往下面看。
### 分发消息到clients
还记得topic分发消息的方法（messagePump）吗？当消息到来的时候，会被拷贝分发到每个channel。如果消息不需要延迟发送，就调用PutMessage函数，否则调用PutMessageDeferred函数。

#### 即时消息的转发
根据前面的分析我们大致能猜到，PutMessage操作inFlightMessages。跟踪一下PutMessage的代码：
```go
func (c *Channel) PutMessage(m *Message) error {
	c.RLock()
	defer c.RUnlock()
	if c.Exiting() {
		return errors.New("exiting")
	}
	err := c.put(m)
	if err != nil {
		return err
	}
	atomic.AddUint64(&c.messageCount, 1)
	return nil
}
func (c *Channel) put(m *Message) error {
	select {
	case c.memoryMsgChan <- m:
	default:
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, c.backend)
		bufferPoolPut(b)
		c.ctx.nsqd.SetHealth(err)
		if err != nil {
			c.ctx.nsqd.logf(LOG_ERROR, "CHANNEL(%s): failed to write message to backend - %s",
				c.name, err)
			return err
		}
	}
	return nil
}
```
这部分跟topic分发消息几乎一样，但是消息写入memoryMsgChan之后就不知去向了。通过全局搜索可以发现在protocol_v2.go里面有跟topic又几乎一致的messagePump函数：
```go
//代码很长，只列出处理消息的部分
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
	//......
	for {
		select {
		case b := <-backendMsgChan:
			if sampleRate > 0 && rand.Int31n(100) > sampleRate {
				continue
			}

			msg, err := decodeMessage(b)
			if err != nil {
				p.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
				continue
			}
			msg.Attempts++

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage()
			err = p.SendMessage(client, msg, &buf)
			if err != nil {
				goto exit
			}
			flushed = false
		case msg := <-memoryMsgChan:
			if sampleRate > 0 && rand.Int31n(100) > sampleRate {
				continue
			}
			msg.Attempts++

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage()
			err = p.SendMessage(client, msg, &buf)
			if err != nil {
				goto exit
			}
			flushed = false
		case <-client.ExitChan:
			goto exit
		}
	}
}
```
可见，到这一步，无论是memory message还是backend message，都已经发送到client了，但是似乎并未用到inFlightMessages和inFlightPQ。不要着急，看看StartInFlightTimeout：
```go
func (c *Channel) StartInFlightTimeout(msg *Message, clientID int64, timeout time.Duration) error {
	now := time.Now()
	msg.clientID = clientID
	msg.deliveryTS = now
	msg.pri = now.Add(timeout).UnixNano()
	err := c.pushInFlightMessage(msg)
	if err != nil {
		return err
	}
	c.addToInFlightPQ(msg)
	return nil
}
```
这里就很清楚了，会把发送到客户端的信息存到inFlightMessages和inFlightPQ。为什么？还记得NSQ的GUARANTEE-“messages are delivered at least once”么？没错，这是为了解决发送失败的问题。当一条消息被发送给client之后，如果在指定时间内仍没有收到客户端的响应的话，NSQ会把本次发送当做失败处理，然后把消息重新放回队列让别的用户有机会读到。
```go
func (c *Channel) processInFlightQueue(t int64) bool {
	c.exitMutex.RLock()
	defer c.exitMutex.RUnlock()

	if c.Exiting() {
		return false
	}

	dirty := false
	for {
		c.inFlightMutex.Lock()
		msg, _ := c.inFlightPQ.PeekAndShift(t)
		c.inFlightMutex.Unlock()

		if msg == nil {
			goto exit
		}
		dirty = true

		_, err := c.popInFlightMessage(msg.clientID, msg.ID)
		if err != nil {
			goto exit
		}
		atomic.AddUint64(&c.timeoutCount, 1)
		c.RLock()
		client, ok := c.clients[msg.clientID]
		c.RUnlock()
		if ok {
			client.TimedOutMessage()
		}
		c.put(msg)
	}

exit:
	return dirty
}
```
除此之外，client的一些请求，如FINISH、REQ、TOUCH等指令都会导致inFlightMessages和inFlightPQ的变化（删除或重排）。
到这里，可以解释为什么用使用PQ和map两种数据结构来保存message了：前者用时间为序，保存时间和message的对应关系，每次取最快要过期的消息，保证遍历能停在第一条未过期的消息处，减少不必要操作，提高效率；后者保存messageID和message对应关系，保证client操作（总是传入messageID）能在O(1)时间内找到message本身。

同样这也是为什么消息不保证顺序的一个重要原因。

还有几个小细节值得思考的：
1. timeout时间是client设置的，因为每个client处于不一样的环境，所以这个选项留给了client自己配置。如果发现超时时间到了，消息还没处理完，有两个解决办法，TOUCH和设置更长的timeout。
2. 同一channel下每个用户收到的message是各不一样的，分配的过程就是go里面多个consumer listen同一个chan的过程。每个client可以设置一个sampleRate来决定有多大可能性接收发来的消息。
3. 每条消息会记录一个Attempt值，表示发送的次数。client可以根据这个值做一些处理，比如发送10次强制标记为完成。 NSQ是不会主动删除消息的。

#### 延时消息的转发
前面已经大致描述了即时消息的转发流程，延时消息这里就简略一点。直接上代码。
```go
func (c *Channel) PutMessageDeferred(msg *Message, timeout time.Duration) {
	atomic.AddUint64(&c.messageCount, 1)
	c.StartDeferredTimeout(msg, timeout)
}
func (c *Channel) StartDeferredTimeout(msg *Message, timeout time.Duration) error {
	absTs := time.Now().Add(timeout).UnixNano()
	item := &pqueue.Item{Value: msg, Priority: absTs}
	err := c.pushDeferredMessage(item)
	if err != nil {
		return err
	}
	c.addToDeferredPQ(item)
	return nil
}
func (c *Channel) processDeferredQueue(t int64) bool {
	c.exitMutex.RLock()
	defer c.exitMutex.RUnlock()

	if c.Exiting() {
		return false
	}

	dirty := false
	for {
		c.deferredMutex.Lock()
		item, _ := c.deferredPQ.PeekAndShift(t)
		c.deferredMutex.Unlock()

		if item == nil {
			goto exit
		}
		dirty = true

		msg := item.Value.(*Message)
		_, err := c.popDeferredMessage(msg.ID)
		if err != nil {
			goto exit
		}
		c.put(msg)
	}

exit:
	return dirty
}
```
这里就相对简单了，只要把消息存入deferredPQ，时间到了之后取出来，放入inFlightPQ即可。
那么为何deferredPQ没有使用和inFlightPQ一样的数据结构呢？说实话不是很明白，我的看法是，因为NSQ主要消耗资源是内存，inFlight message比deferred message存的数据较多，所以要区分开，尽量减小内存占用。但是一般情况下deferred message不会太多吧？
### 消息清理
某些情况下，channel关闭，需要处理剩下的消息，除非删除channel，否则都会会写到磁盘。
```go
func (c *Channel) flush() error {
	var msgBuf bytes.Buffer

	if len(c.memoryMsgChan) > 0 || len(c.inFlightMessages) > 0 || len(c.deferredMessages) > 0 {
		c.ctx.nsqd.logf(LOG_INFO, "CHANNEL(%s): flushing %d memory %d in-flight %d deferred messages to backend",
			c.name, len(c.memoryMsgChan), len(c.inFlightMessages), len(c.deferredMessages))
	}

	for {
		select {
		case msg := <-c.memoryMsgChan:
			err := writeMessageToBackend(&msgBuf, msg, c.backend)
			if err != nil {
				c.ctx.nsqd.logf(LOG_ERROR, "failed to write message to backend - %s", err)
			}
		default:
			goto finish
		}
	}

finish:
	c.inFlightMutex.Lock()
	for _, msg := range c.inFlightMessages {
		err := writeMessageToBackend(&msgBuf, msg, c.backend)
		if err != nil {
			c.ctx.nsqd.logf(LOG_ERROR, "failed to write message to backend - %s", err)
		}
	}
	c.inFlightMutex.Unlock()

	c.deferredMutex.Lock()
	for _, item := range c.deferredMessages {
		msg := item.Value.(*Message)
		err := writeMessageToBackend(&msgBuf, msg, c.backend)
		if err != nil {
			c.ctx.nsqd.logf(LOG_ERROR, "failed to write message to backend - %s", err)
		}
	}
	c.deferredMutex.Unlock()

	return nil
}
```
代码比较简单，就不赘述了。
## 小结
本文**详细**描述了NSQ的channel，大致总结成以下几点
1. channel的每一条消息至少发送一次，至多发送给一个consumer。
2. channel的消息是无序的。
3. NSQ使用priority queue和map来提高查找和使用消息的效率，但是可能会有潜在冗余和一致性问题。
4. client可以通过设置timeout、sampleRate、maxAttemptCount等参数来过滤接收到的消息。
5. 写文章好累。
