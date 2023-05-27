# enable-if

##目的

允许基于任意类型进行函数重载

##别名

- Explicit Overload Set Management

##动机

enable_if系列模板是一组工具，它允许函数模板或类模板的特化，根据其模板参数的属性从一组匹配的函数或特化中，包含或者排除自身。例如，可以定义一个函数模板，它仅对由traits类定义的任意类型集合启动，从而匹配该集合。enable_if模板也可以应用在启动类模板特化。它的应用在许多文献里都被广泛的讨论（如：参考1，2）。

C++里许多合理的模板函数重载的操作都依赖于SFINAE（substitution-failure-is-not-an-error替换失败不是错误）原则：如果在函数模板实例化期间形成无效的参数或者返回类型，则将实例化从重载集中移除，而不是导致编译错误。下面的示例演示了为什么这一点很重要：

```
int negate(int i) 
{ 
	return -i; 
}
template <class F>
typename F::result_type negate(const F& f) 
{ 
	return -f();
}
```

假设编译器调用 negate(1)。第一个定义显然是更好的匹配，但编译器仍必须考虑（并实例化模板）这两个定义才能找到这一点。将后一个定义实例化 F 为 int 将导致以下结果：

```
int::result_type negate(const int&);
```

这个返回值是无效的。如果编译器将无效的实例化视为错误，并添加一个与代码无关的函数模板可能会导致编译错误，进而破坏原本有效的代码。遵照SFINAE原则，上面的例子不是错误的。后面negate的定义仅从重载集中移除。

enable_if 模板是一种条件控制 SFINAE 创建的工具。

##解决方案及示例

enable_if 模板在语法实现上非常简单。它们总是**成对出现：其中一个为空，另一个具有一个typedef类型** **，该 typedef 将其第二个类型参数转发**。空的struct因为它不包含任何成员，将是一个无效的类型。当编译时条件为 false 时，编译器会选择空的 enable_if 模板，因为此时需要选择一个无效的类型，以触发 SFINAE 。

```
template <bool, class T = void>
struct enable_if
{};
template <class T>
struct enable_if<true, T>
{
	typedef T type;
};

```

下面这个示例，展示了如何基于任意的类型参数，在编译时选择重载的模板函数。假设函数T foo(T t) 可以接收所有的arithmetic类型。可以将 enable_if 模板作用于返回类型，就像这个示例中的用法一样。

```
template <class T>
typename enable_if<is_arithmetic<T>::value, T>::type
foo(T t)
{
	// ...
	return t;
}
```

或者作为一个额外的参数，就像下面的例子中所示：

```
template <class T>
T foo(T t, typename enable_if<is_arithmetic<T>::value >::type* dummy = 0);
```

对于 foo() 添加的这个额外参数给定了一个默认值。由于调用 foo() 的程序将忽略这个虚拟参数，因此可以给它任意的类型。甚至，我们可以允许它是 void *。有了这个想法，我们可以简单地省略 enable_if 的第二个模板参数。这意味着当 is_arithmetic<T> 为 true 时，enable_if<...>::type 表达式得到的结果为 void。

是将enabler写为参数还是在返回值类型中，很大程序取决于个人的喜好，但是对于某些特殊函数来说，只有一个选择的可能性：

- 操作符运算具有固定数量的参数，因此必须在返回类型中使用 enable_if。

  例如：

  ```
  template<typename T>
  typename std::enable_if<std::is_integral<T>::value, T>::type
  operator+(const T& a, const T& b) {
      return a + b;
  }
  //这里使用 enable_if 确保只有当 T 是整数类型时才能进行加法运算。
  ```

  

- 构造函数和析构函数没有返回类型；额外的参数是唯一的选择。

  ```
  class MyClass {
  public:
      template<typename T>
      MyClass(T t, typename 					    std::enable_if<std::is_pointer<T>::value>::type* = nullptr) {
          // 这里在构造函数中使用 enable_if，以便只有当参数 t 是指针类型		   //时才会执行特定的操作。
      }
  };
  ```

  

- 似乎没有办法为转换运算符指定一个enabler。但是，转换构造函数可以具有enabler作为额外的默认参数。

  ```
  //转换运算符
  class MyDouble {
  public:
      operator double() const {
          // do something to convert MyDouble to double
      }
  };
  这里定义了一个将 MyDouble 类型转换为 double 类型的转换运算符。由于转换运算符的返回类型是固定的，即 double，因此无法在返回类型中使用 enable_if 来控制转换运算符的行为。
  
  //转换构造函数
  class MyClass {
  public:
      template<typenameT>
      explicit MyClass(T t, typename 						std::enable_if<std::is_integral<T>::value>::type* = nullptr) 	 {
          // do something when t is an integer
      }
  };
  这里在构造函数中使用 enable_if，以便只有当参数 t 是整数类型时才会执行特定的操作。注意，这里使用了 explicit 关键字，以避免在不希望进行隐式转换的情况下创建对象。
  ```

  

##已知的应用

Boost library, C++ STL, etc.

##相关的惯用法

SFINAE

##参考

1.Jaakko Järvi, Jeremiah Willcock, Howard Hinnant, and Andrew Lumsdaine. Function overloading based on arbitrary properties of types. C/C++ Users Journal, 21(6):25--32, June 2003.

2.Jaakko Järvi, Jeremiah Willcock, and Andrew Lumsdaine. Concept-controlled polymorphism. In Frank Pfennig and Yannis Smaragdakis, editors, Generative Programming and Component Engineering, volume 2830 of LNCS, pages 228--244. Springer Verlag, September 2003.

3.[Boost Libraries enable-if documentation](http://www.boost.org/libs/utility/enable_if.html)