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
std::chrono::duration<double, std::ratio<9, 7>> dura1(3);
std::chrono::duration<double, std::ratio<6, 5>> dura2(1);
auto dura3 = dura1 - dura2;

//@ class std::chrono::duration<double,struct std::ratio<3,35> >
std::cout << typeid(dura3).name() << std::endl;
std::cout << dura3.count() << std::endl;  //@ 31
```

- 9/7,6/5 分子取最大公约数 3，分母取最小公倍数 35，所以统一的形式为：`class std::chrono::duration<double,struct std::ratio<3,35>>`
- 计算时钟周期的方式：`((9 / 7) / (3 / 35) * 3) - ((6 / 5) / (3 / 35) * 1)`

可以通过 duration_cast<>()  将当前的时钟周期转换成其它的时钟周期：

```
std::cout << std::chrono::duration_cast<std::chrono::seconds>(dura3).count() << " minutes" << std::endl;
```

# time_point

time_point 表示一个时间点，用来获取从它的 clock 纪元(比如 1970.1.1) 开始所经历的 duration，time_point  必须用 clock 来计时，time_point  有一个函数 time_from_epoch() 来获取从 1970年1月1日到 time_point  时间经过的 duration。

 ```
using namespace std::chrono;
typedef duration<int, std::ratio<60 * 60 * 24>> days_type;
time_point<system_clock, days_type> today = time_point_cast<days_type>(system_clock::now());
std::cout << today.time_since_epoch().count() << " days since epoch" << std::endl;
 ```

不同 clock 的 time_point 不能进行算术运算。

time_point  可以和 duration 相加减：

```
using namespace std::chrono;
system_clock::time_point now = system_clock::now();
std::time_t last = system_clock::to_time_t(now - hours(24));
std::time_t next = system_clock::to_time_t(now + hours(24));

std::cout << "One day ago,the time was" << std::put_time(std::localtime(&last), "%F %T") << std::endl;
std::cout << "One day after,the time is" << std::put_time(std::localtime(&next), "%F %T") << std::endl;
```

# clocks

clocks 表示当前的系统时钟，内部有 time_point，duration，Rep，Period 等信息，主要用来获取当前时间，以及实现 time_t 和 time_point 的相互转换，clocks 包含如下 3 种时钟：

- system_clock，代表真实世界的挂钟时间，具体事件值依赖于系统，system_clock 保证提供的时间值是一个可读时间。
- steady_clock，不能被调整的时钟，并不一定代表真实世界的挂钟时间，保证先调用 now() 得到的时间值不会递减。
- high_resolution_clock，高精度时钟，实际上是 system_clock 或者 steady_clock 的别名，可以通过 now()  来获取当前的时间点。

```
using namespace std::chrono;
system_clock::time_point t1 = system_clock::now();
std::cout << "hello,world\n";
system_clock::time_point t2 = system_clock::now();
std::cout << (t2 - t1).count() << " tick counts "<<std::endl;
```

通过 duration_cast 将其转换为其它时钟周期的 duration：

````
std::cout << duration_cast<nanoseconds>(t2 - t1).count() << " nanoseconds" << std::endl;
std::cout << duration_cast<microseconds>(t2 - t1).count() << " microseconds" << std::endl;
std::cout << duration_cast<milliseconds>(t2 - t1).count() << " milliseconds" << std::endl;
````

system_clock  的 to_time_t 方法可以将一个 time_point 转换成 ctime：

```

```







