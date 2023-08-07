# MIT 6.824 Lab1 mapreduce



# 前言

MIT 6.824是著名的分布式课程，课程包含了视频，讲义，与作业。而本篇博文将阐述6.824课程的第一个作业的一些思考与解法，记录一些关于Map Reduce系统的思考。

# Map Reduce

Map Reduce作为“谷三篇”的第一篇，出名不是没有原因的，jeff dean的超前思想构建了谷歌搜索的基石，使得谷歌在超大应用系统的构建上得心应手。而Map Reduce则是一个基石中的基石。

Map Reduce的思想就是分布式的，系统中包含一个Master和许多个Worker，Master负责调度Worker与任务分发，容灾等等，Worker则与Master通信，请求任务。

而Map Reduce则把一个任务拆分成Map与Reduce部分，简单来说，Map部分是把输入通过用户定义的Map Function输出成中间文件，再把中间文件作为输入给Reduce，Reduce把中间文件调用Reduce Function，然后合并并输出。

![](/images/mapreduce.png)

整个系统如图所示。如论文所说，这些文件可以是在本地机器上，也可以在分布式文件系统中，这并不影响整个系统框架。

具体Map Reduce的思想可以读一下Google的论文。



# MIT 6.824 Lab1

现代的6.824比以前的要难许多，我做过之前的lab1，当时就把Map Reduce的框架都搭好了，只要写两个函数就算通过了。而这个2020的6.824要求对Go熟悉且要把MapReduce整个实现一个大概出来，前后花了不少时间去思考要怎么来做这个实验。

## 思考

当我们拿到Lab的时候需要做什么？需要思考我们要实现哪些东西。MIT的代码里只给了几个RPC的函数，然后在这个基础去实现Map Reduce。

而我们要做的有：

1. 实现Master管理，这其中需要管理任务的状态，Worker请求任务的处理，Worker任务完成报告的处理，Worker失败超时的处理。
2. 实现Worker请求任务，对Map部分任务的处理，对Reduce部分任务的处理，还有任务完成的上报。
3. 还要实现一个文件锁，不能让Goroutine之间产生Race。

## 实现

我的一些方案是在Master的结构体里管理两个Channel，当Master启动之后，把任务发给MapChannel，然后在另一个Goroutine里面对这个MapChannel进行读取，Channel如果不设置的话，一方没有读，写方会阻塞住，所以只有Worker进行请求任务之后才会继续生产任务。当Map部分完成后，再启动Reduce部分，生产Reduce任务到ReduceChannel。

我们在每个请求任务的RPC返回之前再开启一个Goroutine来等待Worker的任务完成报告，这里我们用到了Go的select语法，并使用一个timer，如果超时就把这个任务再次Push进TaskChannel，即使任务失败了，也能再次把任务分发下去。

对于Worker部分的代码来说，完全就是苦力活，可以看看官方代码里那个非分布式的Worker里代码，可以直接复制过来。Worker如果联系不上Master了，马上退出进程，这样就不用实现一个退出语义了。（其实是我不知道怎么让Master去通知Worker退出）

## 代码

这里只给出一些结构体的定义：

**Master:**

```go
type Master struct {
	MapTaskList    []*MapTask
	ReduceTaskList []*ReduceTask

	MapTaskChan    chan *MapTask
	ReduceTaskChan chan *ReduceTask

	CompleteMapTaskNum    int
	CompleteReduceTaskNum int

	mu      sync.Mutex
	files   []string
	nReduce int
}
```

**Task Def**

```go
type MapTask struct {
	TaskNum    int
	TaskStatus TASK_STATUS
	FileName   string
	NReduce    int
}
type ReduceTask struct {
	TaskNum    int
	TaskStatus TASK_STATUS
	FileName   []string
	NReduce    int
}
```

## 一些吐槽

这次写了蛮久了，因为不是很熟悉Go，学了一会才知道有select这种语法，然后官方的hint里让人把中间文件命名为mr-x-y.txt，这样其实挺坑的，因为testmr.sh里面没有把这个中间文件删除掉，而在文件读写的时候因为hash的缘故，不会把所有的文件清空，这样导致前一个test的中间文件会影响到后一个test。所以我加一个函数，把所有的中间文件清空的，这样也算是TDD吧。

接下来就是要做Lab2了，完成一个Raft。
