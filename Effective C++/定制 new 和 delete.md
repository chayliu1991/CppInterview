# new handler 的行为

new 申请内存失败时会抛出 `"bad alloc"` 异常，此前会调用一个由 std::set_new_handler() 指定的错误处理函数。

“new-handler” 函数通过 std::set_new_handler() 来设置，std::set_new_handler() 定义在 `<new>` 中：

```
namespace std{
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw();
}
```

throw() 是一个异常声明，表示不抛任何异常。例如：` void func() throw(Exception1, Exception2)`  表示 func 可能会抛出 Exception1，Exception2 两种异常。

set_new_handler() 的使用也很简单：

```
void outOfMem(){
    std::cout<<"Unable to alloc memory";
    std::abort();
}
int main(){
    std::set_new_handler(outOfMem);
    int *p = new int[100000000L];
}
```

当 new 申请不到足够的内存时，它会不断地调用 outOfMem。因此一个良好设计的系统中 outOfMem 函数应该做如下几件事情之一：

- 使更多内存可用。
- 安装一个新的 ”new-handler”。
- 卸载当前 ”new-handler”，传递 NULL 给 set_new_handler 即可。
- 抛出 bad_alloc（或它的子类）异常。
- 不返回，可以 abort 或者 exit 。

std::set_new_handler 设置的是全局的 bad_alloc 的错误处理函数，C++ 并未提供类型相关的 bad_alloc 异常处理机制。 但我们可以重载类的 operator new，当创建对象时暂时设置全局的错误处理函数，结束后再恢复全局的错误处理函数。

比如 Widget 类，首先需要声明自己的 set_new_handler 和 operator new：

```
class Widget{
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void * operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler current;
};
 
//@ 静态成员需要定义在类的外面
std::new_handler Widget::current = 0;
std::new_handler Widget::set_new_handler(std::new_handler p) throw(){
    std::new_handler old = current;
    current = p;
    return old;
}
```

关于 abort, exit, terminate 的区别：

- abort 会设置程序非正常退出.
- exit 会设置程序正常退出，当存在未处理异常时C++会调用 terminate， 它会回调由 std::set_terminate 设置的处理函数，默认会调用 abort。

最后来实现 operator new，该函数的工作分为三个步骤：

- 调用 std::set_new_handler，把 Widget::current 设置为全局的错误处理函数。
- 调用全局的 operator new 来分配真正的内存。
- 如果分配内存失败，Widget::current 将会抛出异常。
- 不管成功与否，都卸载 Widget::current，并安装调用 Widget::operator new 之前的全局错误处理函数。

我们通过RAII类来保证原有的全局错误处理函数能够恢复，让异常继续传播：

```
class NewHandlerHolder{
public:
    explicit NewHandlerHolder(std::new_handler nh): handler(nh){}
    ~NewHandlerHolder(){ std::set_new_handler(handler); }
private:
    std::new_handler handler;
    NewHandlerHolder(const HandlerHolder&);     //@ 禁用拷贝构造函数
    const NewHandlerHolder& operator=(const NewHandlerHolder&); //@ 禁用赋值运算符
};
```

然后 Widget::operator new 的实现其实非常简单：

```
void * Widget::operator new(std::size_t size) throw(std::bad_alloc){
    NewHandlerHolder h(std::set_new_handler(current));
    return ::operator new(size);    //@ 调用全局的new，抛出异常或者成功
}   //@ 函数调用结束，原有错误处理函数恢复
```

客户使用 Widget 的方式也符合基本数据类型的惯例： 

```
void outOfMem();
Widget::set_new_handler(outOfMem);
 
Widget *p1 = new Widget;    //@ 如果失败，将会调用outOfMem
//@ 如果失败，将会调用全局的 new-handling function，当然如果没有的话就没有了
string *ps = new string;   

Widget::set_new_handler(0); //@ 把Widget的异常处理函数设为空
Widget *p2 = new Widget;    //@ 如果失败，立即抛出异常
```

仔细观察上面的代码，很容易发现自定义 ”new-handler” 的逻辑其实和 Widget 是无关的。我们可以把这些逻辑抽取出来作为一个模板基类：

```
template<typename T>
class NewHandlerSupport{
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void * operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler current;
};
 
template<typename T>
std::new_handler NewHandlerSupport<T>::current = 0;
 
template<typename T>
std::new_handler NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw(){
    std::new_handler old = current;
    current = p;
    return old;
}
 
template<typename T>
void * NewHandlerSupport<T>::operator new(std::size_t size) throw(std::bad_alloc){
    NewHandlerHolder h(std::set_new_handler(current));
    return ::operator new(size);
}
```

有了这个模板基类后，给 Widget 添加 ”new-handler” 支持只需要 public 继承即可：

```
class Widget: public NewHandlerSupport<Widget>{ ... };
```

其实 NewHandlerSupport 的实现和模板参数 T 完全无关，添加模板参数是因为 handler 是静态成员，这样编译器才能为每个类型生成一个 handler 实例。

1993 年之前 C++ 的 operator new 在失败时会返回 null 而不是抛出异常。如今的 C++ 仍然支持这种 nothrow 的operator new：

