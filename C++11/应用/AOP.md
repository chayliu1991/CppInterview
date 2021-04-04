AOP：

```
#define HAS_MEMBER(member)\
template<typename T, typename... Args>struct has_member_##member\
{\
private:\
	template<typename U> static auto Check(int) -> decltype(std::declval<U>().member(std::declval<Args>()...), std::true_type()); \
	template<typename U> static std::false_type Check(...);\
public:\
	enum{value = std::is_same<decltype(Check<T>(0)), std::true_type>::value};\
};\

HAS_MEMBER(Foo)
HAS_MEMBER(Before)
HAS_MEMBER(After)

template<typename Func, typename... Args>
struct Aspect : NonCopyable
{
	Aspect(Func&& f) : func_(std::forward<Func>(f))
	{
	}

	template<typename T>
	typename std::enable_if<has_member_Before<T, Args...>::value&& has_member_After<T, Args...>::value>::type Invoke(Args&&... args, T&& aspect)
	{
		aspect.Before(std::forward<Args>(args)...);	//@ 核心逻辑之前的切面逻辑
		func_(std::forward<Args>(args)...);		//@ 核心逻辑
		aspect.After(std::forward<Args>(args)...);	//@ 核心逻辑之后的切面逻辑
	}

	template<typename T>
	typename std::enable_if<has_member_Before<T, Args...>::value && !has_member_After<T, Args...>::value>::type Invoke(Args&&... args, T&& aspect)
	{
		aspect.Before(std::forward<Args>(args)...);	//@ 核心逻辑之前的切面逻辑
		func_(std::forward<Args>(args)...);	//@ 核心逻辑
	}

	template<typename T>
	typename std::enable_if<!has_member_Before<T, Args...>::value&& has_member_After<T, Args...>::value>::type Invoke(Args&&... args, T&& aspect)
	{
		func_(std::forward<Args>(args)...);	//@ 核心逻辑
		aspect.After(std::forward<Args>(args)...);	//@ 核心逻辑之后的切面逻辑
	}

	template<typename Head, typename... Tail>
	void Invoke(Args&&... args, Head&& headAspect, Tail&&... tailAspect)
	{
		headAspect.Before(std::forward<Args>(args)...);
		Invoke(std::forward<Args>(args)..., std::forward<Tail>(tailAspect)...);
		headAspect.After(std::forward<Args>(args)...);
	}

private:
	Func func_; //被织入的函数
};

template<typename T> using identity_t = T;

//@ AOP的辅助函数，简化调用
template<typename... AP, typename... Args, typename Func>
void Invoke(Func&& f, Args&&... args)
{
	Aspect<Func, Args...> asp(std::forward<Func>(f));
	asp.Invoke(std::forward<Args>(args)..., identity_t<AP>()...);
}
```

测试：

```
struct AA
{
	void Before(int i)
	{
		std::cout << "Before from AA : " << i << std::endl;
	}

	void After(int i)
	{
		std::cout << "After from AA : " << i << std::endl;
	}
};

struct BB
{
	void Before(int i)
	{
		std::cout << "Before from BB : " << i << std::endl;
	}

	void After(int i)
	{
		std::cout << "After from BB : " << i << std::endl;
	}
};

struct CC
{
	void Before()
	{
		std::cout << "Before from CC" << std::endl;
	}

	void After()
	{
		std::cout << "After from CC" << std::endl;
	}
};

struct DD
{
	void Before()
	{
		std::cout << "Before from DD" << std::endl;
	}

	void After()
	{
		std::cout << "After from DD" << std::endl;
	}
};

void GT()
{
	std::cout << "real GT function" << std::endl;
}

void HT(int a)
{
	std::cout << "real HT function : " << a << std::endl;
}

void testAop()
{
	//@ 织入普通函数
	std::function<void(int)> f = std::bind(&HT, std::placeholders::_1);
	Invoke<AA, BB>(std::function<void(int)>(std::bind(&HT, std::placeholders::_1)), 1);

	//@ 组合了两个切面 AA，BB
	Invoke<AA, BB>(f, 1001);

	//@ 织入普通函数
	Invoke<CC, DD>(&GT);
	Invoke<AA, BB>(&HT, 100);

	//@ 织入 lambda 表达式
	Invoke<AA, BB>([](int i) {}, 99);
	Invoke<CC, DD>([] {});
}
```

测试：

```
struct TimeElapsedAspect
{
	void Before(int i)
	{
		lastTime_ = t_.elapsed();
	}

	void After(int i)
	{
		std::cout<< "time elapsed: " << t_.elapsed() - lastTime_ << std::endl;
	}

private:
	double lastTime_;
	Timer t_;
};

struct LoggingAspect
{
	void Before(int i)
	{
		std::cout << "entering" << std::endl;
	}

	void After(int i)
	{
		std::cout << "leaving" << std::endl;
	}
};

void foo(int a)
{
	std::cout << "real HT Function: " << a << std::endl;
}

int main()
{
	Invoke<LoggingAspect, TimeElapsedAspect>(&foo, 999);
	std::cout << "--------------------" << std::endl;
	Invoke<TimeElapsedAspect, LoggingAspect>(&foo, 666);

	return 0;
}
```



