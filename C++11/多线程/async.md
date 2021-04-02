std::async 比 std::promise、std::packaged_task 和 std::thread 更高一层，可以用来直接创建异步的 task，异步任务的结果也保存在 std::future 中。

std::async  原型：

```
async(std::launch::async | std::launch::deferred,f,args...);
```

- 第一个参数是线程的创建策略：
  - std::launch::async，调用 async 时就开始创建线程，默认是这个选项。
  - std::launch::deferred，延迟加载方式创建线程，调用 async 时不创建线程，直到调用 future 的 get 或者 wait 时才创建。
- f，是线程函数。
- args..，是线程函数的参数。

```
int main(void)
{
	std::future<int> f = std::async(std::launch::async, [] {return 666; });
	std::cout << f.get() << std::endl;


	std::future<int> f2 = std::async(std::launch::async, [] {return 888; });
	f2.wait();

	std::future<int> future = std::async(std::launch::async, [] {std::this_thread::sleep_for(std::chrono::seconds(3));
	return 6; });

	std::cout << "waiting..." << std::endl;
	std::future_status status;
	do
	{
		status = future.wait_for(std::chrono::seconds(1));
		if (status == std::future_status::deferred)
			std::cout << "deferred" << std::endl;
		else if (status == std::future_status::timeout)
			std::cout << "timeout" << std::endl;
		else if (status == std::future_status::ready)
			std::cout << "ready" << std::endl;
	} while (status != std::future_status::ready);

	std::cout << "result is: " << future.get() << std::endl;
	return 0;
}
```

