## Go 并发模式：context

### 介绍

在 Go 服务器中，每个传入请求都在其自己的 goroutine 中处理。 请求处理程序通常会启动额外的 goroutine 来访问后端，例如数据库和 RPC 服务。 处理请求的一组 goroutine 通常需要访问特定于请求的值，例如最终用户的身份、授权令牌和请求的截止日期。 当请求被取消或超时时，处理该请求的所有 goroutine 都应该快速退出，以便系统可以回收它们正在使用的任何资源。

在 Google，我们开发了一个context包，可以轻松地将请求范围的值、取消信号和截止日期跨 API 边界传递给处理请求所涉及的所有 goroutine。 该包作为上下文公开可用。 本文介绍了如何使用该包并提供了一个完整的工作示例。

### Context

context 包的核心是 Context 类型：

```golang
// context携带截止日期、取消信号和请求范围的值
// 跨越 API 边界。 它的方法对于多人同时使用是安全的
// 协程。
type Context interface {
    // Done 返回一个当这个 Context 被取消时关闭的通道
    // 或超时。
    Done() <-chan struct{}

    // Err 表示在 Done 通道之后取消此上下文的原因
    // 已关闭。
    Err() error

    // Deadline 返回此 Context 将被取消的时间，如果截止时间设置的话。
    Deadline() (deadline time.Time, ok bool)

    // Value 返回与 key 关联的值，如果没有，则返回 nil。
    Value(key interface{}) interface{}
}
```

（这个描述是浓缩的；[godoc](https://golang.org/pkg/context) 是权威的。）

Done 方法返回一个通道，该通道作为代表上下文运行的函数的取消信号：当通道关闭时，函数应该放弃它们的工作并返回。 Err 方法返回一个错误，指示取消上下文的原因。 管道和取消文章更详细地讨论了 Done 通道习语。

Context 没有 Cancel 方法，原因与 Done 通道是只接收的原因相同：接收取消信号的函数通常不是发送信号的函数。 特别是，当父操作为子操作启动 goroutine 时，这些子操作应该无法取消父操作。 相反，WithCancel 函数（如下所述）提供了一种取消新 Context 值的方法。

Context对于多个 goroutine 同时使用是安全的。 代码可以将单个 Context 传递给任意数量的 goroutine 并取消该 Context 以向所有这些 goroutine 发出信号。

Deadline 方法允许函数决定它们是否应该开始工作； 如果剩下的时间太少，可能就不值得了。 代码还可以使用截止日期来设置 I/O 操作的超时时间。

Value允许上下文携带请求范围的数据。 该数据必须是安全的，以便多个 goroutine 同时使用。

### Derived contexts

context包提供了从现有值派生新Context值的函数。 这些值形成一棵树：当一个 Context 被取消时，所有从它派生的 Context 也被取消。

Background是任何Context的根； 它永远不会被取消：

```golang
// Background返回一个空的上下文。 它永远不会被取消，没有截止日期，
// 并且没有值。 Background通常用于主、初始化和测试，
// 并作为传入请求的顶级Context。
func Background() Context
```

WithCancel 和 WithTimeout 返回派生的 Context 值，这些值可以比父 Context 更早取消。 与传入请求关联的Context通常在请求处理程序返回时被取消。 WithCancel 也可用于在使用多个副本时取消冗余请求。 WithTimeout 可用于设置后端服务器请求的截止日期：

```golang
// WithCancel 通过父级context返回其 新context 并携带cancel 取消方法
// parent.Done 关闭或取消被调用。
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// CancelFunc 取消上下文。
type CancelFunc func()

// WithTimeout 返回父context的副本，其 Done channel 在 parent.Done 关闭、调用取消或超时后立即关闭。
// 新context deadline 是 now+timeout 和父级context 的deadline（如果有）中的较早者。 
// 如果计时器仍在运行，则取消函数会释放其资源。
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

WithValue 提供了一种将请求范围的值与context相关联的方法：

```golang
// WithValue 返回父级context的副本，其 Value 方法为 key 返回 val。
func WithValue(parent Context, key interface{}, val interface{}) Context
```

查看如何使用 context 包的最佳方法是通过一个工作示例。


### 示例：Google 网页搜索

我们的示例是一个 HTTP 服务器，它处理的url像 /search?q=golang&timeout=1s 并将查询参数"golang"转发到  Google Web Search API 呈现结果，timeout 参数告诉服务器在超时后取消请求。

代码分为三个包：

* [server](https://blog.golang.org/context/server/server.go) 为 /search 提供 main 函数和处理程序。
* [userip](https://blog.golang.org/context/userip/userip.go) 提供从请求中提取用户 IP 地址并将其与附加到Context中
* [google](https://blog.golang.org/context/google/google.go) 提供了用于向 Google 发送查询的搜索功能。

### server端程序

