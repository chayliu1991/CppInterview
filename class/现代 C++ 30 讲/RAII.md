RAII(Resource Acquire Is Initialization)，是 C++ 所特有的资源管理方式。

RAII 依托栈和析构函数，来对所有的资源进行管理。

例如下面工厂方法，使用 RAII 方式管理：

```
enum class ShapeType
{
	circle,
	triangle,
	rectangle,
};

class Shape
{
public:
	virtual void draw()
	{
		printf("Shape draw()\n");
	};
};

class Circle : public Shape
{
public:
	void draw() override
	{
		printf("Circle draw()\n");
	}
};

class Triangle : public Shape
{
public:
	void draw() override
	{
		printf("Triangle draw()\n");
	}
};

class Line : public Shape
{
public:
	void draw() override
	{
		printf("Line draw()\n");
	}
};

class ShapeWrapper
{
public:
	explicit ShapeWrapper(Shape* ptr) :ptr_(ptr)
	{
	}

	~ShapeWrapper()
	{
		delete ptr_;
	}

private:
	Shape* ptr_;
};

ShapeWrapper create_shape(ShapeType type)
{
	switch (type)
	{
	case ShapeType::circle:
		return ShapeWrapper(new Circle());
	case ShapeType::triangle:
		return ShapeWrapper(new Triangle());
	case ShapeType::rectangle:
		return ShapeWrapper(new Line());
	default:
		throw "no this shape type";
	}
}
```





