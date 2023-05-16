# Boost mutant

##目的

在不重组或复制数据项的情况下，反转一对普通旧数据（POD）类型。

##别名

##动机

这种惯用语最好的动机是使用 Boost.Bimap 数据结构。

Boost.Bimap 是一个用于 C++ 的双向映射库。在 bimap<X,Y> 中，类型 X 和 Y 的值都可以作为键。这样的数据结构的实现可以使用 boost 变异体惯用语进行优化。

##解决方案及示例

Boost mutant惯用法使用  reinterpret_cast，并且严格依赖于这样的情况：两个互换的不同结构的内存布局具有相同数据成员（类型和顺序）。尽管 C++  标准没有保证此属性，但几乎所有编译器都满足它。此外，只要使用 POD 类型，惯用法就是标准的。下面的示例显示了 boost mutant惯用法的工作原理。

```
template <class Pair>
struct Reverse
{
	typedef typename Pair::first_type second_type;
	typedef typename Pair::second_type first_type;
	second_type second;
	first_type first;
};

template <class Pair>
Reverse<Pair>& mutate(Pair & p)
{
	return reinterpret_cast<Reverse<Pair> &>(p);
}
int main(void)
{
	std::pair<double, int> p(1.34, 5);
	std::cout << "p.first = " << p.first << ", p.second = " 				<<p.second <<std::endl;
	std::cout << "mutate(p).first = " << mutate(p).first << ", 			mutate(p).second =" << mutate(p).second << std::endl;
}
```

给定一个只包含POD数据成员的std::pair<X,Y>对象，大多数编译器上，Reverse<std::pair<X,Y>>的布局与pair的布局相同。Reverse模板将数据成员的名称反转，而不是物理意义上的反转数据。使用一个mutate的辅助函数可以轻松地构造一个Reverse<Pair>引用，它可以被视为原始pair对象上的视图。上面程序的输出证实了，可以在不重新组织数据的情况下获得反转：

```
 p.first = 1.34，p.second = 5
 mutate(p).first = 5，mutate(p).second = 1.34
```

##已知的应用

• [Boost.Bimap](http://www.boost.org/doc/libs/1_43_0/libs/bimap/doc/html/index.html)

##相关的惯用法

##参考

Boost.Bimap是一个C++库，用于实现双向映射数据结构，由Matias  Capeletto开发。这个库提供了一个bimap类，它允许用户同时按照两种不同的方式对数据进行索引。例如，在一个bimap<X,Y>中，类型X和Y的值都可以用作键，这使得用户可以通过X来查找Y，或者通过Y来查找X。Boost.Bimap库还支持多种不同的索引方式，例如序列化索引、多重映射和集合映射。此外，Boost.Bimap库还使用了许多C++11特性，例如右值引用和变长模板参数包。
