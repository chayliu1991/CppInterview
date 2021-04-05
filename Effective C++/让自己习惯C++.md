# 视 C++为一个语言联邦

C++ 是多范式的程序设计语言。同时支持：过程式编程，面向对象编程，函数式编程，泛型编程，元编程。

C++ 的四种次语言：

- **C 语言**。C++ 是基于 C 设计的，你可以只使用 C++ 中 C 的那部分语法。此时你会发现你的程序反映的完全是C的特征：没有模板、没有异常、没有重载。
- **Object-Oriented C++**。面向对象程序设计也是 C++ 的设计初衷：构造与析构、封装与继承、多态、动态绑定的虚函数。
- **Template C++**。这是 C++ 的泛型编程部分。另外模板元编程也是一个新兴的程序设计范式，虽然有点非主流。
- **STL**。这是一个特殊的模板库，它的容器、迭代器和算法优雅地结合在一起，只是在使用时你需要遵循它的程序设计惯例。当然你也可以基于其他想法来构建模板库。

C++ 程序设计的惯例并非一尘不变，而是取决于你使用 C++ 语言的哪一部分。

# 尽量使用 const、enum、inline 等替换 #define

## const 常量代替 #define

- const 常量能够出现在符号表中，方便调试，宏定义的名字不会被添加到符号表中，给调试带来很大的麻烦。
- 宏定义因为进行的宏替换，有时候会造成代码冗余，const 常量能够很好的避免这个问题。

```
#define ASPECT_RATIO 1.653     		//@ 不推荐
const double AspectRatio = 1.653;	//@ 推荐	
```

- const 常量可以作为类属成员，#define 则毫无封装性。
- 整型族类属常量可以在类中声明时直接初始化。

```
class GamePlayer {
private:
  static const int NumTurns = 5;      //@ 声明一个类属常量
  int scores[NumTurns];               //@ 使用常量
  ...
};
```

上面 NumTurns 是 declaration（声明），而不是 definition（定义），还需要提供一个独立的定义：

```
const int GamePlayer::NumTurns;
```

- 应该把它放在一个实现文件而非头文件中。
- 因为类属常量的初始值在声明时已经提供初值，在此无须重复给初值
- 定义时不需要 static 关键字。

## enum 

enum 也可以作为整型常量使用，并且无法取得其地址。

```
class GamePlayer {
private:
  enum { NumTurns = 5 };                           
  int scores[NumTurns];             
  ...
};
```

## inline 函数代替 #define

使用内联函数代替宏定义的函数将会在不损失效率的情况下降低发生错误的可能性。

```
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

编写这样的宏一定要带上括号，但是有时候即使带上括号，也无法避免错误：

```
int a = 5, b = 0;
CALL_WITH_MAX(++a, b); 		//@ a 自增了两次
CALL_WITH_MAX(++a, b+10); 	//@ a 自增了一次
```

可以通过一个内联函数的模板来获得宏的效率，以及完全可预测的行为和常规函数的类型安全：

```
template<typename T>    
inline void callWithMax(const T& a, const T& b) 
{                                       
	f(a > b ? a : b);                        
}
```

# 尽量使用 const

## const 与指针

- const 出现在 `*` 左边，则指针指向的内容是 const。
- const 出现在 `*` 右边，则指针本身是 const。
- const 出现在  `*` 两边，两者都是 const。

```
char greeting[] = "Hello";
char* p = greeting;	//@ non-const data,non-const pointer
const char* p = greeting;	//@ non-const pointer,const data
char* const p = greeting;	//@ const pointer,non-const data
const char* const p = greeting; //@ const pointer,const data
```

当指针指向的内容是常量时，将 const 放在类型前和放在类型后是没有区别的：

```
//@ 等价的形式
void f1(const Widget *pw);	
void f1(Widget const *pw);	
```

当指针指向的内容是常量时，表示无法通过指针修改变量的值，但是可以通过其它方式修改指针指向变量的值：

```
int a = 1;
const int *p = &a;
cout << *p << endl;	//@ 1
*p = 2;	//@ error, data is const
a = 2;
cout << *p << endl;	//@ 2
```

指针本身是常量，表示指针表示的地址是固定的，但是其指向的内容是可以改变的：

```
int a = 1, b = 2;
int* const p = &a;
cout << *p << endl;	//@ 1
p = &b;	//@ error, pointer is const
*p = b;
cout << *p << endl;	//@ 2
```

## const 与迭代器

- const iterator：表示 iterator 本身是常量，不能将这个 iterator 指向另外不同的东西，但是它所指向的东西可以变化。
- conts_iterator：表示  iterator 指向的内容不能发生变化，但是其本身可以变化。

```
std::vector<int> vec;

const std::vector<int>::iterator iter = vec.begin();
*iter = 10;	//@ 正确，迭代器指向的内容可以变化
iter++;		//@ 错误，迭代器本身是常量不可以变化

