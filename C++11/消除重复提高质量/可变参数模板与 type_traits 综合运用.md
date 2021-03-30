# Optional  的实现

Optional<T>  内部可能存储了 T 类型，也可能没有存储 T 类型，只有当 Optional  被 T 初始化之后，Optional  才是有效的，否则无效。

```
#include <type_traits>

template <typename T>
class Optional
{
	using data_t = typename std::aligned_storage<sizeof(T), std::alignment_of<T>::value>::type;

public:
	Optional() {}
	Optional(const T& t)
	{
		Create(t);
	}

	Optional(const Optional& other)
	{
		if (other.Isinit())
			Assign(other);
	}

	~Optional()
	{
		Destroy(); //@ 删除缓冲区对象
	}

	//@ 根据参数创建
	template <typename ... Args>
	void Emplace(Args...args)
	{
		Destroy();
		Create(std::forward<Args>(args)...);
	}

	//@ 是否已经初始化
	bool  IsInit() const { return m_hasInit; }

	//@ 判断是否已经初始化
	explicit operator bool()const
	{
		return IsInit();
	}

	//@ 从 Optional 中获取对象
	T const& operator*() const
	{
		if (IsInit())
			return *((T*)(&m_data));
		throw std::logic_error("is not init");
	}

private:
	template <typename ...Args>
	void Create(Args&& ...args)
	{
		new (&m_data) T(std::forward<Args>(args)...); //@ placement new
		m_hasInit = true;
	}

	//@ 销毁缓冲区对象
	void Destroy()
	{
		if (m_hasInit)
		{
			m_hasInit = false;
			((T*)(&m_data))->~T();
		}
	}

	void Assign(const Optional& other)
	{
		if (other.IsInit())
		{
			Copy(other.m_data);
			m_hasInit = true;
		}
		else
			Destroy();
	}

	void Copy(const data_t& val)
	{
		Destroy();
		new (&m_data) T(*((T*)(&val)));
	}

private:
	bool m_hasInit = false; //@ 是否已经被初始化
	data_t m_data;	//@ 内存对齐的缓冲区,如果按照1字节对齐，调用 placement new 时可能会引起效率问题或出错
};
```

测试：

```
struct MyStruct
{
	MyStruct(int a, int b) :m_a(a), m_b(b) {}
	int m_a;
	int m_b;
};

int main()
{
	Optional<std::string> s1("hello");
	if (s1)
		std::cout << "s1 init:" << *s1 << std::endl;

	Optional<std::string> s2 = s1;
	if (s1)
		std::cout << "s1 init:" << *s1 << std::endl;
	if (s2)
		std::cout << "s2 init:" << *s2 << std::endl;

	Optional<MyStruct> t;
	if (t)
		std::cout << "t init:" << (*t).m_a << "," << (*t).m_b << std::endl;
	t.Emplace(2,5);
	if (t)
		std::cout << "t init:" << (*t).m_a << "," << (*t).m_b << std::endl;

	return 0;
}
```

# 惰性求值 lazy 实现

惰性求值一般用于函数式编程语言中，使用延迟求值的时候，表达式不在它被绑定到变量之后就立即求值，而是在需要的时候再求值。

```
#include "Optional.hpp"

template <typename T>
struct Lazy
{
	Lazy() {}

	template<typename Func, typename ...Args>
	Lazy(Func& f, Args&&...args)
	{
		m_func = [&f, &args...]{ return f(args...); };
		//m_func = std::bind(f,std::forward<Args...>(args)...);  //@ ok
	}

	T Value()
	{
		if (!m_value.IsInit())
		{
			m_value = m_func();
		}
		return *m_value;
	}

	bool IsValueCreated()const
	{
		return m_value.IsInit();
	}

private:
	std::function<T()> m_func; //@ 用于存放传入的函数,返回 T 类型，无参数，参数可以通过 lambda 表达式绑定
	Optional<T> m_value;    //@ 用于存放求值结果，可以通过 Optional 特性判断是否已经初始化，节省变量
};

//@ 包装函数，用于自动推断返回类型
template<class Func, typename ...Args>
Lazy<typename std::result_of<Func(Args...)>::type> make_lazy(Func&& func, Args&& ...args)
{
	return Lazy<typename std::result_of<Func(Args...)>::type>(std::forward<Func>(func), std::forward<Args>(args)...);
}
```

