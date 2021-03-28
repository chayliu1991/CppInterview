C++ 11 中的无序容器类型有：

- unordered_map/unordered_multimap
- unordered_set/unordered_multiset

有序容器内部是红黑树，插入元素时会自动排序。

无序容器内部是散列表，通过哈希实现，不需要排序，因而效率更高。

因为无序容器内部是散列表，因此无序容器的 key 需要提供 hash_value 函数。对于自定义 key，需要提供 hash 函数和比较函数。

```
struct Key
{
	std::string first;
	std::string second;
};

struct KeyHash
{
	std::size_t operator()(const Key& k) const
	{
		return std::hash<std::string>()(k.first) ^ (std::hash<std::string>()(k.second) << 1);
	}
};

struct KeyEqual
{
	bool operator()(const Key& lhs, const Key& rhs) const
	{
		return lhs.first == rhs.first && lhs.second == lhs.second;
	}
};

int main()
{
	//@ 自定义类型需要提供哈希函数和比较函数
	std::unordered_map<Key, std::string, KeyHash, KeyEqual> m = {
		{{"John","Doe"},"example"},
		{{"Mary","Sue"},"another"}
	};
	return 0;
}
```









