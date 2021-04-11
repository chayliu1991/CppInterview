# vector 和 string 优先于动态分配的数组

如果使用  new 来动态分配内存，使用者必须承担以下的责任：

- 确保之后调用 delete 将内存释放
- 确保使用的是正确的 delete 形式，对于单个对象要用 delete，对于数组对象需要用 delete[]
- 确保对于一个对象只 delete 一次

vector、string 自动管理其所包含元素的构造与析构，并有一系列的 STL 算法支持，同时 vector 也能够保证和老代码的兼容。

使用了引用计数的 string 可以避免不必要的内存分配和字符串拷贝(COW- copy on write)，但是在多线程环境里，对这个 string 进行线程同步的开销远大于 COW 的开销。此时，可以考虑使用 vector<char> 或动态数组。

# 使用 reserve 来避免不必要的重新分配

```
int test_item_14()
{
	std::vector<int> v;
	//@ 如果不使用reserve,下面的循环在进行过程中将导致多次重新分配
	//@ 加上reserve，则在循环过程中,将不会再发生重新分配
	v.reserve(1000); 
	for (int i = 1; i <= 1000; ++i) 
		v.push_back(i); 
	return 0;
}
```

对于 vector 和 string，增长过程是这样来实现的：

- 分配一块儿原内存大小数倍的新内存，对于 vector 和 string 而言，通常是两倍
- 将原来容器中的元素拷贝到新内存中
- 析构旧内存中的对象
- 释放旧内存

reserve 成员函数能使你把重新分配的次数减少到最低限度，从而避免了重新分配和指针/迭代器/引用失效带来的开销。避免重新分配的关键在于，尽早地使用 reserve，把容器的容量设为足够大的值，最好是在容器刚被构造出来之后就使用 reserve。

reserve 以及与 reserve 相关的几个函数：

- size() 容器中现有的元素的个数
- capacity() 容器在不重新分配内存的情况下可容纳元素的总个数
- resize(Container::size_type n)  将容器的 size 强制改变为 n 
- - n > size 将现有容器中的元素拷贝到新内存，并将空余部分用默认构造的新函数填满
  - n < size 将尾部的元素全部析构掉
- reserve(Container::size_type n) 将容器的 size 改变至少为 n
  - n > size 将现有容器中的元素拷贝到新内存，多余部分的内存仍然空置
  - n < size 对容器没有影响

通常有两种方式使用 reserve 避免不必要的内存分配：

- 预测大致所需的内存，并在构造容器之后就调用 reserve 预留内存
- 先用 reserve 分配足够大的内存，将所有元素都加入到容器之后再去除多余内存

#  注意 string 实现的多样性

- string 的值可能会被引用计数，也可能不会。很多实现在默认情况下会使用引用计数，但它们通常提供了关闭默认选择的方法，往往是通过预处理宏来做到这一点。
- string 对象大小的范围可以是一个 char* 指针大小的 1 倍到 7 倍。
- 创建一个新的字符串值可能需要零次、一次或两次动态分配内存。
- string 对象可能共享，也可能不共享其大小和容量信息。
- string 可能支持，也可能不支持针对单个对象的分配子。
- 不同的实现对字符内存的最小分配单位有不同的策略。

# 了解如何把 vector 和 string 数据传给旧的 API

将 vector 传递给接受数组指针的函数，要注意 vector 为空的情况。迭代器并不等价于指针，所以不要将迭代器传递给参数为指针的函数。

```
void foo(const int* ptr, size_t size);

vector<int> v;
...
foo(v.empty() ? NULL : &v[0], v.size());
```

将 string 传递给接受字符串指针的函数。该方法还适用于 s 为空或包含 `\0`  的情况：

```
void foo(const char* ptr);

string s;
...
foo(s.c_str());
```

使用初始化数组的方式初始化 vector：

```
//@ 向数组中填入数据
size_t fillArray(int* ptr, size_t size);

int maxSize = 10;
vector<int> v(maxSize);
v.resize(fillArray(&v[0],v.size()));
```

借助 vector 与数组内存布局的一致性，我们可以使用 vector 作为中介，将数组中的内容拷贝到其他 STL 容器之中或将其他 STL 容器中的内容拷贝到数组中：

```
//@ 向数组中填入数据
size_t fillArray(int* ptr, size_t size);

vector<int> v(maxSize);
v.resize(fillArray(&v[0],v.size()));
set<int> s(v.begin(),v.end());

void foo(const int* ptr, size_t size);

list<int> l();
...

vector<int> v(l.begin(),l.end());
foo(v.empty()? NULL : &v[0],v.size());
```

