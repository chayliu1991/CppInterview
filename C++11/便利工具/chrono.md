chrono 提供的三种时间类型：

- duration，时间间隔
- clocks，时钟
- time point，时间点

# duration

duration 表示时间间隔，用来记录时间长度，其原型如下：

```
template <class Rep,class Period = std::ratio<1,1>>
class duration;
```

- Rep，数值类型，表示时钟数的类型。
- Period，是一个默认模板参数 std::ratio，表示每个时钟周期的秒数，其原型：

```
template <std::intmax_t Num,std::intmax_t Denom = 1>
class ratio;
```

- Num，代表分子。
- Denom，代表分母，默认为1。

std::ratio 代表的是一个分子除以分母的分数值，比如：

```
std::ratio<2>  //@ 代表一个时钟周期是 2 秒。
std::ratio<60> //@ 代表一分钟
std::ratio<60*60> //@ 代表一小时
std::ratio<1，1000> //@ 一毫秒
std::ratio<1，1000000> //@ 一微秒
std::ratio<1，1000000000> //@ 一纳秒
```

标准库定义了一些常用的时间间隔：

```
typedef duration <Rep,ratio<3600,1>> hours;
typedef duration <Rep,ratio<60,1>> minutes;
typedef duration <Rep,ratio<1,1>> seconds;
typedef duration <Rep,ratio<1,1000>> milliseconds;
typedef duration <Rep,ratio<1,1000000>> microseconds;
typedef duration <Rep,ratio<1,1000000000>> nanoseconds;
```

实际使用时：

```
std::this_thread::sleep_for(std::chrono::milliseconds(2));
```

获取时间间隔的时钟周期数的方法 count()：

```
int main()
{
	std::chrono::milliseconds ms(3);
	std::chrono::microseconds us = 2 * ms;
	std::cout << "3 ms duration has " << ms.count() << " ticks\n" <<
		"6000 us duration has " << us.count() << " ticks\n";
	return 0;
}
```

计算时间间隔：

```
std::chrono::minutes t1(10);  //@ 10 分钟，600 秒
std::chrono::seconds t2(60);
std::chrono::seconds t3 = t1 - t2; //@ 540秒，t3的时钟周期是 1秒
std::cout << t3.count() << " seconds" << std::endl;
```

duration 的加减运算有一定规则，当两个 duration 的时钟周期不同的时候，会先统一成一种时钟，然后再作加减运算，统一成一种时钟的规则如下：

```

```









