# 内存模型基础

为了避免 race condition，线程就要规定执行顺序。

- 一种方式是使用 mutex，后一线程必须等待前一线程解锁。
- 第二种方式是使用原子操作来避免竞争访问同一内存位置。

原子操作是不可分割的操作，要么做了要么没做，不存在做一半的状态。

# C++ 中的原子操作和原子类型

## 标准原子类型

标准原子类型定义在 <atomic> 中。也可以用 mutex 模拟原子操作，实际上标准原子类型可能就是这样实现的，它们都有一个 is_lock_free 函数，返回 true  说明该原子类型操作是无锁的，用的是原子指令，返回 false 则是用锁。

```
struct A { int a[1000]; };
struct B { int x, y; };

std::cout << std::boolalpha
    << std::atomic<A>{}.is_lock_free()
    << "\n"
    << std::atomic<B>{}.is_lock_free()
    << "\n";
```

原子操作的主要用处是替代 mutex 实现同步。如果原子操作内部是用 mutex 实现的，就不会有期望的性能提升，还不如直接用 mutex 来同步。C++17 中每个原子类型都有一个 is_always_lock_free  成员变量，为 true 时表示该原子类型在此平台上 lock-free。

```
std::cout << std::atomic<int>{}.is_always_lock_free; //@ 1
```

C++17 之前可以用标准库为各个原子类型定义的 ATOMIC_xxx_LOCK_FREE 宏来判断该类型是否无锁：

- 值为0表示原子类型是有锁的。
- 值为2表示无锁。
- 值为1表示运行时才能确定。

```
//@ VS2017中的定义
#define ATOMIC_BOOL_LOCK_FREE        2
#define ATOMIC_CHAR_LOCK_FREE        2
#define ATOMIC_CHAR16_T_LOCK_FREE    2
#define ATOMIC_CHAR32_T_LOCK_FREE    2
#define ATOMIC_WCHAR_T_LOCK_FREE     2
#define ATOMIC_SHORT_LOCK_FREE       2
#define ATOMIC_INT_LOCK_FREE         2
#define ATOMIC_LONG_LOCK_FREE        2
#define ATOMIC_LLONG_LOCK_FREE       2
#define ATOMIC_POINTER_LOCK_FREE     2
```

标准库中为 std::atomic 对内置类型的特化定义了类型别名，比如：

```
namespace std {
    using atomic_bool = std::atomic<bool>;
    using atomic_char = std::atomic<char>;
}
```

通常类型 `std::atomic<T>` 的别名就是 atomic_T，只有以下几种例外：signed 缩写为 s，unsigned 缩写为 u，long long 缩写为 llong：

```
namespace std {
    using atomic_schar = std::atomic<signed char>;
    using atomic_uchar = std::atomic<unsigned char>;
    using atomic_uint = std::atomic<unsigned>;
    using atomic_ushort = std::atomic<unsigned short>;
    using atomic_ulong = std::atomic<unsigned long>;
    using atomic_llong = std::atomic<long long>;
    using atomic_ullong = std::atomic<unsigned long long>;
}
```









