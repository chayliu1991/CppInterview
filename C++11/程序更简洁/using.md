# using 可读性更好

定义函数指针时，使用 `using` 别名可读性更好：

```
typedef void(*Func)(int);
using F = void(*)(int);
```

# 定义模板别名的唯一选择

定义模板别名时候，必须使用 `using`：

```
template<typename T>
using Vec = std::vector<int>; //@ Vec<int> 等价于 std::vector<int>
```

