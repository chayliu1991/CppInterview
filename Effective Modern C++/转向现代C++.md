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

# 用 enum class 替代 enum

一般在大括号中声明的名称，只在大括号的作用域内可见，但这对  enum 成员例外。enum 成员属于 enum 所在的作用域，因此作用域内不能出现同名实例。

```
enum X { a, b, c };
int a = 1; //@ 错误：a已在作用域内声明过
```

C++11 引入了限定作用域的枚举类型，用 enum class 关键字表示：

```
enum class X { a, b, c };
int a = 1; //@ OK
X x = X::a; //@ OK
X y = b; //@ 错误
```

enum class`的另一个优势是不会进行隐式转换：

```
enum X { a, b, c };
X x = a;
if (x < 3.14) ... //@ 不应该将枚举与浮点数进行比较，但这里合法

enum class Y { a, b, c };
Y y = Y::a;
if (x < 3.14) ... //@ 报错：不允许比较
//@ 但enum class允许强制转换为其他类型
if (static_cast<double>(x) < 3.14) ... //@ OK
```

C++11 之前的 enum 不允许前置声明，而 C++11 的 enum 和 enum class 都可以前置声明：

 ```
enum Color; //@ C++11之前错误
enum class X; //@ OK
 ```

C++11 之前不能前置声明 enum 的原因是编译器为了节省内存，要在 enum  被使用前选择一个足够容纳成员取值的最小整型作为底层类型。

```
enum X { a, b, c }; //@ 编译器选择底层类型为char
enum Status { //@ 编译器选择比char更大的底层类型
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    indeterminate = 0xFFFFFFFF
};
```

不能前置声明的一个弊端是，由于编译依赖关系，在 enum 中仅仅添加一个成员可能就要重新编译整个系统。如果在头文件中包含前置声明，修改 enum class 的定义时就不需要重新编译整个系统，如果 enum class 的修改不影响函数的行为，则函数的实现也不需要重新编译。

C++11 支持前置声明的原因很简单，底层类型是已知的，用 std::underlying_type 即可获取。也可以指定枚举的底层类型，如果不指定，enum class 默认为  int，enum  则不存在默认类型。

```
enum class X : std::uint32_t;
//@ 也可以在定义中指定
enum class Y : std::uint32_t { a, b, c };
```

C++11 中使用 enum 更方便的场景只有一种，即希望用到 enum 的隐式转换时：

```
enum X { name, age, number };
auto t = std::make_tuple("downdemo", 6, "13312345678");
auto x = std::get<name>(t); //@ get的模板参数类型是std::size_t，name可隐式转换为std::size_t
```

如果用 enum class，则需要强制转换：

```
enum class X { name, age, number };
auto t = std::make_tuple("downdemo", 6, "13312345678");
auto x = std::get<static_cast<std::size_t>(X::name)>(t);
```

可以用一个函数来封装转换的过程，但也不会简化多少：

```
template<typename E>
constexpr auto f(E e) noexcept
{
	return static_cast<std::underlying_type_t<E>>(e);
}

auto x = std::get<f(X::name)>(t);
```

# 用 =delete 替代 private 作用域来禁用函数

C++11 之前禁用拷贝的方式是将拷贝构造函数和拷贝赋值运算符声明在 private 作用域中，并且不给予具体的实现：

```
class A {
private:
    A(const A&); // 不需要定义
    A& operator=(const A&);
};
```

C++11 中可以直接将要删除的函数用 =delete 声明，习惯上会声明在 public 作用域中，这样在使用删除的函数时，会先检查访问权再检查删除状态，出错时能得到更明确的诊断信息。

```
class A {
public:
    A(const A&) = delete;
    A& operator(const A&) = delete;
};
```

private 作用域中的函数还可以被成员和友元调用，而 =delete 是真正禁用了函数，无法通过任何方法调用。

任何函数都可以用 =delete 声明，比如函数不想接受某种类型的参数，就可以删除对应类型的重载。

```
void f(int);
void f(double) = delete; //@ 拒绝double和float类型参数
f(3.14); //@ 错误
```

=delete 还可以禁止模板对某个类型的实例化：

```
template<typename T>
void f(T x) {}

template<>
void f<int>(int) = delete;

f(1); //@ 错误：使用已删除的函数
template<typename T>
void processPointer(T * ptr);
```

类内的函数模板也可以用这种方式禁用：

```
class A {
public:
template<typename T>
	void f(T x) {}
};

