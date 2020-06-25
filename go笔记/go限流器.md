# Golang限流器

## 使用

限流器是微服务中必不缺少的一环，可以起到保护下游服务，防止服务过载等作用。限流器的实现方法有很多种，例如滑动窗口法、Token Bucket、Leaky Bucket等。

其实golang标准库中就自带了限流算法的实现，即`golang.org/x/time/rate`。该限流器是基于Token Bucket(令牌桶)实现的。

简单来说，令牌桶就是想象有一个固定大小的桶，系统会以恒定速率向桶中放Token，桶满则暂时不放。而用户则从桶中取Token，如果有剩余Token就可以一直取。如果没有剩余Token，则需要等到系统中被放置了Token才行。

本文则主要集中介绍下该组件的具体使用方法：

### 构造一个限流器

```go
limiter := NewLimiter(10, 1);
```

这里有两个参数：

- 第一个参数是`r Limit`。代表每秒可以向Token桶中产生多少token。Limit实际上是float64的别名。
- 第二个参数是`b int`。b代表Token桶的容量大小。

其含义为令牌桶大小为1, 以每秒10个Token的速率向桶中放置Token。

除了直接指定每秒产生的Token个数外，还可以用Every方法来指定向Token桶中放置Token的间隔，例如：

```go
limit := Every(100 * time.Millisecond);
limiter := NewLimiter(limit, 1);
```

每100ms往桶中放一个Token。本质上也就是一秒钟产生10个。

Limiter提供了三类方法供用户消费Token，用户可以每次消费一个Token，也可以一次性消费多个Token。而每种方法代表了当Token不足时，各自不同的对应手段。

### Wait/WaitN

```go
func (lim *Limiter) Wait(ctx context.Context) (err error)
func (lim *Limiter) WaitN(ctx context.Context, n int) (err error)
```

当使用Wait方法消费Token时，如果此时桶内Token数组不足(小于N)，那么Wait方法将会阻塞一段时间，直至Token满足条件。如果充足则直接返回。

这里可以看到，Wait方法有一个context参数。我们可以设置context的Deadline或者Timeout，来决定此次Wait的最长时间。

### Allow/AllowN

```go
func (lim *Limiter) Allow() bool
func (lim *Limiter) AllowN(now time.Time, n int) bool
```

AllowN方法表示，截止到某一时刻，目前桶中数目是否至少为n个，满足则返回true，同时从桶中消费n个token。反之返回不消费Token，false。

通常对应这样的线上场景，如果请求速率过快，就直接丢到某些请求。

### Reserve/ReserveN

```go
func (lim *Limiter) Reserve() *Reservation
func (lim *Limiter) ReserveN(now time.Time, n int) *Reservation
```

ReserveN的用法就相对来说复杂一些，当调用完成后，无论Token是否充足，都会返回一个Reservation*对象。

你可以调用该对象的Delay()方法，该方法返回了需要等待的时间。如果等待时间为0，则说明不用等待。必须等到等待时间之后，才能进行接下来的工作。

或者，如果不想等待，可以调用Cancel()方法，该方法会将Token归还。

举一个简单的例子，我们可以这么使用Reserve方法。

```go
r := lim.Reserve()
if !r.OK() {  
  // Not allowed to act! Did you remember to set lim.burst to be > 0 ?    
  return
}
time.Sleep(r.Delay())
Act() // 执行相关逻辑
```

### 动态调整速率

Limiter支持可以调整速率和桶大小：

1. SetLimit(Limit) 改变放入Token的速率
2. SetBurst(int) 改变Token桶大小

有了这两个方法，可以根据现有环境和条件，根据我们的需求，动态的改变Token桶大小和速率。

## 实现分析

上面谈到，time/rate是基于Token Bucket(令牌桶)算法实现的限流。其代码也非常简洁，去除注释后，也就200行左右的代码量。

### 背景

简单来说，令牌桶就是想象有一个固定大小的桶，系统会以恒定速率向桶中放Token，桶满则暂时不放。而用户则从桶中取Token，如果有剩余Token就可以一直取。如果没有剩余Token，则需要等到系统中被放置了Token才行。

一般介绍Token Bucket的时候，都会有一张这样的原理图：

![image-20200624220016581](/Users/zhanghaisong/Documents/book/img/image-20200624220016581.png)

从这个图中看起来，似乎令牌桶实现应该是这样的：

有一个Timer和一个BlockingQueue。Timer固定的往BlockingQueue中放token。用户则从BlockingQueue中取数据。