测试：

```
struct BigObject
{
	BigObject()
	{
		std::cout << "lazy load big object" << std::endl;
	}
};

struct SmallObj
{
	SmallObj()
	{
		m_BigObj = make_lazy([] {return std::make_shared<BigObject>(); });
	}

	void Load()
	{
		m_BigObj.Value();
	}

	Lazy<std::shared_ptr<BigObject>> m_BigObj;
};

int Foo(int x)
{
	return x * 2;
}

void TestLazy()
{
	//@ 带参数的普通函数
	int y = 4;
	auto lazyer1 = make_lazy(Foo, y);
	std::cout << lazyer1.Value() << std::endl;

	//@ 不带参数的 lambda 
	Lazy<int> lazyer2 = make_lazy([] {return 12; });
	std::cout << lazyer2.Value() << std::endl;

	//@ 带参数的 fuction
	std::function <int(int)> f = [](int x) {return x + 2; };
	auto lazyer3 = make_lazy(f, 3);
	std::cout << lazyer3.Value() << std::endl;

	//@ 延迟加载大的对象
	SmallObj st;
	st.Load();
}

int main()
{
	TestLazy();
	return 0;
}
```

# dll 帮助类

```
#include "Windows.h"  
#include <string>
#include <map>
#include <iostream>
#include <functional>

class DllParser
{
public:
	DllParser() :m_hMod(nullptr)
	{
	}

	~DllParser()
	{
		UnLoad();
	}

	bool Load(const std::string& dllpath)
	{
		m_hMod = LoadLibraryA(dllpath.data());
		if (nullptr == m_hMod)
		{
			std::cout << "Load library failed"<<std::endl;
			return false;
		}
		return true;
	}

	bool UnLoad()
	{
		if (m_hMod == nullptr)
		{
			return true;
		}

		auto b = FreeLibrary(m_hMod);
		if (!b)
			return false;
		m_hMod = nullptr;
		return true;
	}
	
	template <typename T>
	std::function<T> GetFunction(const std::string& funcName)
	{
		auto it = m_map, find(funcName);
		if (it == m_map.end())
		{
			auto addr = GetProcAddress(m_hMod,funcName.c_str());
			if (!addr)
				return nullptr;
			m_map.insert(std::make_pair(funcName,addr));
			it = m_map.find(funcName);
		}
		return std::function<T>((T*)(it->second));
	}

	template <typename T,typename...Args>
	typename std::result_of<std::function<T>(Args...)>::type ExcecuteFunc(const std::string& fucName, Args...args)
	{
		auto f = GetFunction<T>(fucName);
		if (f == nullptr)
		{
			std::string str = "can not find this function " + fucName;
			throw std::exception(str.c_str());
		}
		return f(std::forward<Args>(args)...);
	}


private:
	HMODULE m_hMod;
	std::map<std::string, FARPROC> m_map;

};
```

# lambda 链式调用

```
#include <functional>
#include <iostream>
#include <type_traits>

template <typename T>
class Task;

template <typename R, typename ...Args>
class Task<R(Args...)>
{
public:
	Task(std::function<R(Args...)>&& f) :m_fn(std::move(f)) {}
	Task(std::function<R(Args...)>& f) :m_fn(f) {}

	R Run(Args&&...args)
	{
		return m_fn(std::forward<Args>(args)...);
	}

	template <typename F>
	auto Then(F&& f)->Task<typename std::result_of<F(R)>::type(Args...)>
	{
		using return_type = typename std::result_of<F(R)>::type;
		auto func = std::move(m_fn);
		return Task<return_type(Args...)>([func, &f](Args...args)
		{
			return f(func(std::forward<Args>(args)...));
		});
	}

private:
	std::function<R(Args...)> m_fn;
};
```

