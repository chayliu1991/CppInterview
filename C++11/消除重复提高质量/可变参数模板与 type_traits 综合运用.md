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

