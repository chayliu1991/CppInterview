# 用 auto 替代显式类型声明

auto 声明的变量必须初始化，因此使用 auto 可以避免忘记初始化的问题。

```
int a; //@ 潜在的未初始化风险
auto b; //@ 错误：必须初始化
```

对于名称非常长的类型，如迭代器相关的类型，用 auto 声明可以大大简化工作：

```
template<typename It>
void f(It b, It e)
{
    while (b != e)
    {
        auto currentValue = *b;
        //@ typename std::iterator_traits<It>::value_type currentValue = *b;
        ...
    }
}
```

lambda 生成的闭包类型是编译期内部的匿名类型，无法得知，使用 auto 推断就没有这个问题：

```
auto f = [](auto& x, auto& y) { return x < y; };
```

如果不使用 auto，可以改用 std::function：

```
//@ std::function的模板参数中不能使用auto
std::function<bool(int&, int&)> f = [](auto& x, auto& y) { return x < y; };
```

std::function 与 auto 的最大区别在于，auto 和闭包类型一致，内存量和闭包相同，而 std::function 是类模板，它的实例有一个固定大小，这个大小不一定能容纳闭包，于是会分配堆上的内存以存储闭包，导致比 auto 变量占用更多内存。此外，编译器一般会限制内联，std::function 调用闭包会比 auto 慢。

auto 可以避免简写类型存在的潜在问题。比如如下代码有潜在隐患：

```
std::vector<int> v;
unsigned sz = v.size(); //@ v.size()类型实际为std::vector<int>::size_type
//@ 在32位机器上std::vector<int>::size_type与unsigned尺寸相同
//@ 但在64位机器上，std::vector<int>::size_type是64位，而unsigned是32位
```

如下代码也有潜在问题：

```
std::unordered_map<std::string, int> m; //@ m的元素类型实际是std::pair<const std::string, int>
for (const std::pair<std::string, int>& p : m) ... //@ 类型不一致，仍要转换，期间要构造大量临时对象
```

如果显式类型声明能让代码更清晰或有其他好处就不用强行 auto，此外 IDE 的类型提示也能缓解不能直接看出对象类型的问题。

# auto 推断出非预期类型时，先强制转换出预期类型

如下代码没有问题：

 ```
bool x = f()[0];
if (x) 
	std::cout << "OK"<<"\n";
 ```

但如果把显式声明改为 auto 则会出现非预期行为：

```
std::vector<bool> f()
{
	return std::vector<bool>{ true, false };
}

auto x = f()[0]; //@ 改用auto声明
if (x) std::cout << "OK" << "\n"; //@ 错误：未定义行为
```

原因在于实际上得到的类型不是 bool：

```
auto x = f()[0]; //@ x类型为std::vector<bool>::reference
```

`std::vector<bool>` 不是真正的 STL 容器，也不包含 bool  类型元素。它是 std::vector 对于 bool 类型的特化，为了节省空间，每个元素用一个 bit（而非一个 bool ）表示，于是 operator[] 返回的应该是单个 bit 的引用，但 C++ 中不存在指向单个 bit 的指针，因此也不能获取单个 bit 的引用。

```
std::vector<bool> v { true, false };
bool* p = &v[0]; //@ 错误
std::vector<bool>::reference* q = &v[0]; //@ 正确
```

因此需要一个行为类似单个 bit 并可以被引用的对象，也就是 std::vector::reference，它可以隐式转换为 bool。

```
bool x = f()[0];
```

而对于 `auto` 推断则不会进行隐式转换：

```
auto x = f()[0]; //@ std::vector<bool>::reference x = f()[0];
//@ x不一定指向std::vector<bool>的第0个bit，这取决于std::vector<bool>::reference的实现
//@ 一种实现是含有一个指向一个machine word的指针，word持有被引用的bit和这个bit相对word的offset
//@ 于是x持有一个由opeartor[]返回的临时的machine word的指针和bit的offset
//@ 这条语句结束后临时对象被析构，于是x含有一个空悬指针，导致后续的未定义行为
if(x) ... // 相当于int* p; if(p) ...
```

std::vector::reference 是一个代理类（proxy class，模拟或扩展其他类型的类）的例子，比如 std::shared_ptr 和 std::unique_ptr 是很明显的代理类。还有一些为了提高数值计算效率而使用表达式模板技术开发的类，比如给定一个 Matrix 类和它的对象：

```
Matrix sum = m1 + m2 + m3 + m4;
```

Matrix 对象的 operator+ 返回的是结果的代理而非结果本身，这样可以使得表达式的计算更为高效。

```
auto x = m1 + m2; //@ x可能是Sum<Matrix, Matrix>而不是Matrix对象
```

auto 推断出代理类的问题实际很容易解决，事先做一次到预期类型的强制转换即可：

```
auto x = static_cast<bool>(f()[0]);
```









