---
layout: article-detail
category: nsq
author: 龙飞 
title: DiskQueue 
---
在[NSQ Topic](../../09/17/nsq-topic)一文中提到了，当消息堆积太多时，NSQ会把部分消息写入一个backend。这个backend实际是一个叫做diskQueue的数据结构。本文主要分析该结构。

同样，我们带着以下几个疑问来阅读代码。
1. NSQ会在什么情况下把数据写入/读出diskQueue?
2. diskQueue的数据是怎么存储的，如何保证数据正确读写？

## diskQueue结构
照惯例先看一下diskQueue的数据结构
```go
type diskQueue struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms

	// run-time state (also persisted to disk)
	readPos      int64
	writePos     int64
	readFileNum  int64
	writeFileNum int64
	depth        int64

	sync.RWMutex

	// instantiation time metadata
	name            string
	dataPath        string
	maxBytesPerFile int64 // currently this cannot change once created
	minMsgSize      int32
	maxMsgSize      int32
	syncEvery       int64         // number of writes per fsync
	syncTimeout     time.Duration // duration of time per fsync
	exitFlag        int32
	needSync        bool

	// keeps track of the position where we have read
	// (but not yet sent over readChan)
	nextReadPos     int64
	nextReadFileNum int64

	readFile  *os.File
	writeFile *os.File
	reader    *bufio.Reader
	writeBuf  bytes.Buffer

	// exposed via ReadChan()
	readChan chan []byte

	// internal channels
	writeChan         chan []byte
	writeResponseChan chan error
	emptyChan         chan int
	emptyResponseChan chan error
	exitChan          chan int
	exitSyncChan      chan int

	logf AppLogFunc
}
```
可见其内部结构很简单，包括一些读写文件的指针，当前读（写）到的文件位置，每个文件允许的最大大小等。根据这些信息我们可以推测：
1. 数据在diskQueue中是以磁盘文件形式存储的，并且这些文件的内容是连续的，可以通过不同的fileNum索引到。
2. 这些文件的大小相近。

## diskQueue的创建
首先来看看diskQueue的创建流程，不出意外的话，会有一些load操作，比如读取当前正在读出（写入）数据的文件号，读（写）位置等。
```go
func New(name string, dataPath string, maxBytesPerFile int64,
	minMsgSize int32, maxMsgSize int32,
	syncEvery int64, syncTimeout time.Duration, logf AppLogFunc) Interface {
	d := diskQueue{
		name:              name,
		...
	}

	// no need to lock here, nothing else could possibly be touching this instance
	err := d.retrieveMetaData()
	if err != nil && !os.IsNotExist(err) {
		d.logf(ERROR, "DISKQUEUE(%s) failed to retrieveMetaData - %s", d.name, err)
	}

	go d.ioLoop()
	return &d
}
```
不出所料，果然会调用retrieveMetaData来load数据，我们看看“meta data”都包含什么。
```go
func (d *diskQueue) retrieveMetaData() error {
	var f *os.File
	var err error

	fileName := d.metaDataFileName()
	f, err = os.OpenFile(fileName, os.O_RDONLY, 0600)
	if err != nil {
		return err
	}
	defer f.Close()

	var depth int64
	_, err = fmt.Fscanf(f, "%d\n%d,%d\n%d,%d\n",
		&depth,
		&d.readFileNum, &d.readPos,
		&d.writeFileNum, &d.writePos)
	if err != nil {
		return err
	}
	atomic.StoreInt64(&d.depth, depth)
	d.nextReadFileNum = d.readFileNum
	d.nextReadPos = d.readPos

	return nil
}
```
答案很明显，包含当前diskQueue的消息数（depth），读（写）文件序号、读（写）位置。

自然有load的地方，肯定有对应save的地方，即persistMetaData，其调用在sync中。至于sync干了什么，稍后分析。

回到创建diskQueue的地方，除了retrieveMetaData之外，New还启动了一个ioLoop的goroutine。
## ioLoop

