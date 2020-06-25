## go timer

本文介绍 Timer，Tick， Sleep 的实现机制。版本是 GO 1.9 。

##  Ticker

每隔 duration 时间会把当前的时间点放入到 channel 中，应用可以从 channel 进行读取。应用需要周期性的时间间隔，可以使用此方法。

使用 Ticker 有两种方式，NewTicker 可以获取 Ticker 实例，Stop 可以显示的停止 Tick 运行。Stop 可以释放 timer 资源，但不会关闭 channel，防止应用层报错。

如果 Ticker 一直随应用运行，不会关闭，可以使用 time.Tick 直接获取 time channel。这个没有 Ticker 实例，无法显示关闭。 timer 会一直运行。

## Timer

定时器，和 Tick 类似，经过 duration 时间，Timer 会触发，并且往 channel 写入当前时间点，此时 Timer 不再计时。当应用层重新调用 Reset 函数，才又开始计时，这个是和 Tick 不同的。Tick 是周期性的计时。

Timer 还支持计时结束时，触发自定义函数。AfterFunc 会返回 Timer 实例。AfterFunc 调用后，只会计时结束后触发一次自定义函数调用，如果需要再次触发，需要显示调用 Timer.Reset 函数。

如果结束定时器，调用 Stop 即可。

## Ticker 运行机制

在 go 源码的 time/tick.go 中，可以看到 NewTicker 实现

> ```
> func NewTicker(d Duration) *Ticker {
>  if d <= 0 {
>  panic(errors.New("non-positive interval for NewTicker"))
>  }
>  // Give the channel a 1-element time buffer.
>  // If the client falls behind while reading, we drop ticks
>  // on the floor until the client catches up.
>  c := make(chan Time, 1)
>  t := &Ticker{
>  C: c,
>  r: runtimeTimer{
>  when: when(d),
>  period: int64(d),
>  f: sendTime,
>  arg: c,
>  },
>  }
>  startTimer(&t.r)
>  return t
> }
> ```

可以看到 channel 是带有1个元素的缓冲区。重点关注 runtimeTimer 的定义

\* when 何时触发定时器。所有的时间值都是纳秒级别， int64 表示。
\* period: 触发周期。根据此值计算下一次触发时间点， 可以简单理解成 when = when + period。当前时间触发后，会更新 when 值。
\* f : 定时器触发时，调用的函数。注意这个函数必须是非阻塞的，否则会阻塞整个 timer 的执行。
\* arg : 调用函数 f 时，传入的参数值。

看到 Ticker 对应的触发函数是 sendTime, 函数实现如下(time/sleep.go)：

> func sendTime(c interface{}, seq uintptr) {
> // Non-blocking send of time on c.
> // Used in NewTimer, it cannot block anyway (buffer).
> // Used in NewTicker, dropping sends on the floor is
> // the desired behavior when the reader gets behind,
> // because the sends are periodic.
> select {
> case c.(chan Time) <- Now():
> default:
> }
> }

会把当前的时间点放入到 channel 中，如果应用层没有及时获取 channel 中的值，会直接丢弃当前的时间点，走 default 逻辑。

startTimer 是开启了定时器，此时定时器启动执行。startTimer 相应代码在 runtime/time.go 中。startTimer -> addtimer -> addtimerLocked, 具体实现在 addtimerLocked 中

