---
layout: post
title: Go｜记一次内存问题定位
categories: [golang]
mermaid: true
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# 起因

​ 正常情况下，我很少去关注go开发的程序的内存情况（因为一般情况下编译环节以及runtime
GC已经帮我处理了大部分需要操心的内容，只要我不写那种大对象持续维护基本不怎么需要关注）。

​ 这天同事表示他的服务在线上运行，监控显示内存在持续增长，并没有停下来的意思，“八卦”的我就开始跟他一起进行了定位排查。

# 现象

​ 从监控看，随着应用服务的运行和对外提供服务，副本内存持续上升，且一直到几个G不见停止。

![image-20240717151300974](/images/posts/2024-07-18-golang-memory-check-image-20240717151300974.png)

<center>图一：大致示意，非现场实际图</center>

![image-20240718202449986](/images/posts/2024-07-18-golang-memory-check-image-20240718202449986.png)

<center>图二：真实用率图，当时16G内存大约已经用到2.5G</center>

# 总结

为了避免大家看定位过程繁琐冗长中途放弃，就让结论优先：



> [!IMPORTANT]
>
> - **监控展示的内存用量不一定代表程序实际用量，可能还包括其他组成部分，具体需要看具体的CRI或容器平台采集方式；**
> - **程序在运行过程中记录日志会占用系统cache、active file cache等用量，正常情况下程序rss不增长问题不大；**
> - **Go pprof heap只能采样分析heap用量，对于比如文件资源句柄等陷入系统范畴的内存用量无法体现；**
> - **Go程序在使用系统资源（比如文件）时需比使用自身数据结构更加注意复用性：**
    >
- **自身数据结构用量不复用最多性能弱、内存水位高，基本不触发内存泄漏类问题，且pprof heap比较容易体现和定位；**
>   - **系统资源不复用可能触发内存泄漏类问题，且pprof heap不容易体现和定位。**

# 初次定位阶段

​ 这部分我没有参与，事后了解应该是该服务的开发同事自行在压测环境做了pprof分析，大致定位到某个接口可能存在问题，于是在压测环境将该接口逻辑置空，再进行复现发现上升趋势仍旧存在但相较之前平缓很多。

# 二次定位阶段：监控展示问题

### 观察压测现象

​ 这个时候我介入进行压测（此时我不知道有个接口逻辑被置空）。看到的现象如下：

- pprof分析heap并不明显，top显示占比最大的不过几M级别（图为大致示意，真实监控图当时未留存）。

  ![image-20240718181901858](/images/posts/2024-07-18-golang-memory-check-image-20240718181901858.png)

- 容器内top展示的res与监控展示的内存量存在出入：res一般在几十M量级，而监控展示在几百M量级（图为大致示意，真实监控图当时未留存）。

![image-20240717152324908](/images/posts/2024-07-18-golang-memory-check-image-20240717152324908.png)

> [!IMPORTANT]
>
> 于是我的注意力被监控展示吸引，猜想可能程序没有问题，只是展示有问题
>
> （当然最后证明我是错的，程序和展示都有问题）

### 寻找监控计算方式

​ 基于上面的现象和猜想，问题转换了，也成了这个阶段我寻求真相的焦点：

<center><strong>监控展示的数据采集的是什么（或者说计算方式是什么）？</strong></center>

- 我询问了伏羲（监控提供方）的同事，得知监控展示的内存还包含了active file cache。
- 我也尝试了下CGroup数据计算，发现 `cgroup/memory.usage_in_bytes - /cgroup/memory.stat[中的cache]`与监控展示数据差不多。

![image-20240717154228836](/images/posts/2024-07-18-golang-memory-check-image-20240717154228836.png)

```shell
cat /sys/fs/cgroup/memory/memory.usage_in_bytes
17179373568 # 16383.52734375M

cat /sys/fs/cgroup/memory/memory.stat     
cache 16395632640 # 15636.09375M
...

# 16383.52734375-15636.09375=747.43359375M（多次测算基本都一致，前后略微有一点差异）
```

> [!NOTE]
>
> memory.stat中除了cache、rss外，还包含了mapped file、active file cache等，因此确实基本可以证实伏羲同事所言。

### 猜测

​ 既然知道了监控的采集算法中还包含了active file cache，那么监控的内存会比实际程序使用内存高，且在程序提供服务的一段时间内逐步升高至的原因也就露出水面了——
**日志文件**：

- 程序在提供服务时会记录日志，会导致active file cache等随着日志文件占用的增加而增长。
- 但一般没有太大影响，即便因此内存到达容器限制上限，也不会因此触发OOM，而是会主动释放相关cache。

---

### 验证