这固然是Token Bucket的一种实现方式，这么做也非常直观，但是效率太低了：我们需要不仅多维护一个Timer和BlockingQueue，而且还耗费了一些不必要的内存。

在Golang的`timer/rate`中的实现, 并没有单独维护一个Timer，而是采用了lazyload的方式，直到每次消费之前才根据时间差更新Token数目，而且也不是用BlockingQueue来存放Token，而是仅仅通过计数的方式。

### Token的生成和消费

上面谈到，Token的消费方式有三种。但其实在内部实现，最终三种消费方式都调用了reserveN函数来生成和消费Token。

我们看下reserveN函数的具体实现，整个过程非常简单：

1. 计算从当前时间到上次取Token的时刻，期间一共新产生了多少Token。因为我们知道Token Bucket每秒产生的Token数，只要把上次取Token的时间记录下来，求出时间差值，就可以很容易的将时间差转化为Token数。`time/rate`中对应的实现函数为`tokensFromDuration`，后面会详细讲下。从时间转化为Token数目，大致公式如下。

    NewTokensNum = Rate * PastSeconds 

   同时，当前Token数目 = 新产生的Token数目 + 之前剩余的Token数目 - 要消费的Token数目。

2. 如果消费后剩余Token数目大于零，说明此时Token桶内仍不为空，此时无需等待。如果Token数目小于零，则需等待一段时间。那么这个时候，我们需要将负值的Token数转化为需要等待的时间。`time/rate`中对应的实现函数为`durationFromTokens`

3. 将需要等待的时间等相关结果返回给调用方。

从上面可以看出，其实整个过程就是利用了**Token数可以和时间相互转化**的原理。而如果Token数为负，则需要等待相关时间即可。

**注意：**如果当消费时，Token桶中的Token数目已经为负值了，依然可以按照上述流程进行消费。随着负值越来越小，等待的时间将会越来越长。从结果来看，这个行为跟用Timer+BlockQueue实现是一样的。

此外，整个过程为了保证线程安全，更新令牌桶相关数据时都用了mutex加锁。

对于Allow函数实现时，只要判断需要等待的时间是否为0即可，如果大于0说明需要等待，则返回False，反之返回True。

对于Wait函数，直接`t := time.NewTimer(delay)`，等待对应的时间即可。

### float精度问题

从上面原理讲述可以看出，在Token和时间的相互转化函数`durationFromTokens`和`tokensFromDuration`中，涉及到float64的乘除运算。一谈到float的乘除，我们就需要小心精度问题了。

而Golang在这里也踩了坑，以下是`tokensFromDuration`最初的实现版本

```go
func (limit Limit) tokensFromDuration(d time.Duration) float64 {
	return d.Seconds() * float64(limit)
}
```

这个操作看起来一点问题都没：每秒生成的Token数乘于秒数。然而，这里的问题在于，`d.Seconds()`已经是小数了。两个小数相乘，会带来精度的损失。

所以就有了这个issue:golang.org/issues/34861。

修改后新的版本如下：

```go
func (limit Limit) tokensFromDuration(d time.Duration) float64 {
	sec := float64(d/time.Second) * float64(limit)
	nsec := float64(d%time.Second) * float64(limit)
	return sec + nsec/1e9
}
```

`time.Duration`是`int64`的别名，代表纳秒。分别求出秒的整数部分和小数部分，进行相乘后再相加，这样可以得到最精确的精度。

### 数值溢出问题

我们讲reserveN函数的具体实现时，第一步就是计算从当前时间到上次取Token的时刻，期间一共新产生了多少Token，同时也可得出当前的Token是多少。

我最开始的理解是，直接可以这么做：

```go
// elapsed表示过去的时间差
elapsed := now.Sub(lim.last)
// delta表示这段时间一共新产生了多少
Tokendelta = tokensFromDuration(now.Sub(lim.last))

tokens := lim.tokens + delta
if(token > lim.burst){
		token = lim.burst
}
```

其中，`lim.tokens`是当前剩余的Token，`lim.last`是上次取token的时刻。`lim.burst`是Token桶的大小。使用tokensFromDuration计算出新生成了多少Token，累加起来后，不能超过桶的容量即可。

这么做看起来也没什么问题，然而并不是这样。

在`time/rate`里面是这么做的，如下代码所示：

```go
maxElapsed := lim.limit.durationFromTokens(float64(lim.burst) - lim.tokens)
elapsed := now.Sub(last)
if elapsed > maxElapsed {
	elapsed = maxElapsed
}

delta := lim.limit.tokensFromDuration(elapsed)

tokens := lim.tokens + delta
if burst := float64(lim.burst); tokens > burst {
	tokens = burst
}
```