> 
> func addtimerLocked(t *timer) {
> // when must never be negative; otherwise timerproc will overflow
> // during its delta calculation and never expire other runtime timers.
> if t.when < 0 {
> t.when = 1<<63 - 1
> }
> t.i = len(timers.t)
> timers.t = append(timers.t, t)
> siftupTimer(t.i)
> if t.i == 0 {
> // siftup moved to top: new earliest deadline.
> if timers.sleeping {
> timers.sleeping = false
> notewakeup(&timers.waitnote)
> }
> if timers.rescheduling {
> timers.rescheduling = false
> goready([timers.gp](http://timers.gp/), 0)
> }
> }
> if !timers.created {
> timers.created = true
> go timerproc()
> }
> }

timers 是全局变量，管理所有的 timer, 使用数组维护的。数组中的 timer 是有序的，使用了堆排序算法，把 when 最小值排到数组前面，也就是把最先触发定时的 timer 排在最前。

每次新增定时器 timer 时，会调用 siftupTimer 调整数组顺序。

如果当前 timers 是空数组，需要调整 timers 状态， sleeping =false, rescheduling =false, 如果 timerproc 的 goroutine 为 idle 状态，进行唤醒。调用 goready 函数。

如果 timers 是初次创建，会调用 timerproc ，定时器逻辑全在这里实现。

timerproc 死循环执行, 执行逻辑如下

\1. 获取当前时间点，纳秒级别
\2. 如果 timers 数组为空，把执行 timerproc 的 goroutine 置为 idle 状态，节省资源，不必要的空转
\3. 如果 timers 非空，取出 timers 数组中的第一个值，与 now 比较 timer 的 when 值， 如果 when 值大，说明 timers 中的所有定时器都还未触发，timerproc 的 goroutine sleep， 直到 when 值时刻
\4. 如果数组第一值的 when < now, 说明已经到了触发时间点，如果 timer 的 period 有值，说明是周期性触发，更新 timer 下次触发时间点

> 
> delta = t.when - now
> t.when += t.period * (1 + -delta/t.period)
> siftdownTimer(0)

下次的时间点，不是简单的 t.when += t.period, 而是在 0 ~ period 之间。考虑调度因素，选择 0 ~ period 更合理，保证在 period 内会触发。siftdownTimer 使用堆排序算法调整 timer 顺序，timer 的 when 值增加了，可能需要下沉，排在数组后面。

\5. 如果没有设置 period 值，则移除 timers 数组

\6. 调用 timer.f 函数，当然， timer.arg 是其中的参数

> 
> f := t.f
> arg := t.arg
> seq := t.seq
> f(arg, seq)


Ticker.Stop 函数最终实现对应 runtime/time.go 中的 deltimer 函数。实际就是把 timer 从 timers 数组中删除，删除之后，还需要调整 timers 的数组顺序。

## Timer 的运行机制

Timer 和 Ticker 底层实现走的是同一样一套逻辑。NewTimer 定义在 runtime/sleep.go 中实现

> 
> func NewTimer(d Duration) *Timer {
> c := make(chan Time, 1)
> t := &Timer{
> C: c,
> r: runtimeTimer{
> when: when(d),
> f: sendTime,
> arg: c,
> },
> }
> startTimer(&t.r)
> return t
> }

和 Tick 定义类似，但缺少了 period 的定义。如果定时器触发了， Timer 会被 timers 移除，不在 timers 数组中运行。如果需要运行，需要再次调用 Reset 函数。这里可以看到，触发的函数也是 sendTime, 和 Tick 是一样的。

> 
> func (t *Timer) Reset(d Duration) bool {
> if t.r.f == nil {
> panic("time: Reset called on uninitialized Timer")
> }
> w := when(d)
> active := stopTimer(&t.r)
> t.r.when = w
> startTimer(&t.r)
> return active
> }

Reset 主要是重新计算了 when 值，加入到了 timers 数组中，等待再次触发。

Timer 还支持自定义的函数处理。

> 
> func AfterFunc(d Duration, f func()) *Timer {
> t := &Timer{
> r: runtimeTimer{
> when: when(d),
> f: goFunc,
> arg: f,
> },
> }
> startTimer(&t.r)
> return t
> }
>
> func goFunc(arg interface{}, seq uintptr) {
> go arg.(func())()
> }

如果使用 AfterFunc ， 定时器触发时会调用 goFunc 函数，参数 arg 就是我们在 AfterFunc 中自定义的函数 f。AfterFunc 返回 Timer 实例，这样可以显示调用 Stop 进行关闭定时器，或者调用 Reset 也能再次触发。

## Ticker 和 Timer 的区别

\* 如果是周期性的调用，推荐使用 Ticker， 性能更高，实现更简单
\* 如果只是偶尔的触发定时器，使用 Timer，更节省资源，就算不调用 Stop，触发一次后，也不会一直存在在 timers 数组中
\* Timer 使用更灵活些，支持自定义函数的场景
\* Ticker 需要关注资源泄露的情况，如果 Ticker 不在使用，要显示调用 Stop，否则会一直存在在 timers 数组中

## Sleep 实现机制

sleep 底层实现在 runtime/time.go 函数中，对应函数为 timeSleep。

> 
> func timeSleep(ns int64) {
> if ns <= 0 {
> return
> }
>
> t := getg().timer
> if t == nil {
> t = new(timer)
> getg().timer = t
> }
> *t = timer{}
> t.when = nanotime() + ns
> t.f = goroutineReady
> t.arg = getg()
> lock(&timers.lock)
> addtimerLocked(t)
> goparkunlock(&timers.lock, "sleep", traceEvGoSleep, 2)
> }

也是开启了一个计时器，计算了触发时间值 when， 对应的触发函数 goroutineReady， 触发时会把 goroutine 的等待状态变为运行状态。goparkunlock 会把当前的 goroutine 变为等待状态。

缺少 period 的定义，这样也是触发一次，会在 timers 数组中移除。

## 总结

\* golang timer 在底层实现上，支持纳秒级别
\* 各个 timer 不是单独进行系统调用获取时间，而是 timers 统一调用，性能更高
\* 触发时间点尽力保证在 period 内，如果 period 比较小，在高 CPU 压力下，也很难保证。这种情况下，看到的现象是，有时会隔一段时间(比 period 长)触发，但是下次触发非常快，因为这时，下次的 when 值不会更新
\* 已经存在 timer 的情况下，调低 period 或者 duration 值，性能影响比较小
\* 如果会产生大量 timer 的情况下，性能比较差，数组元素多，每次都要进行堆排序算法调整
\* Timer 的启动或者 Reset 会计算 when 值，这时会获取系统函数，大量使用 Timer 性能会比较差
\* Ticker 要注意资源泄露，不使用的情况需要及时 Stop，否则会一直存在在数组中