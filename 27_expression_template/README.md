# 模板表达式(Expression Template)

##目的

- 在C++中创建特定于领域的嵌入式语言 （DSEL）
- 支持对C++表达式（例如数学表达式）的惰性求值，这些表达式可以在程序晚于其定义的地方执行。
- 将表达式（而不是表达式的结果）作为参数传递给函数。

##别名
##动机

领域特定语言 （DSL） 是一种开发程序的方法，其中要解决的问题是使用更接近问题域的表示法来表达，而不是过程语言提供的通常表示法（循环、条件等）。特定于领域的嵌入式语言（DSEL）是DSL的一种特例，其中符号**嵌入**在宿主语言中（例如，C++）。基于C++的DSEL的两个突出例子是[Boost Spirit Parser Framework](http://spirit.sourceforge.net)和[Blitz++](http://www.oonumerics.org/blitz)科学计算库。Spirit提供了一种符号，可以将**EBNF**语法直接写入C++程序中，而Blitz++允许符号对矩阵进行数学运算。显然，C++本身并没有提供这种符号。使用这种符号的主要好处是程序非常直观地表达了程序员的意图，使程序更具可读性。它**大大降低**了开发和维护成本。

那么，这些库（Spirit和Blitz++）是如何在抽象层面实现这种提高C++能力的方法呢？ 是的，你猜对了，答案是**模板表达式**. 

表达式模板背后的关键思想是对表达式的**惰性计算**。C++ 本身不支持表达式的延迟计算。例如，在下面的代码片段中，加法表达式 （x+x+x） 在调用函数 foo 之前执行。

```
int x;
foo(x + x + x); //只有在这行才有加法加法表达式
```

函数 foo 永远不会真正的知道它收到的参数是如何计算的。加法表达式在其第一次也是唯一一次计算之后从未真正存在过。c++的这种默认行为对于现实世界的绝大多数程序来说是必要且充分的。但是，有些程序稍后需要表达式来一次又一次地计算它。例如，将数组中的每个整数增加三倍。

```
int expression(int x)
{
	return x + x + x; //注意相同的表达式
}
// .... 这里有更多其他的代码
const int N = 5;
double A[N] = { 1, 2, 3, 4, 5};
std::transform(A, A+N, A, std::ptr_fun(expression)); // 将每个整数增加三倍
```

这是C/C++中支持数学表达式惰性求值的传统方法。表达式被包装在函数中，函数又作为参数传递。在这种技术中，函数调用和临时变量的生成都会带来性能上的开销，而且很多时候，表达式在源代码中的位置离调用的地方很远，这会对可读性和可维护性产生很不好的影响。表达式模板通过内联表达式的方式来解决问题，这消除了对函数指针的需求，并将表达式和调用的地方更加紧密的结合在一起。

##解决方案及示例

表达式模板使用[Recursive Type Composition惯用法](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Recursive_Type_Composition)。递归类型组合使用类模板的实例，这些实例包含同一模板作为成员变量的其他实例。**同一模板的多次重复实例化会产生类型的抽象语法树 （AST）**。递归类型组合已用于创建线性类型列表以及以下示例中使用的二叉表达式树。

```
#include <iostream>
#include <vector>
struct Var {
	double operator()(double v) { 
		return v; 
	}
};
struct Constant {
    double c;
    Constant(double d) : c (d) {}
    double operator()(double) 
    { 
    	return c; 
    }
};
template < class L, class H, class OP >
struct DBinaryExpression {
	L l_;
	H h_;
    DBinaryExpression(L l, H h) : l_ (l), h_ (h) {}
    double operator()(double d){ 
    	return OP::apply (l_ (d), h_(d)); 
    }
};
struct Add {
	static double apply(double l, double h){ 
		return l + h; 
	}
};
template < class E >
struct DExpression {
    E expr_;
    DExpression(E e) : expr_(e) {}
    double operator()(double d){ 
    	return expr_(d); 
    }
};
template <class Itr, class Func >
void evaluate(Itr begin, Itr end, Func func)
{
    for (Itr i = begin; i != end; ++i)
    	std::cout << func (*i) << std::endl;
}
int main (void)
{
    typedef DExpression<Var> Variable;
    typedef DExpression<Constant> Literal;
    typedef DBinaryExpression<Variable , Literal, Add> VarLitAdder;
    typedef DExpression<VarLitAdder> MyAdder;
    Variable x((Var()));
    Literal l(Constant (50.00));
    VarLitAdder vl_adder(x, l);
    MyAdder expr(vl_adder);
    std::vector<double> a;
    a.push_back(10);
    a.push_back(20);
    // It is (50.00 + x) but does not look like it.
    evaluate(a.begin(), a.end(), expr);
    return 0;
}

```

**注释：这里先对“同一模板的多次重复实例化会产生类型的抽象语法树 （AST）”这句话进行一个解释。**

**在C++中，模板是一种通用的编程工具，它允许程序员编写可重用的代码，以处理各种不同类型的数据。当程序在编译时实例化一个模板时，编译器会生成相应的代码，用于处理特定的数据类型。如果同一个模板被多次实例化，就会生成多个不同的代码片段，这些代码片段之间可以通过抽象语法树进行比较和分析，以便进行优化和其他操作。**

**举个例子：**

**假设我们有一个模板类`MyTemplate`，它有一个模板参数`T`，并定义了一个成员函数`print()`，用于打印`T`类型的数据。现在我们在程序中多次对`MyTemplate<int>`进行实例化，如下所示：**

**MyTemplate<int> a, b, c;**

**这样就会生成多个不同的`MyTemplate<int>`对象，每个对象都有自己的代码实现。这些不同的代码实现之间可以通过类型的抽象语法树进行比较和分析，以便进行优化和其他操作。**

**例如，编译器可以使用抽象语法树来检测是否有重复的代码片段，以便进行代码重用和优化。抽象语法树还可以用于模板元编程，以实现在编译时生成代码的高级技术，例如模板元函数和模板元类**。

这里与Composite设计模式比较来说更有用处点。模板 DExpression 可以被看作Composite设计模式中的抽象基类。它表达了接口中的共性。在expression templates中，通用接口是运算符重载的函数。DBinaryExpression运用了组合，同时也运用了适配（它适配了Add的接口到DExpression中）。常量和变量是两种不同类型的叶节点，并且它们还遵循DExpression的接口。DExpression将DBinaryExpression，常量和变量的复杂性都隐藏在一个统一的接口后面，使它们能协同工作。任何的二元运算符重载（比如除法、乘法等）都可以替代加法。

上面的例子并没有显示如何在编译时生成递归类型。此外，expr实际上是一个数学表达式，虽然看起来根本不像是一个数学表达式。下面的代码演示如何使用重载 + 运算符的重复实例化递归组合类型。

```
template<class A, class B>
DExpression<DBinaryExpression<DExpression<A>,DExpression<B>,Add>>
operator+(DExpression<A> a, DExpression<B> b)
{
	typedef DBinaryExpression<DExpression<A>,DExpression<B>,Add>  ExprT;
	return DExpression<ExprT>(ExprT(a,b));
}
```



上面的重载+运算符做了两件事 ：

- 它添加了语法糖

- 启用递归类型组合，受编译器的限制

  

  因此，它可以被 调用evaluate替换，如下所示：

  ```
  evaluate (a.begin(), a.end(), x + l + x);
  /// 它是(2*x + 50.00), 看起来确实像一个数学表达式。
  ```

##已知的应用

- [Blitz++ Library](http://www.oonumerics.org/blitz)
- [Boost Spirit Parser Framework](http://spirit.sourceforge.net)
- [Boost Basic Linear Algebra](http://www.boost.org/libs/numeric/ublas)
- [LEESA: Native XML Programming using an Embedded Query and Traversal Language in C++](http://www.dre.vanderbilt.edu/LEESA)
- [std::valarray as implemented by GNU libstdc++](http://gcc.gnu.org/viewcvs/trunk/libstdc%2B%2B-v3/include/std/valarray?view=co) , [LLVM libc++](http://llvm.org/svn/llvm-project/libcxx/trunk/include/valarray) , and others

##相关的惯用法

Recursive Type Composition(Chapter 55.0.7)

##参考

- [Expression Templates](http://www.angelikalanger.com/Articles/Cuj/ExpressionTemplates/ExpressionTemplates.htm) - Klaus Kreft & Angelika Langer
- [Expression Templates](http://web.archive.org/web/20050315091836/http://osl.iu.edu/~tveldhui/papers/Expression-Templates/exprtmpl.html) - Todd Veldhuizen
- [Faster Vector Math Using Templates](http://www.flipcode.com/archives/Faster_Vector_Math_Using_Templates.shtml) - Tomas Arce