template<>
void A::f<int>(int) = delete;
```

当然，写在 private 作用域也可以起到禁用的效果：

```
class A {
public:
    template<typename T>
    void f(T x) {}
private:
    template<>
    void f<int>(int);
};
```

但把模板和特化置于不同的作用域不太合逻辑，与其效仿 =delete 的效果，不如直接用 =delete。

# 用 override 标记被重写的虚函数

虚函数的重写很容易出错，因为要在派生类中重写虚函数，必须满足一系列要求：

- 基类中必须有此虚函数
- 基类和派生类的函数名相同（析构函数除外）
- 函数参数类型相同
- const 属性相同
- 函数返回值和异常说明相同

C++11 多出一条要求：引用修饰符相同。引用修饰符的作用是，指定成员函数仅在对象为左值（成员函数标记为 &）或右值（成员函数标记为 &&）时可用：

```
class A {
public:
    void f()& { std::cout << 1 << "\n"; } //@ *this是左值时才使用
    void f()&& { std::cout << 2 << "\n"; } //@ *this是右值时才使用
};

A makeA() { return A{}; }

A a;
a.f(); //@ 1
makeA().f(); //@ 2
```

对于这么多的要求难以面面俱到，比如下面代码没有任何重写但可以通过编译：

```
class A {
public:
    virtual void f1() const;
    virtual void f2(int x);
    virtual void f3() &;
    void f4() const;
};

class B : public A {
public:
    virtual void f1();
    virtual void f2(unsigned int x);
    virtual void f3() &&;
    void f4() const;
};
```

为了保证正确性，C++11 提供了 override 来标记要重写的虚函数，如果未重写就不能通过编译：

```
class A {
public:
    virtual void f1() const;
    virtual void f2(int x);
    virtual void f3() &;
    virtual void f4() const;
};

class B : public A {
public:
    virtual void f1() const override;
    virtual void f2(int x) override;
    virtual void f3() & override;
    void f4() const override;
};
```

override 是一个 contextual keyword，只在特殊语境中保留，override 只有出现在成员函数声明末尾才有保留意义，因此如果以前的遗留代码用到了 override 作为名字，不用改名就可以升到 C++11。

```
class A {
public:
	void override(); //@ 在C++98和C++11中都合法
};
```

C++11 还提供了另一个 contextual keyword：final 可以用来指定虚函数禁止被重写：

```
class A {
public:
    virtual void f() final;
    void g() final; //@ 错误：final只能用于指定虚函数
};

