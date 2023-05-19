# 建构追踪器 (Construction Tracker)

##目的
对于同时需要初始化多个对象的建构函数，如果出现异常，可以标识出具体抛出异常的成员变量。此惯用法，用于跟踪对象的构造和销毁。它通常用于调试和性能分析，以帮助开发人员了解程序中每个对象的生命周期和资源使用情况。

##别名

##动机
当在构造函数的初始化列表存在多个对象时，一旦有异常发生，它们抛出的异常是相同的(std::exception), 追踪出具体抛出异常的类就需要点技巧了，因为整个初始化列表是处于一个try代码段的，它还有一个专属的名称:构造函数try代码段(constructor try block), 其实无非就是'function-try block'。

##解决方案及示例
本惯用法使用了一个简单的技术就能成功的追踪到初始化列表中的对象。使用一个计数器，每个对象成功初始后就进行累加，它巧妙地使用括号运算符将计数器递增注入到构造函数的调用之间，所有这些都对类的使用者不可见。

```
#include <iostream>
#include <stdexcept>
#include <cassert>

struct B{
    B(char const*){
    	throw std::runtime_error("B Error"); 
    }
};
struct C{
    C(char const*) { 
    	throw std::runtime_error("C Error"); 
    }
};
class A {
   B b_;
   C c_;
   enum TrackerType { NONE, ONE, TWO };
public:
   A(TrackerType tracker = NONE)
   try    // 建构函数try代码段
     : b_((tracker = ONE, "hello")) // 抛出std::exception
     , c_((tracker = TWO, "world")) // 抛出std::exception
     {
        assert(tracker == TWO);
        //... 建构体 ...
     }
   catch (std::exception const& e)
     {
        if (tracker == ONE) {
           std::cout << "B threw: " << e.what() << std::endl;
        }
        else if (tracker == TWO) {
           std::cout << "C threw: " << e.what() << std::endl;
        }
        throw;
     }
};

int main (void)
{
    try {
        A a;
    }
    catch (std::exception const & e) {
          std::cout << "Caught: " << e.what() << std::endl;
    }
    return 0;
}
```
其中用双括号来包含tracker赋值的语句。这个惯用法要求构造函数必须要有一个参数。如果没有参数，就需要一个适配类来接受一个假的参数，然后跳用默认建构函数， 这个适配类可以通过Parameterized Base Class惯用法完实现。这个适配类也可以封装到A的内部。在A的构造函数中，追踪变量有一个默认值，所以用户也无法感知。

**备注：对上面的代码，做个解释，b_((tracker = ONE, "hello"))向只有一个参数的构造函数的对象传入由逗号运算符分割的表达式(tracker = ONE, "hello")，tracker = ONE因为不是char const*类型，所以它就被忽略了。相反，“hello”被传递给了构造函数。**

##已知的应用

##相关的惯用法

##参考
* [Smart Pointers Reloaded (III):Constructor Tracking](http://erdani.org/publications/cuj-2004-02.pdf)