C++ 标准要求 vector 中的元素存储在连续的内存中，就像数组一样。string 中的数据不一定存储在连续的内存中，而且 string 的内部表示不一定是以空字符结尾的。

```
void doSomething(const int* pInts, size_t numInts) {}
void doSomething(const char* pString) {}
size_t fillArray(double* pArray, size_t arraySize) { return (arraySize - 2); }
size_t fillString(char* pArray, size_t arraySize) { return (arraySize - 2); }
 
int test_item_16()
{
	//@  vector/string --> C API
	std::vector<int> v{ 1, 2 };
	if (!v.empty()) {
		doSomething(&v[0], v.size());
		doSomething(v.data(), v.size()); //@  C++11
		//@  doSomething(v.begin(), v.size()); //@  错误的用法
		doSomething(&*v.begin(), v.size()); //@  可以，但不易于理解
	}
 
	std::string s("xxx");
	doSomething(s.c_str());
 
	//@  C API 初始化vector
	const int maxNumDoubles = 100;
	std::vector<double> vd(maxNumDoubles); //@  创建大小为maxNumDoubles的vector
	vd.resize(fillArray(&vd[0], vd.size())); //@  使用fillArray向vd中写入数据，然后把vd的大小改为fillArray所写入的元素的个数
 
	//@  C API 初始化string
	const int maxNumChars = 100;
	std::vector<char> vc(maxNumChars); //@  创建大小为maxNumChars的vector
	size_t charsWritten = fillString(&vc[0], vc.size()); //@  使用fillString向vc中写入数据
	std::string s2(vc.cbegin(), vc.cbegin() + charsWritten); //@  通过区间构造函数，把数据从vc拷贝到s2中
 
	//@  先让C API把数据写入到一个vector中，然后把数据拷贝到期望最终写入的STL容器中，这一思想总是可行的
	std::deque<double> d(vd.cbegin(), vd.cend()); //@  把数据拷贝到deque中
	std::list<double> l(vd.cbegin(), vd.cend()); //@  把数据拷贝到list中
	std::set<double> s22(vd.cbegin(), vd.cend()); //@  把数据拷贝到set中
 
	//@  除了vector和string以外的其它STL容器也能把它们的数据传递给C API，你只需要把每个容器的元素拷贝到一个vector中，然后传给该API
	std::set<int> intSet; //@  存储要传给API的数据的set
	std::vector<int> v2(intSet.cbegin(), intSet.cend()); //@  把set的数据拷贝到vector
	if (!v2.empty()) doSomething(&v2[0], v2.size()); //@  把数据传给API
 
	return 0;
}
```

# 使用 swap 技巧除去多余的容量

```
vector<int>(v).swap(v);
```

vector<int>(v) 使用 v 创建一个临时变量，v 中空余的内存将不会被拷贝到这个临时变量的空间中，再利用 swap 将这个临时变量与 v 进行交换，相当于去除掉了 v 中的多余内存。

由于 STL 实现的多样行，swap 的方式并不能保证去掉所有的多余容量，但它将尽量将空间压缩到其实现的最小程度。

利用 swap 的交换容器的值的好处在于可以保证容器中元素的迭代器、指针和引用在交换后依然有效。 

```
vector<int> v1;
v1.push_back(1);
vector<int>::iterator i = v1.begin();
 
vector<int> v2(v1);
v2.swap(v1);
cout<<*i<<endl;  //@ output 1  iterator指向v2的begin
```

但是在使用基于临时变量的 swap 要当心 iterator 失效的情况：

```
vector<int> v1;
v1.push_back(1);
vector<int>::iterator i = v1.begin();

vector<int>(v1).swap(v1);
cout<<*i<<endl;  //@ crash here
```

原因在于构造的临时变量在该行结束后就被析构了。

C++11 中增加了 shrink_to_fit 成员函数：

```
class Contestant {};
 
int test_item_17()
{
	//@ 从contestants矢量中除去多余的容量
	std::vector<Contestant> contestants;
	//@ ... //@ 让contestants变大，然后删除它的大部分元素
	//@ vector<Contestant>(contestants)创建一个临时矢量，vector的拷贝构造函数只为所拷贝的元素分配所需要的内存
	std::vector<Contestant>(contestants).swap(contestants);
 
	contestants.shrink_to_fit(); //@ C++11
 
	std::string s;
	/@/ ... //@ 让s变大，然后删除它的大部分字符
	std::string(s).swap(s);
    
	s.shrink_to_fit(); //@ C++11
 
	std::vector<Contestant>().swap(contestants); //@ 清除contestants并把它的容量变为最小
 
	std::string().swap(s); //@ 清除s并把它的容量变为最小
 
	return 0;
}
```