class B : public A {
public:
    virtual void f() override; //@ 错误：f不可重写
};
```

final 还可以用于指定某个类禁止被继承：

```
class A final {};
class B : public A {}; //@ 错误：A禁止被继承
```

# 用 std::cbegin 和 std::cend 获取 const_iterator

需要迭代器但不修改值时就应该使用 const_iterator，获取和使用 const_iterator 十分简单：

```
std::vector<int> v{ 2, 3 };
auto it = std::find(std::cbegin(v), std::cend(v), 2); //@ C++14
v.insert(it, 1);
```

上述功能很容易扩展成模板：

```
template<typename C, typename T>
void f(C& c, const T& x, const T& y)
{
    auto it = std::find(std::cbegin(c), std::cend(c), x);
    c.insert(it, y);
}
```

# 用 noexcept 标记不抛异常的函数

C++98 中，必须指出一个函数可能抛出的所有异常类型，如果函数有所改动则 exception specification 也要修改，而这可能破坏代码，因为调用者可能依赖于原本的 exception specification，所以 C++98 中的 exception specification 被认为不值得使用。

C++11 中达成了一个共识，真正需要关心的是函数会不会抛出异常。一个函数要么可能抛出异常，要么绝对不抛异常，这种 maybe-or-never 形成了 C++11 exception specification 的基础，C++98 的 exception specification 在 C++17 移除。

函数是否要加上 noexcept 声明与接口设计相关，调用者可以查询函数的 noexcept 状态，查询结果将影响代码的异常安全性和执行效率。因此函数是否要声明为 noexcept 就和成员函数是否要声明为 const 一样重要，如果一个函数不抛异常却不为其声明 noexcept，这就是接口规范缺陷。

noexcept 的一个额外优点是，它可以让编译器生成更好的目标代码。为了理解原因只需要考虑 C++98 和 C++11 表达函数不抛异常的区别。

```
int f(int x) throw(); //@ C++98
int f(int x) noexcept; //@ C++11
```

如果一个异常在运行期逃出函数，则 exception specification 被违反。在 C++98 中，调用栈会展开到函数调用者，执行一些无关的动作后中止程序。C++11 的一个微小区别是是，在程序中止前只是可能展开栈。这一点微小的区别将对代码生成造成巨大的影响。

noexcept 声明的函数中，如果异常传出函数，优化器不需要保持栈在运行期的展开状态，也不需要在异常逃出时，保证其中所有的对象按构造顺序的逆序析构。而声明为 throw() 的函数就没有这样的优化灵活性。总结起来就是：

```
RetType function(params) noexcept; //@ most optimizable
RetType function(params) throw(); //@ less optimizable
RetType function(params); //@ less optimizable
```

这个理由已经足够支持给任何已知不会抛异常的函数加上 noexcept，比如移动操作就是典型的不抛异常函数。

std::vector::push_back 在容器空间不够容纳元素时，会扩展新的内存块，再把元素转移到新的内存块。C++98 的做法是逐个拷贝，然后析构旧内存的对象，这使得 push_back 提供强异常安全保证：如果拷贝元素的过程中抛出异常，则 std::vector 保持原样，因为旧内存元素还未被析构。

std::vector::push_back 在 C++11 中的优化是把拷贝替换成移动，但为了不违反强异常安全保证，只有确保元素的移动操作不抛异常时才会用移动替代拷贝。

swap 函数是需要 noexcept 声明的另一个例子，不过标准库的 swap 用 noexcept 操作符的结果决定。

```
//@ 数组的swap
template <class T, size_t N>
void swap(T(&a)[N], T(&b)[N]) noexcept(noexcept(swap(*a, *b))); //@ 由元素类型决定noexcept结果
//@ 比如元素类型是class A，如果swap(A, A)不抛异常则该数组的swap也不抛异常

// std::pair的swap
template <class T1, class T2>
struct pair {
    …
        void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
            noexcept(swap(second, p.second)));
    …
};
```

虽然 noexcept 有优化的好处，但将函数声明为 noexcept 的前提是，保证函数长期具有 noexcept 性质，如果之后随意移除 noexcept 声明，就有破坏客户代码的风险。

大多数函数是异常中立的，它们本身不抛异常，但它们调用的函数可能抛异常，这样它们就允许抛出的异常传到调用栈的更深一层，因此异常中立函数天生永远不具备 noexcept 性质。

如果为了强行加上 noexcept 而修改实现就是本末倒置，比如调用一个会抛异常的函数是最简单的实现，为了不抛异常而环环相扣地来隐藏这点（比如捕获所有异常，将其替换成状态码或特殊返回值），大大增加了理解和维护的难度，并且这些复杂性的时间成本可能超过 noexcept 带来的优化。

对某些函数来说，noexcept 性质十分重要，内存释放函数和所有的析构函数都隐式 noexcept，这样就不必加 noexcept声明。析构函数唯一未隐式 noexcept  的情况是，类中有数据成员的类型显式将析构函数声明 noexcept(false)。但这样的析构函数很少见，标准库中一个也没有。

有些库的接口设计者会把函数区分为 wide contract 和 `narrow contract：

- wide contrac， 函数没有前置条件，不用关心程序状态，对传入的实参没有限制，一定不会有未定义行为，如果知道不会抛异常就可以加上 noexcept。
- narrow contract， 函数有前置条件，如果条件被违反则结果未定义。但函数没有义务校验这个前置条件，它断言前置条件一定满足（调用者负责保证断言成立），因此加上 `noexcept` 声明也是合理的。

```
//@ 假设前置条件是s.length(//@ <= 32
void f(const std::string& s)//@ noexcept;
```

但如果想在违反前置条件时抛出异常，由于函数的 noexcept 声明，异常就会导致程序中止，因此一般只为 wide contract 函数声明 noexcept

在 noexcept 函数中调用可能抛异常的函数时，编译器不会帮忙给出警告：

```
void start();
void finish();
void f() noexcept
{
    start();
    … //@ do the actual work
    finish();
}
```