```
Widget *p1 = new Widget;    // @失败时抛出 bad_alloc 异常
assert(p1 != 0);            //@ 这总是成立的
Widget *p2 = new (std::nothrow) Widget;
if(p2 == 0) ...             //@ 失败时 p2 == 0
```

“nothrow new”  只适用于内存分配错误。而构造函数也可以抛出的异常，这时它也不能保证是 new 语句是 ”nothrow” 的。

# 了解 new 和 delete 所谓合理替换时机

为什么需要自定义 operator new 或 operator delete ?

- 检测使用错误。new 得到的内存如果没有 delete 会导致内存泄露，而多次 delete 又会引发未定义行为。如果自定义 operator new 来保存动态内存的地址列表，在 delete 中判断内存是否完整，便可以识别使用错误，避免程序崩溃的同时还可以记录这些错误使用的日志。
- 提高效率。全局的 new 和 delete 被设计为通用目的（general purpose）的使用方式，通过提供自定义的new，我们可以手动维护更适合应用场景的存储策略。
- 收集使用信息。在继续自定义 new 之前，你可能需要先自定义一个 new 来收集地址分配信息，比如动态内存块大小是怎样分布的？分配和回收是先进先出 FIFO 还是后进先出 LIFO？
- 实现非常规的行为。比如考虑到安全，operator new 把新申请的内存全部初始化为0。
- 其他原因，比如抵消平台相关的字节对齐，将相关的对象放在一起等等。

自定义一个 operator new 很容易的，比如实现一个支持越界检查的 new：

```
static const int signature = 0xDEADBEEF;    //@ 边界符
typedef unsigned char Byte; 
 
void* operator new(std::size_t size) throw(std::bad_alloc) {
    //@ 多申请一些内存来存放占位符 
    size_t realSize = size + 2 * sizeof(int); 
 
    //@ 申请内存
    void *pMem = malloc(realSize);
    if (!pMem) throw bad_alloc(); 
 
    //@ 写入边界符
    *(reinterpret_cast<int*>(static_cast<Byte*>(pMem)+realSize-sizeof(int))) 
        = *(static_cast<int*>(pMem)) = signature;
 
    //@ 返回真正的内存区域
    return static_cast<Byte*>(pMem) + sizeof(int);
}
```

其实上述代码是有一些瑕疵的：

- operator new 应当不断地调用 new handler，上述代码中没有遵循这个惯例。
- 有些体系结构下，不同的类型被要求放在对应的内存位置。比如 double 的起始地址应当是 8 的整数倍，int 的起始地址应当是 4 的整数倍。上述代码可能会引起运行时硬件错误。
- 起始地址对齐。C++要求动态内存的起始地址对所有类型都是字节对齐的，new 和 malloc 都遵循这一点，然而我们返回的地址偏移了一个 int。 

实现一个 operator new 很容易，但实现一个好的 operator new 却很难。

# 编写 new 和 delete 时需固守常规

new 和delete 必须遵循的惯例：

- operator new 需要无限循环地获取资源，如果没能获取则调用 ”new handler”，不存在 ”new handler” 时应该抛出异常。 
- operator new 应该处理 size == 0 的情况。
- operator delete 应该兼容空指针。
- operator new/delete 作为成员函数应该处理 size > sizeof(Base) 的情况（因为继承的存在）。

## 外部 operator new/delete

先看看如何实现一个外部（非成员函数）的 operator new： 

- 给出返回值很容易。当内存足够时，返回申请到的内存地址；当内存不足时，返回空或者抛出 bad_alloc 异常。
- 每次失败时调用 ”new handler”，并重复申请内存却不太容易。只有当 ”new handler” 为空时才应抛出异常。
- 申请大小为零时也应返回合法的指针。允许申请大小为零的空间确实会给编程带来方便。

考虑到上述目标，一个非成员函数的 operator new 大致实现如下：

```
void * operator new(std::size_t size) throw(std::bad_alloc)
{
    if(size == 0) size = 1;
    while(true)
    {
        //@ 尝试申请
        void *p = malloc(size);
 
        //@ 申请成功
        if(p) return p;
 
        //@ 申请失败，获得new handler
        new_handler h = set_new_handler(0);
        set_new_handler(h);
 		
 		//@ 调用 new_handler
        if(h) 
        	(*h)();
        else 
        	throw bad_alloc();
    }
}
```

- size == 0 时申请大小为 1 看起来不太合适，但它非常简单而且能正常工作。况且你不会经常申请大小为 0 的空间吧？
- 两次 set_new_handler 调用先把全局 ”new handler” 设置为空再设置回来，这是因为无法直接获取”new handler”，多线程环境下这里一定需要锁。
- while(true) 意味着这可能是一个死循环。所以Item 49提到，”new handler” 要么释放更多内存、要么安装一个新的  ”new handler”，如果你实现了一个无用的 ”new handler” 这里就是死循环了。

相比于 new，实现 delete 的规则要简单很多。唯一需要注意的是 C++ 保证了 delete  一个  NULL 总是安全的，你尊重该惯例即可。

同样地，先实现一个外部（非成员）的 delete：

```
void operator delete(void *rawMem) throw(){
    if(rawMem == 0) return; 
    // 释放内存
}
```

## 成员 operator new/delete





