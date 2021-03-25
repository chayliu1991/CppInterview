# 可调用对象

- 函数指针
- 具有 operator() 成员函数的类对象(仿函数)
- 可以转换成函数指针的类对象
- 类的成员(函数)指针

```
void func(void)
{
	std::cout << "func" << std::endl;
}

struct Foo
{
	void operator()(void)
	{
		std::cout << "functor" << std::endl;
	}
};

struct Bar
{
	using fr_t = void(*)(void);

	static void func(void)  //@ 必须是静态函数，没有隐式的 this 参数
	{
		std::cout << "Bar::func" << std::endl;
	}

	operator fr_t(void)
	{
		return func;
	}
};

struct A
{
	int i_;
	void mem_func(void)
	{
		std::cout << "mem_func" << std::endl;
	}
};

int main()
{
	void(*func_ptr)(void) = &func;  //@ 函数指针,也可以直接写成 void(*func_ptr)(void) = func;
	func_ptr();

	Foo foo;
	foo();  //@ 仿函数

	Bar bar;
	bar();  //@ 可转换为函数指针的类对象

	void (A::*mem_func_ptr)(void) = &A::mem_func;  //@ 类成员函数指针，这里必须使用 & 
	int A::*mem_obj_ptr = &A::i_;	 //@ 类成员指针
	A a;
	(a.*mem_func_ptr)();
	a.*mem_obj_ptr = 123;

	return 0;
}
```

# 可调用对象包装器 —— std::function

std::function 是可调用对象的包装器，是一个类模板，可以容纳除了类成员(函数)指针以外所有的可调用对象。

```
void func(void)
{
	std::cout << __FUNCTION__ << std::endl;
}

class Foo
{
public:
	static int foo_func(int a)
	{
		std::cout << __FUNCTION__ << "("<<a<<")"<<std::endl;
		return a;
	}
};

class Bar
{
public:
	int operator()(int a)
	{
		std::cout << __FUNCTION__ << "(" << a << ")" << std::endl;
		return a;
	}
};

int main()
{
	std::function<void(void)> fr1 = func;  //@ 绑定一个普通函数
	fr1();
	
	std::function<int(int)> fr2 = Foo::foo_func; //@ 绑定类得静态成员函数
	fr2(2);

	Bar bar;
	std::function<int(int)> fr3 = bar; //@ 绑定一个仿函数
	fr3(1);

	return 0;
}
```

## std::function 做回调

```
class A
{
	std::function<void(void)> callback_;

public:
	A(const std::function<void(void)>& f) :callback_(f) {}

	void notify(void)
	{
		if(callback_)
			callback_(); //@ 回调到上层
	}
};

class Foo
{
public:
	void operator()(void)
	{
		std::cout << __FUNCTION__ << std::endl;
	}
};

int main()
{
	Foo foo;
	A a(foo);
	a.notify();
	return 0;
}
```

## std::function 做函数指针

```
void call_when_even(int x, const std::function<void(int)>& f)
{
	if (!(x & 1) && f) //@ x % 2 == 0
	{
		f(x);
	}
}

void output(int x)
{
	std::cout << x << std::endl;
}

int main()
{
	for (int i = 0; i < 10; ++i)
	{
		call_when_even(i, output);
	}
	return 0;
}
```

# std::bind 绑定器

std::bind 作用：

- 将可调用对象与其参数绑定成一个仿函数。
- 只绑定部分参数，改变函数调用时需要传参的个数和顺序。

## 绑定可调用对象和参数返回一个仿函数

```
void call_when_even(int x, const std::function<void(int)>& f)
{
	if (!(x & 1) && f) //@ x % 2 == 0
	{
		f(x);
	}
}

void output(int x)
{
	std::cout << x << " ";
}

void output_add_2(int x)
{
	std::cout << x + 2 << " ";
}

int main()
{
	{
		auto fr = std::bind(output, std::placeholders::_1);
		for (int i = 0; i < 10; ++i)
		{
			call_when_even(i,fr);
		}
		std::cout << std::endl;
	}

	{
		auto fr = std::bind(output_add_2, std::placeholders::_1);
		for (int i = 0; i < 10; ++i)
		{
			call_when_even(i,fr);
		}
		std::cout << std::endl;
	}

	return 0;
}
```

## 改变传参的个数和顺序

```
void output(int x, int y)
{
	std::cout << x << " " << y << std::endl;
}

int main()
{
	std::bind(output, 1, 2)(); //@ 1 2
	std::bind(output, std::placeholders::_1, 2)(1);  //@  1 2
	std::bind(output, 2, std::placeholders::_1)(1);  //@  2 1

	//std::bind(output, 2, std::placeholders::_2)(1); //@ 错误，调用时没有第二个参数
	std::bind(output, 2, std::placeholders::_2)(1,2); //@ 2 2

	std::bind(output,std::placeholders::_2, std::placeholders::_1)(1, 2); //@ 2 1

	return 0;
}
```

## std::bind 和 std::function 配合使用

绑定类的成员(函数)指针，从而实现可调用对象的大一统表示方法：

```
class A
{
public:
	int i_;
	void output(int x, int y)
	{
		std::cout << x << " " << y << std::endl;
	}
};

int main()
{
	A a;
	std::function<void(int, int)> fr = std::bind(&A::output,&a,std::placeholders::_1,std::placeholders::_2);
	fr(1,2);

	std::function<int&(void)> fr_i = std::bind(&A::i_,&a);
	fr_i() = 123;
	std::cout << a.i_ << std::endl;

	return 0;
}
```

## 增强 std::bind1st  和 std::bind2nd

std::bind1st 和 std::bind2nd 是之前标准库中用于将二元算子转换成一元算子的方法：

```
std::vector<int> vec{ 1,2,3,4,5,6,7,8,9,10 };
int count1 = std::count_if(vec.begin(), vec.end(), std::bind1st(std::less<int>(), 5)); //@ 第一个参数固定为5，查找大于 5 的元素个数
int count2 = std::count_if(vec.begin(), vec.end(), std::bind2nd(std::less<int>(), 5)); //@ 第二个参数固定为5，查找小于 5 的元素个数
```

直接使用 std::bind ：

```
int count3 = std::count_if(vec.begin(), vec.end(), std::bind(std::less<int>(), 5, std::placeholders::_1)); //@ 第一个参数固定为5，查找大于 5 的元素个数
int count4 = std::count_if(vec.begin(), vec.end(), std::bind(std::less<int>(), std::placeholders::_1,5)); //@ 第二个参数固定为5，查找小于 5 的元素个数
```

## 组合使用 std::bind

复合多个函数(闭包)

```
using std::placeholders::_1;

std::vector<int> vec{ 1,2,3,4,5,6,7,8,9,10 };
auto f = std::bind(std::logical_and<bool>(),	 //@ 逻辑与
	std::bind(std::greater<int>(), _1, 5),  //@ 大于5
	std::bind(std::less_equal<int>(),_1, 10)); //@ 小于等于10
int count = std::count_if(vec.begin(),vec.end(),f);
```













