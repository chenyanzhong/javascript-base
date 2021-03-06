## Eventloop

### <h2 id='1'>1.事件机制</h2>

#### 1.1 模型

#### 1.2 执行栈与事件队列
执行栈:<br/>
javascript是单线程,无法多线程执行事件,所以点击事件、定时器只能一一执行,</br>
执行栈就是当前执行事件的地方,比如一个方法、定时器的方法等<br/>
<br/>
事件队列:<br/>
那异步、定时等事件怎么办呢?</br>
他们会先挂起,等待异步执行请求数据完毕,然后插入存储在事件队列中,</br>
定时执行时间会到判断执行时间,插入事件队列中,但不能保证一定准时执行,会有几毫米的误差
再由执行栈执行完毕去事件队列的事件继续执行

#### 1.3 宏任务与微任务

以下事件属于宏任务：

```
setInterval()
setTimeout()
```

以下事件属于微任务:

```
new Promise()
new MutaionObserver()
```

执行栈任务完毕,会检查是否有微任务,有则执行,去事件队列读取宏任务,如定时任务</br>
微任务永远在宏任务之前执行完毕</br>

### <h2 id='2'>node事件循环</h2>

#### 2.1 node事件机制模型
```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘                     ┌───────────────┐
│  ┌──────────┴────────────┐     //外部事件输入    │   incoming:   │
│  │         poll          │<────connections─────        │
│  └──────────┬────────────┘                     │   data, etc.  │
│  ┌──────────┴────────────┐                     └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
   
```
   
#### 2.1 执行顺序

外部输入数据-->轮询阶段(poll)-->检查阶段(check)--></br>
关闭事件回调阶段(close callback)-->定时器检测阶段(timer)--></br>
I/O事件回调阶段(I/O callbacks)-->闲置阶段(idle, prepare)-->轮询阶段</br>

timer:</br>
这个是定时器阶段，处理setTimeout()和setInterval()的回调函数。
进入这个阶段后，主线程会检查一下当前时间，
是否满足定时器的条件。如果满足就执行回调函数，否则就离开这个阶段

I/O callbacks:</br>
除了以下操作的回调函数，其他的回调函数都在这个阶段执行
setTimeout()和setInterval()的回调函数
setImmediate()的回调函数
用于关闭请求的回调函数，比如socket.on('close', ...)

idle, prepare:</br>
该阶段只供 libuv 内部调用，这里可以忽略。

Poll:</br>
这个阶段是轮询时间，用于等待还未返回的 I/O 事件，比如服务器的回应、用户移动鼠标等等。
这个阶段的时间会比较长。如果没有其他异步任务要处理（比如到期的定时器），会一直停留在这个阶段，等待 I/O 请求返回结果。

check:</br>
该阶段执行setImmediate()的回调函数

close callbacks:</br>
该阶段执行关闭请求的回调函数，比如socket.on('close', ...)

#### 2.2 新增方法

```
process.nextTick();
setImmediate();
```
process.nextTick与promise:</br>
会在本轮执行栈任务执行完毕后执行,比微任务promise()还提前执行</br>
本轮执行栈任务 > process.nextTick() > promise()</br>

setImmediate与setTimeout:</br>
setImmediate 在 check 中执行,与 setTimeout 在 timer 执行,如果在 poll 等待 io 执行,会先进行 check 再执行 timer,
如果符合执行吗, setImmediate 会在 setTimeout 之前执行

   
</br></br></br></br>
相关参考资料来自:</br>
https://zhuanlan.zhihu.com/p/33058983</br>
http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html</br>
