# 源码阅读-Go Context

> go/src/context/context.go

context pkg 整体代码量比较少 首先来看下 context pkg 的组成

其中定义了三个 interface:

```text
type Context interface {...}

type canceler interface {...}
type stringer interface {...}
```

和三个继承了 Context 的 struct:

```text
// 实现了 canceler interface
type cancelCtx struct {
type timerCtx struct {

type valueCtx struct {
```

上面结构构成了 context pkg 主要的功能主体

### Context interface

> A Context carries a deadline, a cancellation signal, and other values across API boundaries.

```text
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

interface 定义的四个方法都是操作幂等的（successive calls to Method return the same response.）

`Deadline()` 方法可以用来实现 timeout 相关逻辑 `Done()` 方法会返回一个只读的 channel，用来同步 Context 结束状态 `Err()` 方法用来返回 Context 是为什么 close 的，正常情况下返回 nil `Value(key interface{})` 方法可以用来实现不同协程上下文间变量传递，最常见的是用来传 trace id

#### Empty Context

在实际使用中，我们可能会经常用到 Context pkg 的两个方法：

```text
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```

实际上，这两个方法在行为上是一模一样的，都是返回一个 empty struct 不过在含义上，这两个方法拥有不同的意义及推荐用法：

`Background()` 一般用来创建 top-level Context \(Context 的 root node\) 或测试 `TODO()` 则一般用作占位，由于 Context 官方推荐用法是放在函数的第一参数位且不推荐传递 nil，当重构代码或其他情况没有可用的 ctx 的时候，可以先用来占位

回到上文，emptyCtx 实际上就是个实现了 Context 的实体

```text
type emptyCtx int
func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}
func (*emptyCtx) Done() <-chan struct{} {
    return nil
}
func (*emptyCtx) Err() error {
    return nil
}
func (*emptyCtx) Value(key interface{}) interface{} {
    return nil
}
```

所有方法实现均什么也不做直接返回空值 简单来说它就是一个空的 Context，永远不会 canceled，没有 values，没有 deadline 可以看到 emptyCtx 并未被导出，一般我们在包外都是通过 `Background()` 和 `TODO()` 来使用初始化好的 emptyCtx

接下来看下经常用到的几个包方法

```text
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {...}
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {...}
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {...}
func WithValue(parent Context, key, val interface{}) Context {...}
```

`WithCancel` 用来创建一个 cancelable 的 Context `WithDeadline` 用来创建一个有 deadline 的 Context `WithTimeout` 是对 WithDeadline 进一步封装的糖果方法 `WithValue` 用来创建存储了 k-v 的 Context

### Cancel Context

cancelable 的 ctx 依托于 `cancelCtx` struct 实现，先看下这个 struct：

```text
type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
}

type cancelCtx struct {
    Context

    mu       sync.Mutex            // protects following fields
    done     chan struct{}         // created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}
```

cancelCtx 直接将 interface Context 作为一个匿名属性，所以它也是一个 Context

同时 cancelCtx 实现了 canceler interface

```text
func (c *cancelCtx) Value(key interface{}) interface{} {...}
func (c *cancelCtx) Done() <-chan struct{} {...}
func (c *cancelCtx) Err() error {...}
func (c *cancelCtx) cancel(removeFromParent bool, err error) {...}
```

其中最重要的是 cancel 方法（下面源码中 \[Comment By Eli\] 部分为追加注释）

```text
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    // [Comment By Eli] cancel context 时，必须传入 err 指明原因
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    c.err = err
    // [Comment By Eli] 此处处理是由于 c.done 是懒加载的，只有第一次调用 Done() 方法时才会初始化 channel
    // [Comment By Eli] closedchan 是包级别变量，定义了一个可充用的已经 closed 的 channel
    if c.done == nil {
        c.done = closedchan
    } else {
        close(c.done)
    }
    // [Comment By Eli] 把所有 child context 一并 cancel 掉（递归式的，包括 child 的 childs）
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    // [Comment By Eli] 为 true 此处会把当前 context 从 parent 的 children map 中删除
    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```

`cancel` 方法总结： 1. 设置当前 ctx.err 来指明 cancel 的原因 2. close 掉当前 ctx.done 这个 channel，以广播给监听该 channel 的人发送关闭通知 3. 递归式的把 child ctx 全部 cancel 掉 4. 把自己从父节点的 children map 中移除

先来看下一个 cancelable Context 是如何创建的，代码很简单

```text
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    // [Comment By Eli] 必须有一个父节点
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    c := newCancelCtx(parent)
    // [Comment By Eli] 向上找一个 cancelCtx 把自己挂上去
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}