- 实验步骤：找一个压测过（历史切割、当前handle的日志文件都有）的副本，将这些日志文件清空，再观察监控展示的内存情况、程序实际rss情况。
- 预期结果：程序rss不变化，监控展示内存收缩到与rss差不多的量级。
- 实验结论：符合预期。

```shell
# 查看当下memory.stat
cat /sys/fs/cgroup/memory/memory.stat
cache 3122651136 # 2977.9921875M
rss 39342080 # 37.51953125M
...
active_file 170176512 # 162.29296875M
...

# 删除日志文件
cd /opt/logs/
rm -rf access-2024-06-2*
...

# 重新查看memory.stat
cat /sys/fs/cgroup/memory/memory.stat
cache 135168 # 0.12890625M
rss 39206912 # 37.390625M
...
active_file 0 # 0M
...
```

![image-20240717182231607](/images/posts/2024-07-18-golang-memory-check-image-20240717182231607.png)

## 三次定位阶段：程序资源复用问题

### 新的问题

​ 至此我已经可以解释为什么监控展示的内存比top的res、memory.stat的rss要高的原因了。

​ 就在我以为我已经找到真相的时候，同事告诉了我一个消息:cry:
：正式环境随着持续服务持续运行，top的res、memory.state的rss也在逐步增加，并没有缓下来的意思，且当时已经几百M要破G的趋势。

​ 这意味着，除了二次定位阶段的监控展示原因外，程序可能确实也存在内存问题，所以还得继续定位。

### 新的方向

​ 但是我继续在压测环境不管怎么压，pprof的heap和程序rss都不能很明显地发现问题。

​ 这时候同事跟我说猜测是不是跟那个接口（就是压测环境他之前置空的接口）有关系，我压测不明显是因为压测的是没有那个接口的情况。

​ 于是我们一起核对那个接口的代码，在其中发现了类似这样一段：

```go
package util

// 业务包装的记录埋点日志
func Push(pushData PushData) error {
	pushConfig := config.C.Push

	// 创建新的埋点pusher
	newPusher, err := NewPusher(pushConfig.Path, pushConfig.Activity, pushConfig.Schema, pushConfig.MaxSize, pushConfig.MaxAge, pushConfig.MaxBackups)
	if err != nil {
		return err
	}

	// 记录埋点日志
	err = newPusher.Push(pushData.Event, pushData.UserId, pushData.Result, pushData.Tag1, pushData.Tag2, pushData.UserStatus, pushData.Value)
	if err != nil {
		return err
	}
	return nil
}
```

### 新的猜测

​ 首先毫无疑问，上述代码必定是可以将pusher优化成单例的。

​ 只是从我的认知来讲，即便像这样每次都创建新对象对内存的影响只是稳定时需要较多内存（比如单例稳定时只需要20M，现在可能需要30M或者50M），而不会直接导致内存的持续上涨，原因如下：

- 如果编译器将其分配在栈上：
    - 随着该函数的结束一般也就立刻回收了；
    - 即便导致了栈扩展，后续GC时处理栈收缩（Go协程栈相关有时间单开一篇分享）；
    -
    再即便栈没收缩，http连接释放后该goroutine销毁也会回收（[http连接与goroutine的关系可以参考我之前分享的这篇](../20240717-从源码认识gin和标准库http包)）。
- 如果编译器逃逸分析至堆上，那么GC也会在后续处理过程中回收（看代码逻辑并没有持续持有相关变量）。

---

​ 那么问题可能出现在NewPusher内部，每次NewPusher时会创建一个新的io.Writer去操作文件，所以猜测还是跟日志文件资源有关。

> [!NOTE]
>
> pprof heap只是分析堆的内存分配，而文件资源已经归属于操作系统资源范畴了，这样也能解释为什么pprof heap分析效果不明显。

### 验证

​ 为了验证这个新的猜想，又开始新的一轮验证

##### 实验条件

- 简化实验逻辑，模拟Http
  Server的多goroutine运行模型（[运行模型也可以参考我之前分享的这篇](../20240717-从源码认识gin和标准库http包)），只执行埋点日志逻辑。
- 简化实验环境，本地Mac环境直接执行，理论上darwin与linux在系统文件方面差异不会过大（毕竟都是类unix）。
- 控制复用变量：
    - Pusher复用&&io.Writer不复用
    - Pusher不复用&io.Writer复用
    - Pusher复用&io.Writer复用

##### 实验1:Pusher不复用&io.Writer不复用

- 预期结果：程序内存（rss）出现不正常的逐步增长；pprof heap分析top量级不如外显的内存。
- 代码示例：每次创建新的Pusher+每次创建新的io.Writer