测试：

```
void testTask()
{
	Task<int(int)> task([](int i) { return i; });
	auto res = task.Then([](int i) { return i + 1; }).Then([](int i) {return i + 2; }).Then([](int i) {return i + 3; }).Run(1);
	std::cout << res << std::endl;
}

int main()
{
	testTask();
	return 0;
}
```

# Any 类实现

any 类是一个特殊的只能容纳一个元素的容器，它可以擦除类型，可以赋给它任何类型的值，在实际使用时再根据实际的类型进行转换。

```
#include <memory>
#include <typeindex>

struct Any
{
	Any(void) : m_tpIndex(std::type_index(typeid(void))) {}
	Any(Any& that) : m_ptr(that.Clone()), m_tpIndex(that.m_tpIndex) {}
	Any(Any&& that) : m_ptr(std::move(that.m_ptr)), m_tpIndex(std::move(that.m_tpIndex)) {}

	template<typename U, class = typename std::enable_if<!std::is_same<typename std::decay<U>::type, Any>::value, U>::type>
	Any(U&& value) : m_ptr(new Derived<typename std::decay<U>::type>(std::forward<U>(value))), m_tpIndex(std::type_index(typeid(typename std::decay<U>::type)))
	{
	}

	bool IsNull() const { return !bool(m_ptr); }

	template <class U> bool Is() const
	{
		return m_tpIndex == std::type_index(typeid(U));
	}

	template <class U>
	U& AnyCast()
	{
		if (!Is<U>())
		{
			//@ 不是原来的类型，抛出异常
			std::cout << "can not cast" << typeid(U).name() << " to " << m_tpIndex.name() << std::endl;
			throw std::bad_cast();
		}

		auto derived = dynamic_cast<Derived<U>*> (m_ptr.get());
		return derived->m_value;
	}

	Any& operator=(const Any& a)
	{
		if (m_ptr == a.m_ptr)
			return *this;
		m_ptr = a.Clone();
		m_tpIndex = a.m_tpIndex;
		return *this;
	}

private:
	struct Base;
	typedef std::unique_ptr<Base> BasePtr;
	struct Base
	{
		virtual ~Base() {}
		virtual BasePtr Clone() const = 0;
	};

	template <typename T>
	struct Derived : Base
	{
		template <typename U>
		Derived(U&& value) : m_value(std::forward<U>(value)) {}
		BasePtr Clone()const
		{
			return BasePtr(new Derived<T>(m_value));
		}

		T m_value;
	};

	BasePtr Clone() const
	{
		if (m_ptr != nullptr)
			return m_ptr->Clone();
		return nullptr;
	}

	BasePtr m_ptr;
	std::type_index m_tpIndex;
};
```

- 赋值给 any 时需要擦除值得类型，使用一种通用的方法保存所有的数据类型，这里通过继承去擦除类型：
  - 基类是不含模板参数的，派生类中才有模板参数，这个模板参数类型就是赋值类型。
  - 在赋值时将创建的派生类对象赋值给基类指针，基类的派生类携带了数据类型，基类只是原始数据的占位符，通过多态隐式转换擦除了原始数据类型。因此，任何数据都可以赋值给它，从而实现能存放所有类型数据的目标。
  - 取数据时，向下转换成派生类型来获取原始的数据，当转换失败时，抛出异常。
- 管理对象的生命周期使用 std::unique_ptr。

测试：

```
void testAny()
{
	std::vector<Any> vec;
	vec.push_back(1);
	vec.push_back(std::string("hello"));
	auto v1 = vec[0].AnyCast<int>();
	auto v2 = vec[1].AnyCast<std::string>();

	Any any2 = 100;
	auto res = any2.AnyCast<int>();

	Any any;
	auto r = any.IsNull();  //@ true
	std::string s1 = "hello";
	any = s1;
	any.AnyCast<int>(); //@ crash
}
```











