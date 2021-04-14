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

大括号初始化的缺陷在于，只要类型转换后可以匹配，大括号初始化总会优先匹配参数类型为 [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list) 的构造函数，即使收缩转换会导致调用错误。

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

但特殊的是，参数为空的大括号初始化只会调用默认构造函数。如果想传入真正的空 [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list) 作为参数，则要额外添加一层大括号或小括号。

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

上述问题带来的实际影响很大，比如 [std::vector](https://en.cppreference.com/w/cpp/container/vector) 就存在参数为参数 [std::initializer_list](https://en.cppreference.com/w/cpp/utility/initializer_list) 的构造函数，这导致了参数相同时，大括号初始化和小括号初始化调用的却是不同版本的构造函数：

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

[std::make_shared ](https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared) 和 [std::make_unique](https://en.cppreference.com/w/cpp/memory/unique_ptr/make_unique)就面临了这个问题，而它们的选择是使用小括号初始化并在接口文档中写明这点：

```
auto p = std::make_shared<std::vector<int>>(3, 6);
for (auto x : *p) std::cout << x; //@ 666
```



