# final

final 关键字限制：

- 某个类不能被继承。
- 某个虚函数不能被重写，只能修饰虚成员函数。

```
struct A
{
	virtual void foo() final {}

	//void bar() final {} //@ 错误，final 不能修饰非虚成员函数
};


struct B final : A
{
	virtual void foo() {} //@ 错误，foo 虚函数不能被重写
};

struct C ： B //@ struct B 不能被其它类继承
{
};
```

# override

override 关键字：

- 确保在派生类中声明的重写函数与基类的虚函数有相同的签名。
- 表明某个虚函数一定会被重写。
- 防止在派生类中错误的将本应该重写的基类虚函数写成覆盖了基类虚函数的形式。

```
struct A
{
	virtual void foo() {}
	void bar() {}
};

struct B : A
{
	void foo() override{} //@ 表明要重写基类的 foo 虚函数
	
	void bar() override {} //@ 错误，bar 不是虚函数

	void foo(int a) override {} //@ 错误，派生类试图重写的函数与基类的函数签名并不一致

};
```