与我们最开始的代码不一样的是，它没有直接用`now.Sub(lim.last)`来转化为对应的Token数，而是 先用`lim.limit.durationFromTokens(float64(lim.burst) - lim.tokens)`，计算把桶填满的时间maxElapsed。取elapsed和maxElapsed的最小值。

这么做算出的结果肯定是正确的，但是这么做相比于我们的做法，好处在哪里？

对于我们的代码，当last非常小的时候（或者当其为初始值0的时候），此时`now.Sub(lim.last)`的值就会非常大，如果`lim.limit`即每秒生成的Token数目也非常大时，直接将二者进行乘法运算，**结果有可能会溢出。**

因此，`time/rate`先计算了把桶填满的时间，将其作为时间差值的上限，这样就规避了溢出的问题。

### Token的归还

而对于Reserve函数，返回的结果中，我们可以通过`Reservation.Delay()`函数，得到需要等待时间。同时调用方可以根据返回条件和现有情况，可以调用`Reservation.Cancel()`函数，取消此次消费。当调用`Cancel()`函数时，消费的Token数将会尽可能归还给Token桶。

此外，我们在上一篇文章中讲到，Wait函数可以通过Context进行取消或者超时等， 当通过Context进行取消或超时时，此时消费的Token数也会归还给Token桶。

然而，归还Token的时候，并不是简单的将Token数直接累加到现有Token桶的数目上，这里还有一些注意点：

```go
restoreTokens := float64(r.tokens) - r.limit.tokensFromDuration(r.lim.lastEvent.Sub(r.timeToAct))if restoreTokens <= 0 {	return}
```

以上代码就是计算需要归还多少的Token。其中：

1. `r.tokens`指的是本次消费的Token数
2. `r.timeToAct`指的是Token桶可以满足本次消费数目的时刻，也就是消费的时刻+等待的时长。
3. `r.lim.lastEvent`指的是最近一次消费的timeToAct值

其中：`r.limit.tokensFromDuration(r.lim.lastEvent.Sub(r.timeToAct))` 指的是，从该次消费到当前时间，一共又新消费了多少Token数目。

根据代码来看，要归还的Token要是该次消费的Token减去新消费的Token。不过这里我还没有想明白，为什么归还的时候，要减去新消费数目。

按照我的理解，直接归还全部Token数目，这样对于下一次消费是无感知影响的。这块的具体原因还需要进一步探索。

## uber-go漏桶限流器

uber在Github上开源了一套用于服务限流的go语言库ratelimit, 该组件基于Leaky Bucket(漏桶)实现。

相比于TokenBucket，只要桶内还有剩余令牌，调用方就可以一直消费。而Leaky Bucket相对来说比较严格，调用方只能严格按照这个间隔顺序进行消费调用。

### ratelimit的使用

直接看下uber-go官方库给的例子：

```go
rl := ratelimit.New(100) // per second

prev := time.Now()
for i := 0; i < 10; i++ {
  now := rl.Take()
  fmt.Println(i, now.Sub(prev))
  prev = now
}
```

在这个例子中，我们给定限流器每秒可以通过100个请求，也就是平均每个请求间隔10ms。

### 基本实现

要实现以上每秒固定速率的目的，其实还是比较简单的。

在ratelimit的New函数中，传入的参数是每秒允许请求量(RPS)。我们可以很轻易的换算出每个请求之间的间隔：

```go
limiter.perRequest = time.Second / time.Duration(rate)
```

以上`limiter.perRequest`指的就是每个请求之间的间隔时间。

如下图，当请求1处理结束后, 我们记录下请求1的处理完成的时刻, 记为`limiter.last`。稍后请求2到来, 如果此刻的时间与`limiter.last`相比并没有达到`perRequest`的间隔大小，那么sleep一段时间即可。

![image-20200624221940098](/Users/zhanghaisong/Documents/book/img/image-20200624221940098.png)

对应ratelimit的实现代码如下：

```go
sleepFor = t.perRequest - now.Sub(t.last)
if sleepFor > 0 {
	t.clock.Sleep(sleepFor)
	t.last = now.Add(sleepFor)
} else {
	t.last = now
}
```

### 最大松弛量

我们讲到，传统的Leaky Bucket，每个请求的间隔是固定的，然而，在实际上的互联网应用中，流量经常是突发性的。对于这种情况，uber-go对Leaky Bucket做了一些改良，引入了最大松弛量(maxSlack)的概念。

