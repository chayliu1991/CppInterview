- 弱引用指针 std::weak_ptr 用来监视 std::shared_ptr 不会使引用计数增加，也部管理  std::shared_ptr 内部的指针，主要是监视 std::shared_ptr 的生命周期。

- std::weak_ptr 没有重载 * 和 ->，因为它不共享指针，不能操作资源。
- std::weak_ptr 可以用来返回 this 指针和解决循环引用的问题。

# 基本用法

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





