# 用 std::async 替代 std::thread

异步运行函数的一种选择是，创建一个 std::thread 来运行。

```
int f();
std::thread t(f);
```

另一种方法是使用 std::async，它返回一个持有计算结果的 std::future：

```
int f();
std::future<int> ft = std::async(f);
```

如果函数有返回值，std::thread 无法直接获取该值，而 std::async  返回的 std::future 提供了 get 来获取该值。如果函数抛出异常，get 能访问异常，而 std::thread会调用 std::terminate 终止程序：

```
int f() { return 1; }
auto ft = std::async(f);
int res = ft.get();
```

在并发的C++软件中，线程有三种含义：

- hardware thread 是实际执行计算的线程，计算机体系结构中会为每个 CPU 内核提供一个或多个硬件线程。
- software thread（OS thread 或 system thread）是操作系统实现跨进程管理，并执行硬件线程调度的线程。
- std::thread 是 C++ 进程中的对象，用作底层 OS thread 的 handle。

OS thread 是一种有限资源，如果试图创建的线程超出系统所能提供的数量，就会抛出 std::system_error 异常。这在任何时候都是确定的，即使要运行的函数不能抛异常。

```
int f() noexcept;
std::thread t(f); //@ 若无线程可用，仍会抛出异常
```

解决这个问题的一个方法是在当前线程中运行函数，但这会导致负载不均衡，而且如果当前线程是一个 GUI 线程，将导致无法响应。另一个方法是等待已存在的软件线程完成工作后再新建 std::thread，但一种可能的问题是，已存在的软件线程在等待函数执行某个动作。

即使没有用完线程也可能发生 oversubscription 的问题，即准备运行（非阻塞）的 OS thread 数量超过了 hardware thread，此时线程调度器会为 OS thread 在 hardware thread 上分配 CPU 时间片。当一个线程的时间片用完，另一个线程启动时，就会发生语境切换。这种语境切换会增加系统的线程管理开销，尤其是调度器切换到不同的 CPU core 上的硬件线程时会产生巨大开销。此时，OS thread 通常不会命中 CPU cache（即它们几乎不含有对该软件线程有用的数据和指令），CPU core 运行的新软件线程还会污染 cache 上为旧线程准备的数据，旧线程曾在该 CPU core 上运行过，并很可能再次被调度到此处运行。

避免 oversubscription 很困难，因为 OS thread 和 hardware thread 的最佳比例取决于软件线程变为可运行状态的频率，而这是会动态变化的，比如一个程序从 I/O 密集型转换计算密集型。软件线程和硬件线程的最佳比例也依赖于语境切换的成本和使用 CPU cache 的命中率，而硬件线程的数量和 CPU cache 的细节（如大小、速度）又依赖于计算机体系结构，因此即使在一个平台上避免了 oversubscription 也不能保证在另一个平台上同样有效。

使用 std::async 则可以把 oversubscription 的问题丢给库作者解决：

```
auto ft = std::async(f); //@ 由标准库的实现者负责线程管理
```

这个调用把线程管理的责任转交给了标准库实现。如果申请的软件线程多于系统可提供的，系统不保证会创建一个新的软件线程。相反，它允许调度器把函数运行在对返回的 std::future  调用 get 或 wait 的线程中。

即使使用 std::async，GUI 线程的响应性也仍然存在问题，因为调度器无法得知哪个线程迫切需要响应。这种情况下，可以将 std::async 的启动策略设定为std::launch::async，这样可以保证函数会在调用 get 或 wait 的线程中运行。

```
auto ft = std::async(std::launch::async, f);
```

std::async 分担了手动管理线程的负担，并提供了检查异步执行函数的结果的方式，但仍有几种不常见的情况需要使用 std::thread：

- 需要访问底层线程 API：并发 API 通常基于系统的底层 API（pthread、Windows 线程库）实现，通过 std::thread::native_handle 即可获取底层线程 handle。
- 需要为应用优化线程用法：比如开发一个服务器软件，运行时的 profile 已知并作为唯一的主进程部署在硬件特性固定的机器上。
- 实现标准库未提供的线程技术，比如线程池。

# 用 std::launch::async 指定异步求值

std::async  有两种标准启动策略：

- std::launch::async：函数必须异步运行，即运行在不同的线程上。
- std::launch::deferred：函数只在对返回的 std::future 调用 get 或 wait 时运行。即执行会推迟，调用 get 或 wait 时函数会同步运行，调用方会阻塞至函数运行结束。

std::async 的默认启动策略不是二者之一，而是对二者求或的结果。

