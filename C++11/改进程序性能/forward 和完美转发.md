一个右值引用参数作为函数的形参，在函数内部再转发该参数的时候它已经变成一个左值，并不是原来的类型：

````
template <typename T>
void forwardValue(T& val)
{
	processValue(val);  //@ 右值参数会变成左值
}

template <typename T>
void forwardValue(const T& val)
{
	processValue(val);    //@ 右值参数会变成左值
}
````

完美转发是指在函数模板中，完全按照模板的参数的类型将参数传递给函数模板中调用的另一个函数，这利用 std::forward 实现。

```
template <typename T>
void printT(T& t)
{
	std::cout << "lvalue" << std::endl;
}

template <typename T>
void printT(T&& t)
{
	std::cout << "rvalue" << std::endl;
}

template<typename T>
void TestForward(T&& t)
{
	printT(t);
	printT(std::forward<T>(t));
	printT(std::move(t));
	std::cout << "--------------------------------" << std::endl;
}

void Test()
{
	TestForward(1);
	int x = 1;
	TestForward(x);
	TestForward(std::forward<int>(x));
}

int main()
{
	Test();
	return 0;
}
```

输出：

```
lvalue      //@ 1 是右值，但是调用 printT(t) 时它是左值，因为它是一个具名变量
rvalue
rvalue
--------------------------------
lvalue
lvalue
rvalue
--------------------------------
lvalue
rvalue
rvalue
--------------------------------
```

右值引用、完美转发再结合可变模板参数可以写一个万能的函数包装器：

```
template<class Function,class...Args>
inline auto FuncWrapper(Function&&f,Args&&...args)->decltype(f(std::forward<Args>(args)...))
{
	return f(std::forward<args>(args)...);
}
```

测试：

```
template<class Function,class...Args>
inline auto FuncWrapper(Function&&f,Args&&...args)->decltype(f(std::forward<Args>(args)...))
{
	return f(std::forward<Args>(args)...);
}

void test0()
{
	std::cout << "void" << std::endl;
} 

int test1()
{
	return 1;
}

int test2(int x)
{
	return x;
}

std::string test3(std::string s1,std::string s2)
{
	return s1 + s2;
}

void test()
{
	FuncWrapper(test0);  //@ 没有返回值
	auto res1 = FuncWrapper(test1);  //@ 返回1
	auto res2 = FuncWrapper(test2,2);		//@ 返回2
	auto res3 = FuncWrapper(test3,"aa","bb");   //@ 返回aabb
}

int main()
{
	test();
	return 0;
}
```











