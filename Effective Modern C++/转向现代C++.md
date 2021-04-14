# 创建对象时注意区分()和{}

值初始化有如下方式：

```
int a(0); //@ 初始化值在小括号中
int b = 0; //@ 初始化值在等号后
int c{ 0 }; //@ 初始化值在大括号中
int d = { 0 }; //@ 按int d{ 0 }处理，后续讨论将忽略这种用法
```

使用等号不一定是赋值，也可能是拷贝，对于内置类型来说，初始化和赋值的区别只是学术争议，但对于类类型则不同：

```
X a; //@ 默认构造
X b = a; //@ 拷贝
a = b; //@ 赋值
```

C++11引入了统一初始化，也可以叫大括号初始化。大括号初始化可以方便地为容器指定初始元素：

```
std::vector<int> v{ 1, 2, 3 };
```

大括号初始化同样能为 non-static 数据成员指定默认值，也可以用 = 指定，但不能用小括号初始化指定：

```
class A {
    int x{ 0 }; //@ OK
    int y = 0; //@ OK
    int z(0); //@ 错误
};
```

大括号初始化禁止内置类型的隐式收缩转换，而小括号初始化和 = 不会禁止：

```
double x = 1.1;
double y = 2.2;
int a{ x + y }; //@ 错误：大括号初始化不允许double到int的收缩转换
int b(x + y); //@ OK：double被截断为int
int c = x + y; //@ OK：double被截断为int
```

大括号初始化不用担心 C++'s most vexing parse：

```
class A {
public:
    A() { std::cout << 1; }
};

class B {
public:
    B(std::string) { std::cout << 2; }
};

A a(); //@ 不调用A的构造函数，而是被解析成一个函数声明：A a();
std::string s("hi");
B b(std::string(s)); //@ 不调用B的构造函数，而是被解析成一个函数声明：B b(std::string);

A a2{}; //@ 调用A的构造函数
B b2{ std::string(s) }; //@ 调用B的构造函数

//@ C++11之前的解决办法
A a3;
B b3((std::string(s)));
```

大括号初始化的缺陷在于，只要类型转换后可以匹配，大括号初始化总会优先匹配参数类型为 std::initializer_list 的构造函数，即使收缩转换会导致调用错误。

```
class A {
public:
	A(int) { std::cout << 1 << "\n"; }
	A(std::string) { std::cout << 2 << "\n"; }
	A(std::initializer_list<int>) { std::cout << 3 << "\n"; }
};

A a{ 0 }; //@ 3
A b{ 3.14 }; //@ 错误：大括号初始化不允许double到int的收缩转换
A c{ "hi" }; //@ 2
```

但特殊的是，参数为空的大括号初始化只会调用默认构造函数。如果想传入真正的空 std::initializer_list 作为参数，则要额外添加一层大括号或小括号。

```
class A {
public:
    A() { std::cout << 1 <<"\n"; }
    A(std::initializer_list<int>) { std::cout << 2 << "\n"; }
};
A a{}; //@ 1
A b{ {} }; //@ 2
A c({}); //@ 2
```

上述问题带来的实际影响很大，比如 std::vector 就存在参数为参数 std::initializer_list 的构造函数，这导致了参数相同时，大括号初始化和小括号初始化调用的却是不同版本的构造函数：

```
std::vector<int> v1(3, 6); //@ 元素为3个6
std::vector<int> v2{ 3, 6 }; //@ 元素为3和6
```

这是一种失败的设计，并给模板作者带来了对大括号初始化和小括号初始化的选择困惑：

```
template<typename T, typename... Ts>
decltype(auto) f(Ts &&... args)
{
    T x(std::forward<Ts>(args)...); //@ 用小括号初始化创建临时对象
    return x;
}

template<typename T, typename... Ts>
decltype(auto) g(Ts &&... args)
{
    T x{ std::forward<Ts>(args)... }; //@ 用大括号初始化创建临时对象
    return x;
}

//@ 模板作者不知道调用者希望得到哪个结果
auto v1 = f<std::vector<int>>(3, 6); //@ v1元素为6、6、6
auto v2 = g<std::vector<int>>(3, 6); //@ v2元素为3、6
```

