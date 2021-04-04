MessageBus：

```
class MessageBus : NonCopyable
{
public:
	//@ 注册消息
	template<typename F>
	void Attach(F&& f, const std::string& strTopic = "")
	{
		auto func = to_function(std::forward<F>(f));
		Add(strTopic, std::move(func));
	}

	//@ 发送消息
	template<typename R>
	void SendReq(const std::string& strTopic = "")
	{
		using function_type = std::function<R()>;
		std::string strMsgType = strTopic + typeid(function_type).name();
		auto range = m_map.equal_range(strMsgType);
		for (Iterater it = range.first; it != range.second; ++it)
		{
			auto f = it->second.AnyCast < function_type >();
			f();
		}
	}
	template<typename R, typename... Args>
	void SendReq(Args&&... args, const std::string& strTopic = "")
	{
		using function_type = std::function<R(Args...)>;
		std::string strMsgType = strTopic + typeid(function_type).name();
		auto range = m_map.equal_range(strMsgType);
		for (Iterater it = range.first; it != range.second; ++it)
		{
			auto f = it->second.AnyCast < function_type >();
			f(std::forward<Args>(args)...);
		}
	}

	//@ 移除某个主题, 需要主题和消息类型
	template<typename R, typename... Args>
	void Remove(const std::string& strTopic = "")
	{
		using function_type = std::function<R(Args...)>; //@ typename function_traits<void(CArgs)>::stl_function_type;

		std::string strMsgType = strTopic + typeid(function_type).name();
		int count = m_map.count(strMsgType);
		auto range = m_map.equal_range(strMsgType);
		m_map.erase(range.first, range.second);
	}

private:
	template<typename F>
	void Add(const std::string& strTopic, F&& f)
	{
		std::string strMsgType = strTopic + typeid(F).name();
		m_map.emplace(std::move(strMsgType), std::forward<F>(f));
	}

private:
	std::multimap<std::string, Any> m_map;
	typedef std::multimap<std::string, Any>::iterator Iterater;
};
```

  测试：

```
void testMsgBus()
{
	MessageBus bus;

    //@ 注册消息
	bus.Attach([](int a) {std::cout << "no reference: " << a << std::endl; });
	bus.Attach([](int& a) {std::cout << "lvalue reference: " << a << std::endl; });
	bus.Attach([](int&& a) {std::cout << "rvalue reference: " << a << std::endl; });
	bus.Attach([](const int& a) {std::cout << "const lvalue reference: " << a << std::endl; });
	bus.Attach([](int a) {std::cout << "no reference has return value and key: " << a << std::endl; 
		return a; },"a");

	//@ 发送消息
	int i = 2;
	bus.SendReq<void, int>(2);
	bus.SendReq<int, int>(2,"a");
	bus.SendReq<void, int&>(i);
	bus.SendReq<void, const int&>(2);
	bus.SendReq<void, int&&>(2);

	//@ 移除消息
	bus.Remove<void, int>();
	bus.Remove<int, int>("a");
	bus.Remove<void, int&>();
	bus.Remove<void, const int&>();
	bus.Remove<void, int&&>();

	//@ 发送消息
	bus.SendReq<void, int>(2);
	bus.SendReq<int, int>(2, "a");
	bus.SendReq<void, int&>(i);
	bus.SendReq<void, const int&>(2);
	bus.SendReq<void, int&&>(2);
}
```