我们先理解下整体背景: 假如我们要求每秒限定100个请求，平均每个请求间隔10ms。但是实际情况下，有些请求间隔比较长，有些请求间隔比较短。

请求1完成后，15ms后，请求2才到来，可以对请求2立即处理。请求2完成后，5ms后，请求3到来，这个时候距离上次请求还不足10ms，因此还需要等待5ms。

但是，对于这种情况，实际上三个请求一共消耗了25ms才完成，并不是预期的20ms。在uber-go实现的ratelimit中，可以把之前间隔比较长的请求的时间，匀给后面的使用，保证每秒请求数(RPS)即可。

对于以上case，因为请求2相当于多等了5ms，我们可以把这5ms移给请求3使用。加上请求3本身就是5ms之后过来的，一共刚好10ms，所以请求3无需等待，直接可以处理。此时三个请求也恰好一共是20ms。

在ratelimit的对应实现中很简单，是把每个请求多余出来的等待时间累加起来，以给后面的抵消使用。

```go
t.sleepFor += t.perRequest - now.Sub(t.last)
if t.sleepFor > 0 {
  t.clock.Sleep(t.sleepFor)
  t.last = now.Add(t.sleepFor)
  t.sleepFor = 0
} else {
  t.last = now
}
```

注意：这里跟上述代码不同的是，这里是`+=`。而同时`t.perRequest - now.Sub(t.last)`是可能为负值的，负值代表请求间隔时间比预期的长。

当`t.sleepFor > 0`，代表此前的请求多余出来的时间，无法完全抵消此次的所需量，因此需要sleep相应时间, 同时将`t.sleepFor`置为0。

当`t.sleepFor < 0`，说明此次请求间隔大于预期间隔，将多出来的时间累加到`t.sleepFor`即可。

但是，对于某种情况，请求1完成后，请求2过了很久到达(好几个小时都有可能)，那么此时对于请求2的请求间隔`now.Sub(t.last)`，会非常大。以至于即使后面大量请求瞬时到达，也无法抵消完这个时间。那这样就失去了限流的意义。

为了防止这种情况，ratelimit就引入了最大松弛量(maxSlack)的概念, 该值为负值，表示允许抵消的最长时间，防止以上情况的出现。

```go
if t.sleepFor < t.maxSlack {
  t.sleepFor = t.maxSlack
}
```

ratelimit中maxSlack的值为`-10 * time.Second / time.Duration(rate)`, 是十个请求的间隔大小。我们也可以理解为ratelimit允许的最大瞬时请求为10。

### 高级用法

ratelimit的New函数，除了可以配置每秒请求数(QPS)， 其实还提供了一套可选配置项Option。

```go
func New(rate int, opts ...Option) Limiter
```

Option的类型为`type Option func(l *limiter)`, 也就是说我们可以提供一些这样类型的函数，作为Option，传给ratelimit, 定制相关需求。

但实际上，自定义Option的用处比较小，因为`limiter`结构体本身就是个私有类型，我们并不能拿它做任何事情。

我们只需要了解ratelimit目前提供的两个配置项即可：

#### **`WithoutSlack`**

我们上文讲到ratelimit中引入了最大松弛量的概念，而且默认的最大松弛量为10个请求的间隔时间。

但是确实会有这样需求场景，需要严格的限制请求的固定间隔。那么我们就可以利用WithoutSlack来取消松弛量的影响。

```go
limiter := ratelimit.New(100, ratelimit.WithoutSlack)
```

#### **`WithClock(clock Clock)`**

我们上文讲到，ratelimit的实现时，会计算当前时间与上次请求时间的差值，并sleep相应时间。在ratelimit基于go标准库的time实现时间相关计算。如果有精度更高或者特殊需求的计时场景，可以用WithClock来替换默认时钟。

通过该方法，只要实现了Clock的interface，就可以自定义时钟了。

```go
type Clock interface {
	Now() time.Time
	Sleep(time.Duration)
}

clock &= MyClock{}
limiter := ratelimit.New(100, ratelimit.WithClock(clock))
```



## 总结

Token Bucket其实非常适合互联网突发式请求的场景，其请求Token时并不是严格的限制为固定的速率，而是中间有一个桶作为缓冲。只要桶中还有Token，请求就还可以一直进行。当突发量激增到一定程度，则才会按照预定速率进行消费。

此外在维基百科中，也提到了分层Token Bucket(HTB)作为传统Token Bucket的进一步优化，Linux内核中也用它进行流量控制。