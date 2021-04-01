std::shared_ptr 使用引用计数，每个 std::shared_ptr 的拷贝都指向相同的内存，在最后一个 std::shared_ptr 析构时，内存才会被释放。

# 基本用法

## 初始化

- 构造函数。

- std::make_shared<T>，优先使用，更加高效。

- reset，原来的智能指针如果有值，引用计数会减1。

```
  std::shared_ptr<int> p1(new int(1));
  std::shared_ptr<int> p2;
  p2.reset(new int(2));
  std::shared_ptr<int> p3 = std::make_shared<int>(3);
```


 不能将原始指针直接赋值给智能指针：

```
std::shared_ptr<int> p4 = new int(4); //@  错误
```

## 获取原始指针

- get

```
std::shared_ptr<int> sp = std::make_shared<int>(0);
int* pOri = sp.get();
```

## 指定删除器

- 智能指针初始化时可以指定删除器，当引用计数为0时会自动调用删除器

```
void DeleteIntPtr(int* p)
{
	delete p;
}

std::shared_ptr<int> sp(new int(1), DeleteIntPtr);

//@ 使用 lambda 表达式
std::shared_ptr<int> sp(new int(1), [](int *p) {delete p; });
	
std::shared_ptr<int> sp = std::make_shared<int, DeleteIntPtr>(0); //@ 错误，没有这种形式
```

管理动态数组：

```
std::shared_ptr<int> sp(new int[10], [](int*p) {delete[] p; });

//@ std::default_delete 内部通过调用 delete 实现功能
std::shared_ptr<int> shared_good(new int[10], std::default_delete<int[]>());
```

封装 make_shared_array 函数：

```
template <typename T>
std::shared_ptr<T> make_shared_array(size_t  size)
{
	return std::shared_ptr<T>(new T[size], std::default_delete<T[]>());
}

std::shared_ptr<int> p = make_shared_array<int>(10);
std::shared_ptr<char> p = make_shared_array<char>(100);
```

# 使用 std::shared_ptr 需要注意的问题

### 不要使用一个原始指针初始化多个 std::shared_ptr

 ```
int * ptr = new int;
std::shared_ptr<int> sp1(ptr);
std::shared_ptr<int> sp2(ptr); //@ 逻辑错误
 ```

### 不要在函数实参中创建 std::shared_ptr

```
func(std::shared_ptr<int>(new int),g());
```

因为 C++ 函数的参数计算顺序在不同的编译器有不同的调用约定，一般时从右向左，也可能从左到右。可能的步骤：

- 先调用 new int
- 再调用 g() 
- 再创建  std::shared_ptr

如果 g() 函数调用时发生异常，则  std::shared_ptr 还没有创建，new int 的内存就会泄漏。正确的写法：

```
std::shared_ptr<int> sp(new int)
f(sp,g());
```

### 通过 shared_from_this() 返回 this 指针

不要将 this 指针作为 std::shared_ptr 返回，因为 this 本质上是一个裸指针，因此可能导致重复析构。

```
struct A
{
	std::shared_ptr<A> GetSelf()
	{
		return std::shared_ptr<A>(this); //@ 不要这样做
	}
};

int main(void)
{
	//@ 使用同一个指针 this 构造了两个智能指针，而它们之间没有任何联系
	//@ 离开作用域之后 sp1 sp2 都将析构导致重复析构的问题
	std::shared_ptr<A> sp1(new A);
	std::shared_ptr<A> sp2 = sp1->GetSelf();

	return 0;
}
```

正确返回 this 的 std::shared_ptr  的方法是：让目标类通过派生 std::enable_shared_from_this<T> 类，然后使用基类的成员函数 shared_from_this 来返回 this 的  std::shared_ptr：

```
struct A : std::enable_shared_from_this<A>
{
	std::shared_ptr<A> GetSelf()
	{
		return shared_from_this();
	}
};

int main(void)
{
	//@ OK
	std::shared_ptr<A> sp1(new A);
	std::shared_ptr<A> sp2 = sp1->GetSelf();
	return 0;
}
```

### 避免循环引用

智能指针的循环引用将导致内存泄漏：

```
struct A;
struct B;

struct A {
	std::shared_ptr<B> bPtr;
	~A() { std::cout << "A is deleted!" << std::endl; }
};

struct B {
	std::shared_ptr<A> aPtr;
	~B() { std::cout << "B is deleted!" << std::endl; }
};

void testPtr()
{
	std::shared_ptr<A> aP(new A);
	std::shared_ptr<B> bP(new B);
	aP->bPtr = bP;
	bP->aPtr = aP;
}

int main(void)
{
	testPtr();
	return 0;
}
```

循环引用导致 aP，bP 的引用计数是2，在离开作用域之后， aP，bP 的引用计数都减少为1，并不会减少为0，导致两个指针都不会被析构，产生了内存泄漏。

解决办法是将 A 和 B 中任何一个成员变量改成 std::weak_ptr。





