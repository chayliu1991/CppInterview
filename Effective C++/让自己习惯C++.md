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