```go
func (d *diskQueue) ioLoop() {
	var dataRead []byte
	var err error
	var count int64
	var r chan []byte

	syncTicker := time.NewTicker(d.syncTimeout)

	for {
		// dont sync all the time :)
		if count == d.syncEvery {
			d.needSync = true
		}

		if d.needSync {
			err = d.sync()
			if err != nil {
				d.logf(ERROR, "DISKQUEUE(%s) failed to sync - %s", d.name, err)
			}
			count = 0
		}

		if (d.readFileNum < d.writeFileNum) || (d.readPos < d.writePos) {
			if d.nextReadPos == d.readPos {
				dataRead, err = d.readOne()
				if err != nil {
					d.logf(ERROR, "DISKQUEUE(%s) reading at %d of %s - %s",
						d.name, d.readPos, d.fileName(d.readFileNum), err)
					d.handleReadError()
					continue
				}
			}
			r = d.readChan
		} else {
			r = nil
		}

		select {
		// the Go channel spec dictates that nil channel operations (read or write)
		// in a select are skipped, we set r to d.readChan only when there is data to read
		case r <- dataRead:
			count++
			// moveForward sets needSync flag if a file is removed
			d.moveForward()
		case <-d.emptyChan:
			d.emptyResponseChan <- d.deleteAllFiles()
			count = 0
		case dataWrite := <-d.writeChan:
			count++
			d.writeResponseChan <- d.writeOne(dataWrite)
		case <-syncTicker.C:
			if count == 0 {
				// avoid sync when there's no activity
				continue
			}
			d.needSync = true
		case <-d.exitChan:
			goto exit
		}
	}

exit:
	d.logf(INFO, "DISKQUEUE(%s): closing ... ioLoop", d.name)
	syncTicker.Stop()
	d.exitSyncChan <- 1
}
```
ioLoop的逻辑稍微复杂一些，包括几部分
1. 如果需要同步，调用sync。
2. 如果有内容可读，则读取数据。
3. 监听若干chan，根据不同消息做出不同响应。

### 同步
```go
func (d *diskQueue) sync() error {
	if d.writeFile != nil {
		err := d.writeFile.Sync()
		if err != nil {
			d.writeFile.Close()
			d.writeFile = nil
			return err
		}
	}

	err := d.persistMetaData()
	if err != nil {
		return err
	}

	d.needSync = false
	return nil
}
```
这里可以看到sync函数做的事情很简单：
1. 如果writeFile存在，刷新其内容到磁盘。
2. 保存元数据。

我们比较关心的是sync合适会被调用，跟踪needSync变量可知，有3种情况会触发同步。
1. 定时器触发，diskQueue会定期触发sync操作以保证数据正确性。
2. 读写一定数目（syncEvery）之后触发。
3. 一些特殊的读写事件，包括读取文件变化，读写失败等。

### 读数据
```go
if (d.readFileNum < d.writeFileNum) || (d.readPos < d.writePos) {
	...
	r = d.readChan
} else {
	r = nil
}
```
这里有个很有意思的设置，如果当前可读，则将r设置为readChan，否则设置为nil。

在select中，空的chan会被跳过

所以如果读到数据，会写入readChan；否则会直接跳过。
来看看真正的读数据操作。
```go
func (d *diskQueue) readOne() ([]byte, error) {
	var err error
	var msgSize int32

	if d.readFile == nil {
		curFileName := d.fileName(d.readFileNum)
		d.readFile, err = os.OpenFile(curFileName, os.O_RDONLY, 0600)
		if err != nil {
			return nil, err
		}

		d.logf(INFO, "DISKQUEUE(%s): readOne() opened %s", d.name, curFileName)

		if d.readPos > 0 {
			_, err = d.readFile.Seek(d.readPos, 0)
			if err != nil {
				d.readFile.Close()
				d.readFile = nil
				return nil, err
			}
		}

		d.reader = bufio.NewReader(d.readFile)
	}

	err = binary.Read(d.reader, binary.BigEndian, &msgSize)
	if err != nil {
		d.readFile.Close()
		d.readFile = nil
		return nil, err
	}

	if msgSize < d.minMsgSize || msgSize > d.maxMsgSize {
		// this file is corrupt and we have no reasonable guarantee on
		// where a new message should begin
		d.readFile.Close()
		d.readFile = nil
		return nil, fmt.Errorf("invalid message read size (%d)", msgSize)
	}

	readBuf := make([]byte, msgSize)
	_, err = io.ReadFull(d.reader, readBuf)
	if err != nil {
		d.readFile.Close()
		d.readFile = nil
		return nil, err
	}

	totalBytes := int64(4 + msgSize)

	// we only advance next* because we have not yet sent this to consumers
	// (where readFileNum, readPos will actually be advanced)
	d.nextReadPos = d.readPos + totalBytes
	d.nextReadFileNum = d.readFileNum

	// TODO: each data file should embed the maxBytesPerFile
	// as the first 8 bytes (at creation time) ensuring that
	// the value can change without affecting runtime
	if d.nextReadPos > d.maxBytesPerFile {
		if d.readFile != nil {
			d.readFile.Close()
			d.readFile = nil
		}

		d.nextReadFileNum++
		d.nextReadPos = 0
	}

	return readBuf, nil
}
```
其实很简单，有以下几个步骤：
1. 如果当前没有读文件，则根据读序号打开文件。
2. 首先读出消息长度msgSize，然后根据msgSize。
3. 按需移动next\*指针，如果当前文件已读完，则移动到下一个文件。