std::make_shared 和 std::make_unique 就面临了这个问题，而它们的选择是使用小括号初始化并在接口文档中写明这点：

```
auto p = std::make_shared<std::vector<int>>(3, 6);
for (auto x : *p) std::cout << x; //@ 666
```

# 用 nullptr 替代 0 和 NULL

字面值0本质是 int 而非指针，只有在使用指针的语境中发现 0 才会解释为空指针。

NULL 的本质是宏，没有规定的实现标准，一般在 C++ 中定义为 0，在 C 中定义为 void*。

```
//@ VS2017中的定义
#ifndef NULL
#ifdef __cplusplus
#define NULL 0
#else
#define NULL ((void *)0)
#endif
#endif
```

在重载解析时，NULL 作为参数不会优先匹配指针类型。而 nullptr 的类型是 std::nullptr_t，std::nullptr_t 可以转换为任何原始指针类型。

```
void f(bool) { std::cout << 1 << "\n"; }
void f(int) { std::cout << 2 << "\n"; }
void f(void*) { std::cout << 3 << "\n"; }
f(0); //@ 2
f(NULL); //@ 2
f(nullptr); //@ 3
```

这点也会影响模板实参推断：

```
template<typename T>
void f() {}

f(0); //@ T推断为int
f(NULL); //@ T推断为int
f(nullptr); //@ T推断为std::nullptr_t
```

使用 nullptr 就可以避免推断出非指针类型：

 ```
void f1(std::shared_ptr<int>) {}
void f2(std::unique_ptr<int>) {}
void f3(int*) {}

template<typename F, tpyename T>
void g(F f, T x)
{
	f(x);
}

g(f1, 0); //@ 错误
g(f1, NULL); //@ 错误
g(f1, nullptr); //@ OK

g(f2, 0); //@ 错误
g(f2, NULL); //@ 错误
g(f2, nullptr); //@ OK

g(f3, 0); //@ 错误
g(f3, NULL); //@ 错误
g(f3, nullptr); //@ OK
 ```

使用 nullptr  也能使代码意图更清晰：

```
auto res = f();
if (res == nullptr) ... //@ 很容易看出res是指针类型
```

# 用 using 别名声明替代 typedef

using 别名声明比 typedef 可读性更好，尤其是对于函数指针类型：

```
typedef void (*F)(int);
using F = void (*)(int);
```

C++11 还引入了模板别名，它只能使用 using 别名声明：

```
template<typename T>
using X = std::vector<T>; //@ X<int>等价于std::vector<int>

//@ C++11之前的做法是在模板内部typedef
template<typename T>
struct Y { //@ Y<int>::type等价于std::vector<int>
	typedef std::vector<T> type;
};

//@ 在其他类模板中使用这两个别名的方式
template<typename T>
class A {
	X<T> x;
	typename Y<T>::type y;
};
```

C++11 引入了 type traits，为了方便使用，C++14 为每个 type traits 都定义了模板别名：

```
//@ std::remove_reference的实现
template<typename T>
struct remove_reference {
	using type = T;
};

template<typename T>
struct remove_reference<T&> {
	using type = T;
};

template<typename T>
struct remove_reference<T&&> {
	using type = T;
};

//@ std::remove_reference_t的实现
template<typename T>
using remove_reference_t = typename remove_reference<T>::type;
```

为了简化生成值的 type traits，C++14 还引入了模板变量：

```
//@ std::is_same的实现
template<typename T, typename U>
struct is_same {
	static constexpr bool value = false;
};

//@ std::is_same_v的实现
template<typename T>
constexpr bool is_same_v = is_same<T, U>::value;
```





