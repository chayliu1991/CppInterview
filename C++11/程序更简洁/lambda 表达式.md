# labmbda 表达式

lambda 表达式来源于函数式编程的概念，其具有以下优点：

- 声明式编程风格：就地匿名定义目标函数或者函数对象，不需要额外写一个命名函数或者函数对象。
- 简洁：不需要额外写函数或者函数对象，避免了代码膨胀和功能分散。
- 更灵活：在需要的时间地点实现函数闭包。

# lambda 表达式基本概念与用法

lambda 表达式定义了一个匿名函数，并且可以捕获一定范围内的变量，其语法形式：

```
[capture](params) opt->ret{ body; };
```

- capture：捕获列表。
- params：参数列表。
- opt：函数选项。
- ret：返回值类型。
- body：函数体。

## 捕获列表

- [] : 不捕获任何变量。
- [&] : 捕获外部作用域中的所有变量，并作为引用在函数体中使用(按引用捕获)。
- [=] : 捕获外部作用域中的所有变量，并作为副本在函数体中使用(按值捕获)。
- [=，&foo] : 按引用捕获 foo，并按值捕获所有其它变量。
- [bar] : 按值捕获 bar 变量，不捕获其它变量。
- [&bar] : 按引用捕获 bar 变量，不捕获其它变量。
- [this] : 捕获当前类的 this 指针，让 lambda 表达式拥有当前成员函数同样的访问全选，使用 [&] 和 [=] 捕获，默认添加此项。

## 返回值

- 返回值采用后置的语法实现。
- 允许省略返回值类型定义，编译器会根据语句自动推导。
  - 采用自动推导时，如果有多处返回，类型需要保持一致。
  - 不能推导初始化列表。

```
//@ 错误，lambda 无法推导返回类型
auto f = [](int x) 
{
    if (x & x)
   	 	return 0.0;
    else
    	return 1;
};

//@ 错误，无法根据初始化列表自动推导
auto f = []()
{
	return { 1,2 };
};
```

## 参数列表

- 默认情况下，参数列表为空是，可以省略参数列表。

```
auto f = []	{ return 1;	}; //@  正确
```

## 按值捕获和按引用捕获

- 按值捕获时，默认情况下 lambda 体中不允许修改变量的值，如果想要想要修改变量的值：
  - 按引用捕获，修改变量后外部变量同步修改。
  - 使用 mutable 修饰，修改变量后外部变量不会同步修改。如果使用 mutable 修饰，即使没有参数，也需要写明参数列表。

```
int a = 1;
auto f = [=]() {return ++a; };  //@ 错误，不允许修改
auto f = [=]() mutable {return ++a; };  //@ 可以修改 a，但是 lambda 表达式外部的 a 的值不变
auto f = [&]() mutable {return ++a; };  //@ 可以修改 a，但是 lambda 表达式外部的 a 的值同步该变
```

- 需要注意延迟调用的副作用

```
int a = 0;
auto f = [=] { return a; };
a += 1;
std::cout << f() << std::endl;  //@ 值是 0，如果想同步上下文，需要按引用捕获
```

## lambda 表达式的类型

lambda 表达式在 C++ 11 中称为闭包类型，可以将其理解为一个带 operator() 的类，即仿函数。因而可以使用 std::function 来存储 lambda 表达式，也可以使用 std::bind  来操作 lambda 表达式。

```
std::function<int(int)> f1 = [](int a) {return a; };
std::function<int(void)> f2 = std::bind([](int a) { return a; },123);
```

没有捕获任何变量的 lambda 表达式，可以转换成一个普通的函数指针：

```
using func_t = int(*)(int);
func_t f = [](int a) { return a; };
f(12);
```

lambda  表达式的 operator() 是 const 的，这也是为什么按值捕获无法修改变量的本质原因，使用 mutable 则取消了 operator() 的 const ，因而可以修改变量。

# 声明式编程

```
std::vector<int> vec{1,2,3,4,5,6,7,8,9};
int even_count{ 0 };
std::for_each(vec.begin(), vec.end(), [&even_count](int x) {if (!(x & 1)) ++even_count; }); //@ 不要提前定义仿函数
```

# 在需要的时间和地点实现闭包

```
std::vector<int> vec{1,2,3,4,5,6,7,8,9};
std::count_if(vec.begin(), vec.end(), [](int val) {return val > 5 && val <= 10; });  //@ 大于 5 小于等于10
```







