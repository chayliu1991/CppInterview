# std::unique_ptr  独占性

std::unique_ptr 是一个独占的智能指针，它不允许其它的智能指针共享其内部的指针，不允许通过赋值将一个 std::unique_ptr 赋值给另一个 std::unique_ptr。

```
std::unique_ptr<T> myPtr(new T);
std::unique_ptr<T> otherPtr = myPtr; //@ 错误，不能复制
```

可以通过函数返回给其它的 std::unique_ptr，还可以通过 std::move 来转移到其它的 std::unique_ptr，这样它本身就不再拥有原来指针的所有权。

```
std::unique_ptr<T> myPtr(new T);
std::unique_ptr<T> otherPtr = std::move(myPtr); //@ Ok，可以移动
```

# make_unique

实现 make_unique：

```
//@ 支持普通指针
template <typename T, typename...Args>
inline typename std::enable_if<!std::is_array<T>::value, std::unique_ptr<T>>::type
make_unique(Args&&...args)
{
	return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

//@ 支持动态数组
template <typename T>
inline typename std::enable_if<std::is_array<T>::value && std::extent<T>::value == 0, std::unique_ptr<T>>::type
make_unique(size_t size)
{
	typedef typename std::remove_extent<T>::type U;
	return std::unique_ptr<T>(new U[size]());
}

//@ 过滤掉定长数组
template <typename T, typename...Args>
typename std::enable_if<std::extent<T>::value != 0, void>::type make_unique(Args&&...) = delete;


std::unique_ptr<int> p = make_unique<int>(10); //@ OK
std::unique_ptr<int[]> pArray1 = make_unique<int[]>(10); //@ OK
std::unique_ptr<int[]> pArray2 = make_unique<int[10]>; //@ 错误，不能创建定长数组的 std::unique_ptr
```

# std::unique_ptr 与  std::shared_ptr

## std::unique_ptr  具有独占性

## std::unique_ptr  可以指向一个数组

```
std::unique_ptr<int[]> ptr1(new int[10]); //OK;
std::shared_ptr<int[]> ptr2(new int[10]); //@ 错误，不合法
```

## std::unique_ptr  指定删除器时需要确定删除器的类型

```
std::shared_ptr<int> ptr(new int(1), [](int *p) { delete p; }); //@ OK
//std::unique_ptr<int> ptr2(new int(1), [](int *p) { delete p; }); //@ 错误

//@ lambda 没有捕捉变量时是正确的，因为没有捕获变量的 lambda 可以转换成函数指针，如果捕捉了变量则不可以
std::unique_ptr<int,void(*)(int*)> ptr3(new int(1), [](int *p) { delete p; }); 
```

如果希望 std::unique_ptr 的删除器支持 lambda 则应该写成：

```
std::unique_ptr<int, std::function<void(int*)>> ptr3(new int(1), [&](int *p) { delete p; });
```

也可以自定义 std::unique_ptr 的删除器：

```
struct MyDeleter
{
	void operator()(int*p)
	{
		std::cout << "delete" << std::endl;
		delete p;
	}
};
std::unique_ptr<int, MyDeleter> ptr3(new int(1));
```

## 小结

希望只有一个智能指针管理资源或者管理数组时使用 std::unique_ptr，其它情况使用 std::shared_ptr。