这里可以看到其协议非常简单，就是把每个消息的长度写在改消息前面。因为都是本地读取，所以没有像IP/TCP那样设置各种各样的复杂字段。

readOne返回数据后，diskQueue会尝试把他们写入readChan中。如果写入成功，则更新真正的读写文件序号和位置，如下所示：
```go
func (d *diskQueue) moveForward() {
	oldReadFileNum := d.readFileNum
	d.readFileNum = d.nextReadFileNum
	d.readPos = d.nextReadPos
	depth := atomic.AddInt64(&d.depth, -1)

	// see if we need to clean up the old file
	if oldReadFileNum != d.nextReadFileNum {
		// sync every time we start reading from a new file
		d.needSync = true

		fn := d.fileName(oldReadFileNum)
		err := os.Remove(fn)
		if err != nil {
			d.logf(ERROR, "DISKQUEUE(%s) failed to Remove(%s) - %s", d.name, fn, err)
		}
	}

	d.checkTailCorruption(depth)
}
```
### 写数据
上层可以调用Put来写入消息
```go
func (d *diskQueue) Put(data []byte) error {
	d.RLock()
	defer d.RUnlock()

	if d.exitFlag == 1 {
		return errors.New("exiting")
	}

	d.writeChan <- data
	return <-d.writeResponseChan
}
```
比较有意思的是用了两个chan来完成写操作，任意一个chan都会阻塞Put函数的返回。在ioLoop中当writeChan监听到数据写入时，会调用writeOne写入数据，然后将结果写回writeResponseChan。巧妙地用同步操作来实现了异步写入。

真正写入操作writeOne，因为其过程跟readOne非常类似，此处不再赘述。其中需要注意的地方是使用了writeBuf来合并数据长度和数据本身，一次性写入文件。避免只写入部分数据的问题。

### 错误处理
最后来看一看当读/写发生错误时diskQueue的处理。
首先，写入的错误处理非常简单，直接把错误抛给调用方即可。
而读的错误处理就要复杂许多，但其核心思想就是：跳过当前文件，读写下一个文件。无论是handleReadError还是checkTailCorruption都是这么做的。

## diskQueue读写时机
前面介绍了diskQueue的各个操作的步骤，但是还没有涉及到上层调用。我们需要知道NSQ何时会读/写diskQueue的数据。
事实上从topic.go里面就能很轻易地看到答案。
```go
func (t *Topic) put(m *Message) error {
	select {
	case t.memoryMsgChan <- m:
	default:
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, t.backend)
		bufferPoolPut(b)
		t.ctx.nsqd.SetHealth(err)
		if err != nil {
			t.ctx.nsqd.logf(LOG_ERROR,
				"TOPIC(%s) ERROR: failed to write message to backend - %s",
				t.name, err)
			return err
		}
	}
	return nil
}
```
即在memoryMsgChan满了的时候将数据写入diskQueue。
```go
func (t *Topic) messagePump() {
	...
	if len(chans) > 0 {
		memoryMsgChan = t.memoryMsgChan
		backendChan = t.backend.ReadChan()
	}
	for {
		select {
		case msg = <-memoryMsgChan:
		case buf = <-backendChan:
			msg, err = decodeMessage(buf)
			if err != nil {
				t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
				continue
			}
	...
		}
	}
}
```
而读出则是在messagePump里面的无限for循环里，即只要有数据随时都可能被读出。

对于channel也类似，不再赘述。因为NSQ不保证消息有序，所以这种读写策略是完全可行的。

## 小结
1. NSQ topic/channel会在内存数据channel memoryMsgChan满了之后将数据写入diskQueue。只要diskQueue不为空，就会随时从其中读出数据。
2. diskQueue数据格式为消息长度+消息内容。当数据文件有一个地方出错，后面所有的消息都会丢失，这里可以设计更完善的策略尽量恢复更多消息。
3. golang的chan是个很强大的feature，需要学习理解，融会贯通。