func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{Context: parent}
}
```

创建方法很简单，主要做了两件事： 1. 初始化一个 cancelCtx struct，`cancelCtx{Context: parent}` 2. 向上找一个可用的 cancelCtx 把自己挂上去

挂载这个动作涉及到 context 的运作方式，在看 `propagateCancel()` 方法之前，我们先把其他几个 ctx 看完

### Deadline Context

with deadline 的 ctx 依托于 `timerCtx` struct 实现，先看下这个 struct：

```text
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}
```

`timerCtx` 内嵌了 cancelCtx 以继承它的基础方法如：`Done(), Err(), Value()` `timerCtx` 还包装了 cancelCtx 的 `cancel()` 方法来实现 timer 它提供的主要功能就是一个可定时关闭的 cancelCtx

我们来看下它的 `cancel()` 方法：

```text
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    // [Comment By Eli] 调用继承来的 cancel 方法来关闭自己
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    // [Comment By Eli] 加入计时器未关闭，就关掉它节省资源同时避免再次触发 cancel
    if c.timer != nil {
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

方法实现很简单，来看下创建逻辑：

```text
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }

    // [Comment By Eli] 假如父节点的 Deadline 比当前要创建的早，就直接创建一个 cancelCtx（因为父节点 Deadline 到期时就会 cancel 掉子节点）
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    propagateCancel(parent, c)
    dur := time.Until(d)
    // [Comment By Eli] 已经 Deadline 已经到期了，直接 cancel 掉
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(false, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    // [Comment By Eli] 起一个 timer，当定时到期时执行 cancel
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

主要逻辑有两个地方需要注意： 1. 加入要创建的 ctx deadline 比当前的 parent deadline 要晚，由于 parent cancel 的时候会把 chailds 都 cancel 掉，那么这个 deadline 就没有意义了，因为 parent 一定比它早结束。这时候就直接创建一个 cancelCtx 就行了 `return WithCancel(parent)` 2. 另一个就是定时自动关闭的核心，使用 Timer 的 [func AfterFunc](https://golang.org/pkg/time/#AfterFunc) 方法来实现定时逻辑

而另一个 `WithTimeout` 则是对 `WithDeadline` 的封装

```text
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

### Value Context

value ctx 主要实现为 context 提供 K-V 数据存储传递功能

```text
type valueCtx struct {
    Context
    key, val interface{}
}
```

它的主要方法 `Value` 实现如下：

```text
func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

实现非常简单，就是递归的向上查找对应 key 的 val，直到找到或返回 nil

创建方法逻辑也很简洁

```text
func WithValue(parent Context, key, val interface{}) Context {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    if key == nil {
        panic("nil key")
    }
    if !reflectlite.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}
```

需要注意的就是 key 的类型一定要是可比较的，因为后面 get 的时候需要用到 `==`

### How It Works

从上面源码中创建 ctx 的方式可以看出来，ctx 在整体运行中是链式结构的，更加准确的说其实是树形结构

```text
                       -- cancelCtx2
      -- cancelCtx1 --|
     |                 -- ctx4
ctx--|
      -- deadlineCtx1 -- cancelCtx3 -- deadlineCtx2
```

了解了 ctx 的运行时结构，我们再来看之前遗留的几个方法： 先看下 `propagateCancel`，这个方法在创建几种 cancelCtx 时都会调用

```text
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
    // [Comment By Eli] 加入父节点并不是 cancelCtx，就直接返回了
    done := parent.Done()
    if done == nil {
        return // parent is never canceled
    }

    // [Comment By Eli] 否则就监听父节点的 cancel 状态
    select {
    case <-done:
        // parent is already canceled
        child.cancel(false, parent.Err())
        return
    default:
    }

    // [Comment By Eli] 找出潜在的 cancelCtx，把自己加入到其 children map
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            // parent has already been canceled
            child.cancel(false, p.err)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else {
        // [Comment By Eli] 这里我还不是特别理解，上面已经监听了 parent.Done()，为什么这里还要再单独起一个 goroutine 再去监听
        atomic.AddInt32(&goroutines, +1)
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}
```

`propagateCancel()` 方法的作用主要是： 1. 监听父节点的 close 状态，保证自身与父节点状态保持一致 2. 如果父节点是 `cancelCtx` 把自己加入到父节点的 children map

最后来看下 parentCancelCtx

```text
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    done := parent.Done()
    if done == closedchan || done == nil {
        return nil, false
    }
    // [Comment By Eli] 这里通过 cancelCtx.Value 方法来获取潜在的 cancelCtx
    p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
    if !ok {
        return nil, false
    }
    p.mu.Lock()
    ok = p.done == done
    p.mu.Unlock()
    if !ok {
        return nil, false
    }
    return p, true
}

// [Comment By Eli] cancelCtx.Value，此处把该方法摘出来帮助理解
func (c *cancelCtx) Value(key interface{}) interface{} {
    if key == &cancelCtxKey {
        return c
    }
    return c.Context.Value(key)
}
```

### 小结与思考

#### 使用 context 应该遵循的规则

官方 \[pkg doc\]\[2\] 也给出了使用规范： 1. Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it. The Context should be the first parameter, typically named ctx: 2. Do not pass a nil Context, even if a function permits it. Pass context.TODO if you are unsure about which Context to use. 3. Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions. 4. The same Context may be passed to functions running in different goroutines; Contexts are safe for simultaneous use by multiple goroutines.

#### 关于传递共享数据

先说结论：还是要谨慎使用 `valueCtx`

`valueCtx` 提供了传递共享数据的方式，它的工作方式上文中也有提及，简单来说就是每次调用 `WithValue` 都会创建一个节点，其中存储了一对 K-V

```text
ctx--valueCtx1--valueCtx2--valueCtx3--valueCtx4
```

而其查找方式是递归的向上查找对应 key 的 val，当匹配到理它最近的节点，就会返回

如果我们在代码里滥用 `valueCtx` 进行数据共享，我们的数据状态是非常混乱的，甚至还可能存在不同时机插入同样的 key 导致在链条不同位置取出的 val 不一致的情况，极大增加后续的可维护性

\[2\]: [https://golang.org/pkg/context/](https://golang.org/pkg/context/)

