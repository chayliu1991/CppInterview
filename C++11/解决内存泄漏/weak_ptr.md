- 弱引用指针 std::weak_ptr 用来监视 std::shared_ptr 不会使引用计数增加，也部管理  std::shared_ptr 内部的指针，主要是监视 std::shared_ptr 的生命周期。

- std::weak_ptr 没有重载 * 和 ->，因为它不共享指针，不能操作资源。
- std::weak_ptr 可以用来返回 this 指针和解决循环引用的问题。

# std::weak_ptr  基本用法

## use_count 获取当前观测资源的引用计数

```
std::shared_ptr<int> sp(new int(10));
std::weak_ptr<int> wp(sp);
std::cout << wp.use_count() << std::endl;  //@ 1
std::shared_ptr<int> sp2 = sp;
std::cout << wp.use_count() << std::endl;  //@ 2
```

## expired 判断所观测的资源是否释放

```
std::shared_ptr<int> sp(new int(10));
std::weak_ptr<int> wp(sp);
if (wp.expired())
	std::cout << "std::weak_ptr invalid in:" << __LINE__ << std::endl;

sp.reset();
if (wp.expired())
	std::cout << "std::weak_ptr invalid in:"<< __LINE__ << std::endl; //@ expired
```

## lock 获取监视的 std::shared_ptr

- 返回 std::shared_ptr
- std::shared_ptr 的引用计数加1

```
std::weak_ptr<int> gw;

void func()
{
	if (gw.expired())
	{
		std::cout << "gw is expired" << std::endl;
	}
	else
	{
		std::cout << "use count before lock:" << gw.use_count() << std::endl;
		auto spt = gw.lock();  //@ 返回一个 std::shared_ptr,引用计数加1
		std::cout << "dereference:" << *spt << std::endl;
		std::cout << "use count after lock:" << gw.use_count() << std::endl;
	}
}

int main()
{
	{
		auto sp = std::make_shared<int>(42);
		gw = sp;
		func();
	}

	func();

	return 0;
}
```

# std::weak_ptr 返回 this 指针

不能直接将 this 指针返回给 std::shared_ptr，需要通过派生 std::enable_shared_from_this 类，并通过方法 std::enable_from_this 来返回 智能指针。本质上：

- std::enable_shared_from_this 内部有一个  std::weak_ptr，这个 std::weak_ptr 用来观测 this 指针。
- 调用 shared_from_this  实际上内部调用了 std::weak_ptr 的 lock 方法返回一个 std::shared_ptr。

```
struct A : public std::enable_shared_from_this<A>
{
	std::shared_ptr<A> GetSelf()
	{
		return shared_from_this();
	}

	~A()
	{
		std::cout << "A is deleted" << std::endl;
	}
};
std::shared_ptr<A> sp1(new A);
std::shared_ptr<A> sp2 = sp1->GetSelf();
```

# std::weak_ptr 解决循环引用

std::shared_ptr 的循环引用将导致内存泄漏：

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
	//@ 资源不会被释放
	return 0;
}
```

由于循环引用导致程序结束时 aP 和 bP 的引用计数并没有减少到 0 导致内存泄漏。

解决的办法是将 A 中的 bPtr 或者 B 中的 aPtr 任意一个修改为 std::weak_ptr ：

```
struct A;
struct B;

struct A {
	std::weak_ptr<B> bPtr;
	~A() { std::cout << "A is deleted!" << std::endl; }
};

struct B {
	std::shared_ptr<A> aPtr;
	//@ std::weak_ptr<A> aPtr;
	~B() { std::cout << "B is deleted!" << std::endl; }
};
```





