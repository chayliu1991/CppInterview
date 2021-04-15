RAII(Resource Acquire Is Initialization)，是 C++ 所特有的资源管理方式。

RAII 依托栈和析构函数，来对所有的资源进行管理。

例如下面工厂方法，使用 RAII 方式管理：

```
enum class ShpaeType
{
	circle,
	triangle,
	rectange,
};

class Shape {};
class Circle : public Shape {};
class Triangle : public Shape {};
class Rectange : public Shape {};

class ShapeWarper
{
public:
	explicit ShapeWarper(Shape* ptr):ptr_(ptr)
	{		
	}

	~ShapeWarper()
	{
		delete ptr_;
	}

private:
	Shape* ptr_;
};

ShapeWarper CreateShpe(ShpaeType type)
{
	switch (type)
	{
	case ShpaeType::circle:
		return ShapeWarper(new Circle());
	case ShpaeType::triangle:
		return ShapeWarper(new Triangle());
	case ShpaeType::rectange:
		return ShapeWarper(new Rectange());
	default:
		throw "no this shape type";
	}
}
```





