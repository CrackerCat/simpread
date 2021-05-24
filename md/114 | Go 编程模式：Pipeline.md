> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [time.geekbang.org](https://time.geekbang.org/column/article/332610)

> 你好，我是陈皓，网名左耳朵耗子。

你好，我是陈皓，网名左耳朵耗子。

这节课，我着重介绍一下 Go 编程中的 Pipeline 模式。对于 Pipeline，用过 Unix/Linux 命令行的人都不会陌生，它是一种把各种命令拼接起来完成一个更强功能的技术方法。

现在的流式处理、函数式编程、应用网关对微服务进行简单的 API 编排，其实都是受 Pipeline 这种技术方式的影响。Pipeline 可以很容易地把代码按单一职责的原则拆分成多个高内聚低耦合的小模块，然后轻松地把它们拼装起来，去完成比较复杂的功能。

HTTP 处理
-------

这种 Pipeline 的模式，我在上节课中有过一个示例，我们再复习一下。

上节课，我们有很多 WithServerHead() 、WithBasicAuth() 、WithDebugLog() 这样的小功能代码，在需要实现某个 HTTP API 的时候，我们就可以很轻松地把它们组织起来。

原来的代码是下面这个样子：

http.HandleFunc("/v1/hello", WithServerHeader(WithAuthCookie(hello)))

http.HandleFunc("/v2/hello", WithServerHeader(WithBasicAuth(hello)))

http.HandleFunc("/v3/hello", WithServerHeader(WithBasicAuth(WithDebugLog(hello))))

通过一个代理函数：

type HttpHandlerDecorator func(http.HandlerFunc) http.HandlerFunc

func Handler(h http.HandlerFunc, decors ...HttpHandlerDecorator) http.HandlerFunc {

for i := range decors {

d := decors[len(decors)-1-i]

h = d(h)

}

return h

}

我们就可以移除不断的嵌套，像下面这样使用了：

http.HandleFunc("/v4/hello", Handler(hello,

WithServerHeader, WithBasicAuth, WithDebugLog))

Channel 管理
----------

### Channel 转发函数

首先，我们需要一个 echo() 函数，它会把一个整型数组放到一个 Channel 中，并返回这个 Channel。

func echo(nums []int) <-chan int {

out := make(chan int)

go func() {

for _, n := range nums {

out <- n

}

close(out)

}()

return out

}

然后，我们依照这个模式，就可以写下下面的函数。

### 平方函数

func sq(in <-chan int) <-chan int {

out := make(chan int)

go func() {

for n := range in {

out <- n * n

}

close(out)

}()

return out

}

### 过滤奇数函数

func odd(in <-chan int) <-chan int {

out := make(chan int)

go func() {

for n := range in {

if n%2 != 0 {

out <- n

}

}

close(out)

}()

return out

}

### 求和函数

func sum(in <-chan int) <-chan int {

out := make(chan int)

go func() {

var sum = 0

for n := range in {

sum += n

}

out <- sum

close(out)

}()

return out

}

用户端的代码如下所示（注：你可能会觉得，sum()，odd() 和 sq() 太过于相似，其实，你可以通过 Map/Reduce 编程模式或者是 Go Generation 的方式合并一下）：

var nums = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

for n := range sum(sq(odd(echo(nums)))) {

fmt.Println(n)

}

上面的代码类似于我们执行了 Unix/Linux 命令： echo $nums | sq | sum。同样，如果你不想有那么多的函数嵌套，就可以使用一个代理函数来完成。

type EchoFunc func ([]int) (<- chan int)

type PipeFunc func (<- chan int) (<- chan int)

func pipeline(nums []int, echo EchoFunc, pipeFns ... PipeFunc) <- chan int {

ch := echo(nums)

for i := range pipeFns {

ch = pipeFns[i](ch)

}

return ch

}

然后，就可以这样做了：

var nums = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

for n := range pipeline(nums, gen, odd, sq, sum) {

fmt.Println(n)

}

Fan in/Out
----------

动用 Go 语言的 Go Routine 和 Channel 还有一个好处，就是可以写出 1 对多，或多对 1 的 Pipeline，也就是 Fan In/ Fan Out。下面，我们来看一个 Fan in 的示例。

假设我们要通过并发的方式对一个很长的数组中的质数进行求和运算，我们想先把数组分段求和，然后再把它们集中起来。

下面是我们的主函数：

func makeRange(min, max int) []int {

a := make([]int, max-min+1)

for i := range a {

a[i] = min + i

}

return a

}

func main() {

nums := makeRange(1, 10000)

in := echo(nums)

const nProcess = 5

var chans [nProcess]<-chan int

for i := range chans {

chans[i] = sum(prime(in))

}

for n := range sum(merge(chans[:])) {

fmt.Println(n)

}

}

再看我们的 prime() 函数的实现 ：

func is_prime(value int) bool {

for i := 2; i <= int(math.Floor(float64(value) / 2)); i++ {

if value%i == 0 {

return false

}

}

return value > 1

}

func prime(in <-chan int) <-chan int {

out := make(chan int)

go func () {

for n := range in {

if is_prime(n) {

out <- n

}

}

close(out)

}()

return out

}

我来简单解释下这段代码。

首先，我们制造了从 1 到 10000 的数组；

然后，把这堆数组全部 echo 到一个 Channel 里—— in；

此时，生成 5 个 Channel，接着都调用 sum(prime(in)) ，于是，每个 Sum 的 Go Routine 都会开始计算和；

最后，再把所有的结果再求和拼起来，得到最终的结果。

其中的 merge 代码如下：

func merge(cs []<-chan int) <-chan int {

var wg sync.WaitGroup

out := make(chan int)

wg.Add(len(cs))

for _, c := range cs {

go func(c <-chan int) {

for n := range c {

out <- n

}

wg.Done()

}(c)

}

go func() {

wg.Wait()

close(out)

}()

return out

}

整个程序的结构如下图所示：

![](https://static001.geekbang.org/resource/image/f9/b3/f9d2b599620d5bc191194ff239f0a1b3.jpg)

参考文档
----

如果你还想了解更多类似的与并发相关的技术，我再给你推荐一些资源：

好了，这节课就到这里。如果你觉得今天的内容对你有所帮助，欢迎你帮我分享给更多人。