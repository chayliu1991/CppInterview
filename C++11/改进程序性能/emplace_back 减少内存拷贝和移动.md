emplace_back 能就地通过函数构造对象，不需要拷贝或者移动内存，相比于 push_back  能更好地避免内存的拷贝与移动，使容器插入元素的性能得到进一步提升。在大多数情况下应该优先使用 emplace_back 来代替 push_back。标准库中类似的方法有：emplace，emplace_hint，emplace_front，emplace_after，emplace_back。

emplace_back  通过构造函数的参数就可以构造对象，因此，要求对象有相应的构造函数，如果没有会编译报错。

emplace_back 和 push_back  对比测试：

```
struct Complicated
{
	int year_;
	double country_;
	std::string name_;

	Complicated(int a, double b, std::string s) :year_(a), country_(b), name_(s)
	{
		std::cout << "is constructed" << std::endl;
	}

	Complicated(const Complicated& other) :year_(other.year_), country_(other.country_), name_(other.name_)
	{
		std::cout << "is moved" << std::endl;
	}
};

int main()
{
	std::map<int, Complicated> m;
	int i = 4;
	double d = 5.0;
	std::string s = "C++";
	std::cout << "---insert---" << std::endl;
	m.insert(std::make_pair(1, Complicated(i,d,s)));

	std::cout << "---emplace---" << std::endl;
	m.emplace(2, Complicated(i, d, s));

	std::vector<Complicated> v;
	std::cout << "---emplace_back---" << std::endl;
	v.emplace_back(i, d, s);
	std::cout << "---push_back---" << std::endl;
	v.push_back(Complicated(i, d, s));

	return 0;
}
```

结果：

```
---insert---
is constructed
is moved
is moved
---emplace---
is constructed
is moved
---emplace_back---
is constructed
---push_back---
is constructed
is moved
is moved
```

