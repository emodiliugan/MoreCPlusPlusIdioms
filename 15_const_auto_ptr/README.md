# 				const auto_ptr

##目的

防止持有资源的所有权转移。

**注意：auto_ptr 已经在 C++11 中被弃用，并由 shared_ptr、unique_ptr 和 weak_ptr 代替，因此不再建议使用该习惯用法。**

##别名
##动机

在编程中，经常需要通过编译器来强制执行不可转让所有权的设计决策。在这里，所有权指的是任何资源的所有权，例如内存、数据库连接等。如果我们不希望所获取的资源的所有权在作用域之外或从一个对象转移到另一个对象，则可以使用 const auto_ptr 这个惯用法。auto_ptr 没有任何 cv 限定符（const 和 volatile）时具有移动语义，即在赋值操作中，它会将内存的所有权从右侧对象转移到左侧对象，但同时保证该资源始终只有一个所有者。const  auto_ptr 可以防止资源所有权的转移。

##解决方案及示例

将持有内存资源的 auto_ptr 声明为 const。

```
const auto_ptr  xptr(new X()); 

auto_ptr  yptr(xptr); // 不允许，编译错误。

xptr.release(); // 不允许，编译错误。

xptr.reset(new X()); // 不允许，编译错误。
```

编译器会发出警告，因为 yptr 的拷贝构造函数实际上是一个移动构造函数，它接受一个非 const 引用的 auto_ptr，这是在移动构造函数惯用法中所描述的。由于非 const 引用不能与 const 变量绑定，因此编译器会报错。

##影响
 使用 const auto_ptr惯用法的一个不良后果是，当一个类中包含const auto_ptr成员时，编译器无法为该类提供默认的拷贝构造函数。这是因为编译器生成的拷贝构造函数总是以 const 右值引用作为参数，而这无法与  auto_ptr 的非 const 移动构造函数绑定。解决方案是使用虚拟构造函数惯用法或使用  boost::scoped_ptr，后者通过拒绝访问赋值和拷贝构造函数来显式禁止复制。

举个例：

1.auto_ptr

```
#include <memory>
class A {
public:
    A() : ptr(new int(42)) {}
private:
    const std::auto_ptr<int> ptr;
};

int main() {
    A a1;
    A a2(a1); //error!编译器无法为该类提供默认的拷贝构造函数
    return 0;
}
```

2.scoped_ptr

```
#include <boost/scoped_ptr.hpp>

class A {
public:
    A() : ptr(new int(42)) {}
private:
    boost::scoped_ptr<int> ptr;
};

int main() {
    A a1;
    A a2(a1); //error!显式禁止了拷贝构造函数
    return 0;
}
```

##已知的应用

##相关的惯用法

Move Constructor

##参考

 http://www.gotw.ca/publications/using_auto_ptr_effectively.htm 

http://www.boost.org/libs/smart_ptr/scoped_ptr.htm