std::vector<int>::const_iterator cIter =  vec.begin();
*cIter = 10;	//@ 错误，迭代器指向的内容是常量不可以变化
++cIter;		//@ 正确，迭代器本身可以变化
```

## const 与函数

const 可以用在函数返回值，函数参数，对于成员函数，还可以用于整个函数。

- 无论何时，只要可以就应该将函数的参数声明为 const 类型，除非需要改变这个参数。
- 函数返回 const value 常常可以在不放弃安全和效率的前提下尽可能减少客户造成的影响：

```
class Rational{...};
const Rational operator*(const Rational& lhs,const Rational& rhs);
```

上面函数返回 const object 的原因，可以避免客户如下暴行：

```
Rational a,b,c;
...
(a * b) = c;	//@ 为两个数的乘积赋值，将返回值声明为 const 可以避免此问题
```

const 成员函数的优点：

- 明确哪些方法可以供常量对象调用。
- 使用接口是否会改变对象的意图显而易见。

const 成员函数的注意事项：

- 常量对象只能调用常量方法， 非常量对象优先调用非常量方法，如不存在会调用同名常量方法。
- 常量成员函数也可以在类声明外定义，但声明和定义都需要指定 const 关键字。
- 成员方法添加常量限定符属于函数重载。

```
class TextBlock {
public:
  ...  
  //@ operator[] for const objects
  const char& operator[](std::size_t position) const  
  { return text[position]; }                          

  //@ operator[] for non-const objects
  char& operator[](std::size_t position)           
  { return text[position]; }                          

private:
   std::string text;
};

//@ 使用
TextBlock tb("Hello");
std::cout << tb[0];	//@ 调用 non-const TextBlock::operator[]
                                       
const TextBlock ctb("World");
std::cout << ctb[0];  //@ 调用 const TextBlock::operator[]
```

const objects 常常作为函数的参数传递：

```
void print(const TextBlock& ctb)      
{
  std::cout << ctb[0];   //@ 调用 const TextBlock::operator[]
  ...
}
```

对 const 和 non-const 的 TextBlocks 做不同的操作：

```
std::cout << tb[0];   
tb[0] = 'x';         //@ 因为返回类型是非常量引用，此处正确
std::cout << ctb[0]; 
ctb[0] = 'x';       //@ 错误，返回的常量对象不允许被修改
```

non-const 版本的 operator[] 的返回类型是一个 char 的引用而不是一个 char 本身。如果 operator[] 只是返回一个简单的 char，下面的语句将无法编译：

```
tb[0] = 'x';	//@ 相当于给右值赋值
```

## 比特常量和逻辑常量

编译器强制实行 **bitwise constness** (又称 physical constness,物理上的常量性，即成员函数不更改对象的任何一个 bit 时才可以说是 const)，例如：

```
class CTextBlock {
public:
  ...
  char& operator[](std::size_t position) const   
  { return pText[position]; }                   
                                               
private:
  char *pText;
};
```

 编译器认定它是 bitwise constness 的，但是它却允许以下代码的存在：

```
const CTextBlock cctb("Hello");  //@ 声明有一个常量对象
char *pc = &cctb[0];
*pc = 'J'; //@ cctb 当前的值是 "Jello"
```

这是由于只有 pText 是 cctb 的一部分，其指向的内存并不属于 cctb。

编写程序时应该使用 **conceptual constness** (概念上的常量性或 **logical constness**，逻辑上的常量性）即一个const 成员函数可以处理它所修改的对象的某些 bits，但只有在客户端侦测不出的情况下才得如此。

例如，对于某些特殊类，其中的某些成员的值注定是要改变的，因此可以用 mutable 关键字修饰，从而实现即使对象被设定为 const，其特定成员的值仍然可以改变的效果。此时该类符合 conceptual constness 而不符合 bitwise constness。

```
class CTextBlock {
public:
  ...
  std::size_t length() const;
private:
  char *pText;

  mutable std::size_t textLength;        
  mutable bool lengthIsValid;             
};    

std::size_t CTextBlock::length() const
{
  if (!lengthIsValid) {
    textLength = std::strlen(pText);     
    lengthIsValid = true;        
  }
  return textLength;
}
```

## 避免常量和非常量方法的重复

通常我们需要定义成对的常量和普通方法，只是返回值的修改权限不同。 当然我们不希望重新编写方法的逻辑。最先想到的方法是常量方法调用普通方法，然而这是 C++ 语法不允许的。 于是我们只能用普通方法调用常量方法，并做相应的类型转换：

```
class TextBlock {
public:
	...
		const char& operator[](std::size_t position) const
	{
		...
			return text[position];
	}

	char& operator[](std::size_t position)
	{
		return const_cast<char&>(
			static_cast<const TextBlock&>(*this)[position]);
	}
	...
};
```

# 确保对象在使用前被初始化