```go
func Push(pushData PushData) error {
// 每次创建新的Pusher
newPusher, err := NewPusher(pushConfig.Path,
pushConfig.Activity,
pushConfig.Schema,
pushConfig.MaxSize,
pushConfig.MaxAge,
pushConfig.MaxBackups)
...
err = newPusher.Push(pushData.Event,
pushData.UserId,
pushData.Result,
pushData.Tag1,
pushData.Tag2,
pushData.UserStatus,
pushData.Value)
...
}

func NewPusher(filePath, activity, schema string, maxSize, maxAge, maxBackups int) (*pusher.Pusher, error) {
...
// 每次创建新的io.Writer
writer := &lumberjack.Logger{
Filename:   filePath,
MaxSize:    maxSize,
MaxAge:     maxAge,
MaxBackups: maxBackups,
LocalTime:  true,
Compress:   false,
}
return &pusher.Pusher{
FilePath: filePath,
Activity: activity,
Schema:   schema,
Writer:   writer,
}, nil
}
```

- 内存状况：随着Push执行而逐步增加，66M->115M->直到Push停止停留在270+M

![{"anchor_href":"","base_size":"-1,-1","expected_size":"-1,-1","external_info":"","font_size_type_change_aware_":false,"id":"","image_margin":2,"image_url":"","original_name":"","original_path":"","original_size":"-1,-1","press_can_drag":true,"show_in_image_viewer":true}](/images/posts/2024-07-18-golang-memory-check-{37eb2ae3-ad84-49fc-95d4-f543627f538a}.png)

:arrow_down:

![{"anchor_href":"","base_size":"-1,-1","expected_size":"-1,-1","external_info":"","font_size_type_change_aware_":false,"id":"","image_margin":2,"image_url":"","original_name":"","original_path":"","original_size":"-1,-1","press_can_drag":true,"show_in_image_viewer":true}](/images/posts/2024-07-18-golang-memory-check-{46131b79-7e1e-472a-9c87-a470781d7411}.png)

:arrow_down:

![{"anchor_href":"","base_size":"-1,-1","expected_size":"-1,-1","external_info":"","font_size_type_change_aware_":false,"id":"","image_margin":2,"image_url":"","original_name":"","original_path":"","original_size":"-1,-1","press_can_drag":true,"show_in_image_viewer":true}](/images/posts/2024-07-18-golang-memory-check-{adce2255-294e-4e1a-9157-28a332abfc00}.png)

- pprof heap分析：

​ 总体量级在几十M级别，比270+M小

![image-20240718174615288](/images/posts/2024-07-18-golang-memory-check-image-20240718174615288.png)

​ 最大头的是runtime在创建g（由于模拟多goroutine）

![image-20240718174636484](/images/posts/2024-07-18-golang-memory-check-image-20240718174636484.png)

​ 其次的writer也只占用了3.5M

![image-20240718174711642](/images/posts/2024-07-18-golang-memory-check-image-20240718174711642.png)

- 逃逸分析：Logger和Pusher都逃逸到堆（不过从pprof看，也就几M）

![image-20240718181614137](/images/posts/2024-07-18-golang-memory-check-image-20240718181614137.png)

##### 实验2:Pusher不复用&io.Writer复用

- 预期结果：内存很快就稳定在一个较低水位。
- 代码示例：

```go
func Push(pushData PushData) error {
// 每次创建新的Pusher
newPusher, err := NewPusher(pushConfig.Path,
pushConfig.Activity,
pushConfig.Schema,
pushConfig.MaxSize,
pushConfig.MaxAge,
pushConfig.MaxBackups)
...
err = newPusher.Push(pushData.Event,
pushData.UserId,
pushData.Result,
pushData.Tag1,
pushData.Tag2,
pushData.UserStatus,
pushData.Value)
...
}

// 复用io.Writer
var writer = &lumberjack.Logger{
Filename:   "./logs/pusher.log",
MaxSize:    1024,
MaxAge:     10,
MaxBackups: 100,
LocalTime:  true,
Compress:   false,
}

func NewPusher(filePath, activity, schema string, maxSize, maxAge, maxBackups int) (*pusher.Pusher, error) {
...
return &pusher.Pusher{
FilePath: filePath,
Activity: activity,
Schema:   schema,
Writer:   writer,
}, nil
}
```

- 内存状况：一直到Push结束，都保持在10M上下。

![image-20240718175635896](/images/posts/2024-07-18-golang-memory-check-image-20240718175635896.png)

- 逃逸分析：仅Pusher逃逸到堆。

![image-20240718181733880](/images/posts/2024-07-18-golang-memory-check-image-20240718181733880.png)

##### 实验3:Pusher复用&io.Writer复用

​ 偷个懒不再实验了，已经有结论了，反正不会比Pusher不复用&io.Writer复用要差。看了一下逃逸分析，仅Pusher依然会逃逸到堆上，但这种情况下Pusher只有一个，被多goroutine复用，确实应该再堆上。

![image-20240718193936489](/images/posts/2024-07-18-golang-memory-check-image-20240718193936489.png)