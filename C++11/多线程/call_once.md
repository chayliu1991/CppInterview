某个操作只能执行一次时可以使用 call_once，call_once 需要使用一个 once_flag 作为入参。

```
std::once_flag flag;
void do_once()
{
	std::call_once(flag, []() {std::cout << "Called Once\n"; });
}

int main(void)
{
	std::thread t1(do_once);
	std::thread t2(do_once);
	std::thread t3(do_once);

	t1.join();
	t2.join();
	t3.join();

	return 0;
}
```



