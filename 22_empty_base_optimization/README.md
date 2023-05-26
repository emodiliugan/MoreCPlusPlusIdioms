# 空基类优化(Empty Base Optimization)

##目的

优化空类数据成员的存储

##别名

- EBCO: Empty Base Class Optimization
- 空成员优化

##动机

在C++中，空类时常出现。C++ 要求空类有非零的大小来区分它的对象。举个例子，下面的 EmptyClass 数组必须具有非零大小，因为由数组下标标识的每个对象必须是独一无二的。如果 sizeof(EmptyClass) 为零，则指针的算术运算将无法执行。一般的情况，这样的空类大小是1。

```
class EmptyClass {};
EmptyClass arr[10]; //这个数组的大小不能为零。
```

当相同的空类出现在其他类的数据成员时，它会占用多于一个字节的空间。编译器通常将数据对齐到 4 字节边界，因此空类对象占用的四个字节只是占位符，没有任何有用的作用。避免浪费空间是所有人追求的，可以节省内存并将更多对象装入 CPU  cache lines(CPU的缓存机制)。

##解决方案及示例

当空类被继承时，C++ 会对空类进行特殊处理。编译器可以按照某种方式展开继承层次结构，使空基类不占用空间。例如，在下面的示例中，sizeof(AnInt) 在 32 位架构上为 4 字节，sizeof(AnotherEmpty) 为 1 字节，尽管它们都继承自 EmptyClass。

```
class AnInt : public EmptyClass
{
	int data;
}; //AnInt size = sizeof(int)
class AnotherEmpty : public EmptyClass {};//AnotherEmpty size = 1
```

EBCO以一套策略来进行优化。将空类从成员列表移到基类列表是不可取的，因为这可能会暴露出对用户隐藏的接口。例如，以下应用 EBCO 的方法，会触发优化，但可能也会产生不良的副作用：E1、E2 中的函数（如果有的话）现在对 Foo 类的用户是可见的（尽管由于私有继承，他们无法调用它们）。

```
class E1 {};
class E2 {};
// **** before EBCO ****
class Foo {
E1 e1;
E2 e2;
int data;
}; // sizeof(Foo) = 8
// **** after EBCO ****
class Foo : private E1, private E2 {
int data;
}; // sizeof(Foo) = 4
```

在实际中，有一个使用 EBCO 的方法是将空类作为成员来进行组合，以更高的效率管理空间。以下的 BaseOptimization 模板对其前两个类型参数应用 EBCO惯用法。上面的 Foo 类使用它加以重写。

```
template <class Base1, class Base2, class Member>
struct BaseOptimization : Base1, Base2
{
	Member member;
	BaseOptimization() {}
	BaseOptimization(Base1 const& b1, Base2 const& b2, Member 					const& mem): Base1(b1),Base2(b2),member(mem) { }
};
class Foo {
	BaseOptimization<E1, E2, int> data;
}; //sizeof(Foo) = 4

************上面与下面的相比较来看****************
class Foo {
E1 e1;
E2 e2;
int data;
}; // sizeof(Foo) = 8
```

使用这种技术，Foo 类的继承关系不会发生任何改变。请注意，在上面显示的方法中，基类之间不能发生冲突。也就是说，Base1 和 Base2 是独立层次结构的一部分。

##注意

对象标识问题在不同的编译器上似乎并不一致。空对象的地址可能相同，也可能不同。例如，在 BaseOptimization 类的 first 和  second 成员方法返回的指针可能在某些编译器上相同，在其他编译器上则不同。有关更多讨论，请参见 [StackOverflow](https://stackoverflow.com/questions/7694158/boost-compressed-pair-and-addresses-of-empty-objects)。

##已知的应用

- [boost::compressed_pair](http://www.boost.org/doc/libs/1_47_0/libs/utility/compressed_pair.htm) makes use of this technique to optimize the size of the pair.

- A C++03 emulation of [unique_ptr](http://home.roadrunner.com/~hinnant/unique_ptr03.html) also uses this idiom

##相关的惯用法
##参考

- [The Empty Base Class Optimization (EBCO)](http://www.informit.com/articles/article.aspx?p=31473&seqNum=2)

- [The ”Empty Member” C++ Optimization](http://www.cantrip.org/emptyopt.html)

- Internals of boost::compressed_pair

  http://www.ibm.com/developerworks/aix/library/au-boostutilities/index.html 

  https://en.wikibooks.org/wiki/Category%3A