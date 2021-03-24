# for 循环的新用法

```
std::vector<int> vec{ 1,2,3 };

for (auto v : vec)  //@ 对 v 的修改不会同步到 vec  
    //@ do something

for(auto &v : vec) //@ 对 v 的修改会同步到 vec 
	//@ do something

for(const auto& v : vec) //@ 对 v 不执行拷贝，并且不支持对 v 修改
	//@ do something
```

#  范围 for 的使用细节

## ： 后面的表达式只执行一次

```
std::vector<int> get_vec()
{
	std::cout << "get_vec()" << std::endl;
	return { 1,2,3 };
}
for (auto const v : get_vec()) //@ get_vec 函数只会执行一次，不会在每次遍历时反复执行
	std::cout << v << " ";
```

## 防止在范围 for 中迭代器失效

	std::vector<int> vec{ 1,2,3 };
	
	for (auto const v : vec)
	{
		std::cout << v << " ";
		vec.push_back(0);  //@ 迭代器失效
	}
# 自定义类型支持范围 for

基于范围 for 循环将以下面方式找到容器的 begin 和 end：

- 如容器是一个普通的数组对象，那么 begin 将是数组的首地址，end 将是数组首地址加上容器长度。
- 如果容器是一个类对象，那么范围 for 将试图通过查找类的 begin() 和 end() 函数来定位 begin 和 end。
- 否则范围 for 将使用全局的 begin() 和 end() 函数来定位 begin 和 end。

自定义类型如果想使用范围 for ，需要分别实现 begin() 和 end() 函数。

```
template <typename T>
class Iterator
{
public:
	using value_type = T;
	using size_type = size_t;

private:
	size_type cursor_;	//@ 迭代次数
	const value_type step_; //@ 步长
	value_type value_; //@ 初始值

public:
	Iterator(size_type cur_start, value_type begin_val, value_type step_val) :cursor_(cur_start), step_(step_val), value_(begin_val)
	{
		value_ += (step_ * cursor_);
	}

	value_type operator*()const { return value_; }

	bool operator!=(const Iterator& rhs) const { return cursor_ != rhs.cursor_; }

	Iterator& operator++(void)  //@ 前置 ++ 运算符
	{
		value_ += step_;
		++cursor_;
		return *this;
	}
};

template <typename T>
class Impl
{
public:
	using value_type = T;
	using reference = const value_type&;
	using const_reference = const value_type&;
	using iterator = const Iterator<value_type>;
	using const_iterator = const Iterator<value_type>;
	using size_type = typename iterator::size_type;

private:
	const value_type  begin_;
	const value_type  end_;
	const value_type  step_;	
	const size_type   max_count_; //@ 最大迭代次数

	size_type get_adjust_count() const
	{
		if (step_ > 0 && begin_ >= end_)
			throw std::logic_error("end value must be greater then begin value");
		else if (step_ < 0 && begin_ <= end_)
			throw std::logic_error("end value must be less then begin value");

		size_type x = static_cast<size_type>((end_ - begin_) / step_);
		if (begin_ + (step_ * x) != end_)
			++x;
		return x;
	}

public:
	Impl(value_type begin_val, value_type end_val, value_type step_val) :begin_(begin_val), end_(end_val), step_(step_val), max_count_(get_adjust_count())
	{
	}

	size_type size(void) const { return max_count_; }
	const_iterator begin(void) const { return{ 0,begin_,step_ }; } //@ begin 实现
	const_iterator end(void) const { return{ max_count_,begin_,step_ }; } //@ end 实现
};

template <typename T>
Impl<T> range(T end)
{
	return{ {},end,1 };
}

template <typename T>
Impl<T> range(T begin,T end)
{
	return{ begin,end,1 };
}

template <typename T,typename U>
auto range(T begin, T end, U step)->Impl<decltype(begin + step)> //@ 类型推导示例化模板
{
	using r_t = Impl<decltype(begin + step)>;
	return r_t(begin,end,step);
}


void test_range(void)
{
	std::cout << "range(10):";
	for (auto i : range(10))
		std::cout << " " << i;
	std::cout << std::endl;

	std::cout << "range(2,8):";
	for (auto i : range(2,8))
		std::cout << " " << i;
	std::cout << std::endl;


	std::cout << "range(-2,-10,-2):";
	for (auto i : range(-2, -10, -2))
		std::cout << " " << i;
	std::cout << std::endl;

	std::cout << "range(35,27,-2):";
	for (auto i : range(35, 27, -2))
		std::cout << " " << i;
	std::cout << std::endl;

	std::cout << "range('a','z'):";
	for (auto i : range('a', 'z'))
		std::cout << " " << i;
	std::cout << std::endl;


	std::cout << "range(30.5,35.5):";
	for (auto i : range(30.5, 35.5))
		std::cout << " " << i;
	std::cout << std::endl;
}

int main()
{
	test_range();

	return 0;
}
```

