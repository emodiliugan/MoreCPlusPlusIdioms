# Execute-Around Pointer

##目的

提供一个智能指针对象，在每次函数调用前后都透明的执行相同的操作。这可以看作是面向切面编程(AOP)的一种特殊形式。

##别名

智能指针的双重应用。

##动机

通常需要在每个类成员函数调用之前和之后执行某个功能。例如，在多线程应用程序中，在修改数据之前锁定，修改后解锁。对于一个数据结构，可视化应用程序可能更加关心每次插入或删除操作后其数据结构的大小。

```
using namespace std;
class Visualizer {
	std::vector<int>& vect;
public:
	Visualizer(vector<int>&v) : vect(v) {}
	void data_changed() {
		std::cout << "Now size is: " << vect.size();
	}
};
int main () // A data visualization application.
{
	std::vector<int> vector;
	Visualizer visu(vector);
    //...
    vector.push_back(10);
    visu.data_changed();
    vector.push_back(20);
    visu.data_changed();
    // Many more insert/remove calls here
    // and corresponding calls to visualizer.
}

```

这种重复的函数调用繁琐且容易出错。如果可以自动的调用Visualizer，那就最好不过了。Visualizer不光用于std::vector<int>也可以用于std::list<int>。这种不是单个类的一部分，而是横切多个类的功能，一般被称为切面。这种特殊的惯用法对于设计非常有用。

##解决方案及示例

```
class VisualizableVector {
public:
	class proxy {
	public:
		proxy(vector<int>* v) : vect(v) {
			std::cout << "Before size is: " << vect->size ();
		}
		vector<int>* operator->() {
			return vect;
		}
		~proxy () {
			std::cout << "After size is: " << vect->size ();
		}
	private:
		vector<int>* vect;
	};
	
	VisualizableVector (vector<int>* v) : vect(v) {}
	proxy operator->(){
		return proxy(vect);
	}
private:
	vector<int>* vect;
};
int main()
{
	VisualizableVector vecc(new vector<int>);
	//...
	vecc->push_back(10); //注意是使用->operator而不是.operator
	vecc->push_back(20);
}
```

注解：

**为什么会调用代理类的重载运算符->而不是VisualizableVector的重载运算符->？**原因是VisualizableVector的operator->返回代理类的实例。因此，当你调用vecc->push_back(10)时，实际上是调用代理类的重载运算符->，它返回vector<int>*指针，然后在该指针上调用push_back(10)。

VisualizableVector的重构操作符->创建了一个临时proxy并返回它。proxy对象的构造函数中记录了vector的大小。然后调用proxy的重载->运算符，并通过返回指向vector对象的裸指针，将调用简单地转发到底层vector对象。在对vector的实际调用完成后，调用proxy的析构函数并再次记录其大小。因此，这个logging是透明的并且主函数也变得简洁。这种惯用法是Execute Around Proxy的一种特殊情况,而Execute Around Proxy更加的通用和强大。 

如果我们巧妙地将其与模板结合起来，并将其与重载的->运算符结合，就可以得到这种惯用语的真正威力。

```
template <class NextAspect, class Para>
class Aspect
{
protected:
	Aspect(Para p): para_(p) {}
	Para para_;
public:
	NextAspect operator->()
    {
    	return NextAspect(para_);
    }
};
template <class NextAspect, class Para>
struct Visualizing : Aspect<NextAspect, Para>
{
public:
	Visualizing (Para p):Aspect<NextAspect, Para>(p)
    {
    	std::cout << "Before Visualization aspect" << std::endl;
    }
    ~Visualizing ()
    {
    	std::cout << "After Visualization aspect" << std::endl;
    }
};
template <class NextAspect, class Para>
struct Locking : Aspect<NextAspect, Para>
{
public:
    Locking (Para p): Aspect<NextAspect, Para>(p)
    {
    	std::cout << "Before Lock aspect" << std::endl;
    }
    ~Locking ()
    {
    	std::cout << "After Lock aspect" << std::endl;
    }
};
template <class NextAspect, class Para>
struct Logging : Aspect <NextAspect, Para>
{
public:
    Logging (Para p): Aspect<NextAspect, Para>(p)
    {
    	std::cout << "Before Log aspect" << std::endl;
    }
    ~Logging ()
    {
    	std::cout << "After Log aspect" << std::endl;
    }
};
template <class Aspect, class Para>
class AspectWeaver
{
public:
	AspectWeaver(Para p) : para_(p) {}
    Aspect operator->()
    {
    	return Aspect(para_);
    }
private:
	Para para_;
};

#define AW1(T,U) AspectWeaver<T <U, U>, U >
#define AW2(T,U,V) AspectWeaver<T < U <V, V> , V>, V >
#define AW3(T,U,V,X) AspectWeaver<T < U <V <X, X>, X> , X>, X >

int main()
{
	AW3(Visualizing, Locking, Logging, vector<int>*)
	X(new vector<int>);
	//...
	X->push_back (10); // Note use of -> operator instead of . 								operator
	X->push_back (20);
	return 0;
}
```

##已知的应用
##相关的惯用法
##参考