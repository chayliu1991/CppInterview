# nullptr

C++ 中 NULL 的定义为：

```
#ifndef NULL
    #ifdef __cplusplus
        #define NULL 0
    #else
        #define NULL ((void *)0)
    #endif
#endif
```

由此可以在 C 语言中将 `NULL` 定义成 `(void *)0`，在 C++ 中 `NULL` 就是 0。
```
void foo(void* p)
{
	std::cout << "foo(void* p)" << endl;
}

void foo(int x)
{
	std::cout << "foo(int x)" << endl;
}

foo(NULL); //@ 将调用 foo(int x)
```

`nullptr` 不是整数类型，`nullptr`的确切类型是 `std::nullptr_t`，它可以隐式的转换为所有的原始指针类型。

```
foo(NULL); //@ 将调用 foo(void* p)
```

使用 `NULL` 在模板实参推导中也存在问题：

```
void f1(std::shared_ptr<int>) { std::cout << "f1" << endl; }
template<typename F, typename T>
void g(F f, T x)
{
	f(x);
}

g(f1, NULL); //@ T将被推导成 int 类型，错误
g(f1, nullptr); //@ OK
```