带 noexcept 声明的函数调用了不带 noexcept 声明的函数，这看起来自相矛盾，但也许被调用的函数在文档中写明了不会抛异常，也许它们来自 C 语言的库，也许来自还没来得及根据 C++11 标准做修订的 C++98 库。

# 用 constexpr 表示编译期常量

constexpr 用于对象时就是一个加强版的 const，表面上看 constexpr 表示值是 const，且在编译期（严格来说是翻译期，包括编译和链接，如果不是编译器或链接器作者，无需关心这点区别）已知，但用于函数则有不同的意义。

编译期已知的值可能被放进只读内存，这对嵌入式开发是一个很重要的语法特性。

 ```
int i = 42;
constexpr auto j = i; //@ 错误：i的值在编译期未知
std::array<int, i> v1; //@ 错误：同上
constexpr auto n = 10; //@ OK：10是一个编译期常量
std::array<int, n> v2; //@ OK：n的值是在编译期已知
 ```

constexpr 函数在调用时若传入的是编译期常量，则产出编译期常量，传入运行期才知道的值，则产出运行期值。constexpr 函数可以满足所有需求，因此不必为了有非编译期值的情况而写两个函数。

```
constexpr int pow(int base, int exp) noexcept
{
    … //@ 实现见后
}

constexpr auto n = 5;
std::array<int, pow(3, n)> results; //@ pow(3, n)在编译期计算出结果
```

上面的 constexpr 并不表示函数要返回 const 值，而是表示，如果参数都是编译期常量，则返回结果就可以当编译期常量使用，如果有一个不是编译期常量，返回值就在运行期计算。

```
auto base = 3; //@ 运行期获取值
auto exp = 10; //@ 运行期获取值
auto baseToExp = pow(base, exp); //@ pow在运行期被调用
```

C++11 中，constexpr 函数只能包含一条语句，即一条 return 语句。有两个应对限制的技巧：用条件运算符 ` ?: ` 替代 if-else、用递归替代循环。

```
constexpr int pow(int base, int exp) noexcept
{
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
}
```

C++14 解除了此限制：

```
//@ C++14
constexpr int pow(int base, int exp) noexcept
{
    auto result = 1;
    for (int i = 0; i < exp; ++i) 
    result *= base;
    return result;
}
```

constexpr 函数必须传入和返回 literal type。constexpr 构造函数可以让自定义类型也成为 literal type 。

```
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal) {}
    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }
    void setX(double newX) noexcept { x = newX; } //@ 修改了对象所以不能声明为constexpr
    void setY(double newY) noexcept { y = newY; } //@ 另外C++11中constexpr函数返回类型不能是void
private:
    double x, y;
};

constexpr Point p1(9.4, 27.7); //@ 编译期执行constexpr构造函数
constexpr Point p2(28.8, 5.3); //@ 同上

//@ 通过constexpr Point对象调用xValue和yValue也会在编译期获取值
//@ 于是可以再写出一个新的constexpr函数
constexpr Point midpoint(const Point& p1, const Point& p2) noexcept
{
    return { (p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2 };
}
constexpr auto mid = midpoint(p1, p2); //@ mid在编译期创建
```

因为 `mid` 是编译期已知值，这就意味着如下表达式可以用于模板形参：

```
mid.xValue()*10
//@ 因为上式是浮点型，浮点型不能用于模板实例化，因此还要如下转换一次
static_cast<int>(mid.xValue()*10)
```

C++14 允许对值进行了修改或无返回值的函数声明为 `constexpr`：

```
//@ C++14
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
    : x(xVal), y(yVal) {}
    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }
    constexpr void setX(double newX) noexcept { x = newX; }
    constexpr void setY(double newY) noexcept { y = newY; }
private:
    double x, y;
};

//@ 于是C++14允许写出下面的代码
constexpr Point reflection(const Point& p) noexcept //@ 返回p关于原点的对称点
{
    Point res;
    res.setX(-p.xValue());
    res.setY(-p.yValue());
    return res;
}

constexpr Point p1(9.4, 27.7);
constexpr Point p2(28.8, 5.3);
constexpr auto mid = midpoint(p1, p2);
constexpr auto reflectedMid = reflection(mid); //@ 值为(-19.1, -16.5)，且在编译期已知
```

使用 `constexpr` 的前提是必须长期保证需要它，因为如果后续要删除 `constexpr` 可能会导致许多错误。

















































































