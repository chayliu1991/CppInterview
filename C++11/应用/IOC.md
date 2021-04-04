类型擦除的常用方法：

- 通过多态擦除类型。
- 通过模板来擦除类型。
- 通过某种类型容器来擦除类型。
- 通过某种通用类型来擦除类型。
- 通过闭包来擦除类型。

```
class IocContainer : NonCopyable
{
public:
	IocContainer(void) {}
	~IocContainer(void) {}

	template<class T, typename Depend, typename... Args>
	void RegisterType(const std::string& strKey)
	{
		std::function<T* (Args...)> function = [](Args... args) { return new T(new Depend(args...)); };	//@ 通过闭包擦除了参数类型
		RegisterType(strKey, function);
	}

	template<class T, typename... Args>
	T* Resolve(const std::string& strKey, Args... args)
	{
		if (creatorMap_.find(strKey) == creatorMap_.end())
			return nullptr;

		Any resolver = creatorMap_[strKey];
		std::function<T* (Args...)> function = resolver.AnyCast<std::function<T* (Args...)>>();

		return function(args...);
	}

	template<class T, typename... Args>
	std::shared_ptr<T> ResolveShared(const std::string& strKey, Args... args)
	{
		T* t = Resolve<T>(strKey, args...);

		return std::shared_ptr<T>(t);
	}

private:
	void RegisterType(const std::string& strKey, Any constructor)
	{
		if (creatorMap_.find(strKey) != creatorMap_.end())
			throw std::invalid_argument("this key has already exist!");

		//通过Any擦除了不同类型的构造器
		creatorMap_.emplace(strKey, constructor);
	}

private:
	std::unordered_map<std::string, Any> creatorMap_;
};
```

测试：

```
struct Base
{
	virtual void Func() {}
	virtual ~Base() {}
};

struct DerivedB : public Base
{
	DerivedB(int a, double b) :a_(a), b_(b)
	{
	}
	void Func()override
	{
		std::cout << a_ + b_ << std::endl;
	}
private:
	int a_;
	double b_;
};

struct DerivedC : public Base
{
};

struct A
{
	A(Base* ptr) : ptr_(ptr)
	{
	}

	void Func()
	{
		ptr_->Func();
	}

	~A()
	{
		if (ptr_ != nullptr)
		{
			delete ptr_;
			ptr_ = nullptr;
		}
	}

private:
	Base* ptr_;
};

void testIoc()
{
	IocContainer ioc;
	ioc.RegisterType<A, DerivedC>("C");      //@ 配置依赖关系
	auto c = ioc.ResolveShared<A>("C");

	ioc.RegisterType<A, DerivedB, int, double>("C");   //@ 注册时要注意DerivedB的参数int和double
	auto b = ioc.ResolveShared<A>("C", 1, 2.0); //@ 还要传入参数
	b->Func();
}
```

