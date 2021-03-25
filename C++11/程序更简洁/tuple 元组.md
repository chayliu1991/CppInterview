# std::tuple 简介

- std::tuple 元组是一个固定大小的不同类型值的集合。
- 可以当作结构体用，又不需要创建结构体，但是对于多段结构体，为了可读性，建议不使用。

# 创建 std::tuple

- 使用 std::make_tuple
- 使用 std::tie 创建左值引用元组
- 使用 std::forward_as_tuple 创建右值引用元组

```
std::tuple<std::string, int> tp = std::make_tuple("hello",1);

int x = 1;
std::string str = "hello";
auto tp = std::tie(str, x); //@ tp-> std::tuple<std::string&, int&>

void print_pack(std::tuple<std::string&&, int&&> pack) 
{
	std::cout << std::get<0>(pack) << ", " << std::get<1>(pack) << '\n';
}
std::string str("John");
print_pack(std::forward_as_tuple(str + " Smith", 25));
```

# 获取 std::tuple 的值

- 使用 std::get  获取元组中的元素
- 使用 std::tie 解包，使用 std::ignore 忽略不想解包的元素

```
auto tp = std::make_tuple("hello",123);
auto elem1 = std::get<0>(tp); //@ 获取元组第一个元素
auto elem2 =  std::get<1>(tp); //@ 获取元组第二个元素

std::string str; 
int i;
std::tie(str, i) = tp;

std::string s; 
std::tie(s, std::ignore) = tp;
```

# 连接两个 std::tuple

```
auto tp1 = std::make_tuple("hello",123);
std::tuple<float, std::string, int> tp2{ 3.14,"world",8 };
auto tp = std::tuple_cat(tp1,tp2); //@  "hello", 123, 3.14000010, "world", 8
```