```
auto ft1 = std::async(f); //@ 意义同下
auto ft2 = std::async(std::launch::async | std::launch::deferred, f);
```

默认启动策略允许异步或同步运行函数，这种灵活性使得 std::async 和标准库的线程管理组件能负责线程的创建和销毁、负载均衡以及避免 oversubscription。

但默认启动策略存在一些潜在问题，比如给定线程 `t` 执行如下语句：

```
auto ft = std::async(f);
```

潜在的问题有：

- 无法预知 f 和 t 是否会并发运行，因为 f 可能被调度为推迟运行。
- 无法预知运行 f 的线程是否不同于对 ft 调用 get 或 wait 的线程，如果调用 get 或 wait 的线程是 t，就说明无法预知 f 是否会运行在与 t 不同的某线程上。
- 甚至很可能无法预知 f 是否会运行，因为无法保证在程序的每条路径上，ft 的 get 或 wait 会被调用。

默认启动策略在调度上的灵活性会在使用 thread_local 变量时导致混淆，这意味着如果 f 读写此 thread-local

storage（TLS）时，无法预知哪个线程的局部变量将被访问。

```
auto ft = std::async(f); //@ f的TLS可能和一个独立线程相关，但也可能与对ft调用get或wait的线程相关
```

它也会影响使用 timeout 的 wait-based 循环，因为对返回的 std::future 调用 wait_for 或 wait_until 值。这意味着以下循环看似最终会终止，但实际可能永远运行：

```
using namespace std::literals;

void f()
{
    std::this_thread::sleep_for(1s);
}

auto ft = std::async(f);
while (ft.wait_for(100ms) != std::future_status::ready)
{ //@ 循环至f运行完成，但这可能永远不会发生
    …
}
```

如果选用了 std::launch::async 启动策略，f 和调用 std::async  的线程并发执行，则没有问题。但如果 f 被推迟执行，则 wait_for 总会返回 std::future_status::deferred，于是循环永远不会终止。

这类 bug 在开发和单元测试时很容易被忽略，只有在运行负载很重时才会被发现。解决方法很简单，检查返回的[std::future](https://en.cppreference.com/w/cpp/thread/future)，确定任务是否被推迟。但没有直接检查是否推迟的方法，替代的手法是，先调用一个 timeout-based 函数，比如 [wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for)，这并不表示想等待任何事，而只是为了查看返回值是否为 [std::future_status::deferred](https://en.cppreference.com/w/cpp/thread/future_status)。

```
auto ft = std::async(f);
if (ft.wait_for(0s) == std::future_status::deferred) //@ 任务被推迟
{
    … //@ 使用ft的wait或get异步调用f
}
else //@ 任务未被推迟
{
    while (ft.wait_for(100ms) != std::future_status::ready)
    {
        … //@ 任务未被推迟也未就绪，则做并发工作直至结束
    }
    … //@ ft准备就绪
}
```

综上，[std::async](https://en.cppreference.com/w/cpp/thread/async) 使用默认启动策略创建要满足以下所有条件：

- 任务不需要与对返回值调用 [get](https://en.cppreference.com/w/cpp/thread/future/get) 或 [wait](https://en.cppreference.com/w/cpp/thread/future/wait) 的线程并发执行
- 读写哪个线程的 thread_local 变量没有影响
- 要么保证对返回值调用 [get](https://en.cppreference.com/w/cpp/thread/future/get) 或 [wait](https://en.cppreference.com/w/cpp/thread/future/wait)，要么接受任务可能永远不执行
- 使用 [wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for) 或 [wait_until](https://en.cppreference.com/w/cpp/thread/future/wait_until) 的代码要考虑任务被推迟的可能

只要一点不满足，就可能意味着想确保异步执行任务，这只需要指定启动策略为 [std::launch::async](https://en.cppreference.com/w/cpp/thread/launch)。

```
auto ft = std::async(std::launch::async, f); //@ 异步执行f
```

默认使用 [std::launch::async](https://en.cppreference.com/w/cpp/thread/launch) 启动策略的 [std::async](https://en.cppreference.com/w/cpp/thread/async) 将会是一个很方便的工具，实现如下：

```
template<typename F, typename... Ts>
inline
auto // std::future<std::invoke_result_t<F, Ts...>>
reallyAsync(F&& f, Ts&&... args)
{
    return std::async(std::launch::async,
        std::forward<F>(f),
        std::forward<Ts>(args)...);
}
```

这个函数的用法和 [std::async](https://en.cppreference.com/w/cpp/thread/async) 一样：

```
auto ft = reallyAsync(f); //@ 异步运行f，如果std::async抛出异常则reallyAsync也抛出异常
```

















