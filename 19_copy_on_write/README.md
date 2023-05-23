# 写时拷贝 (Copy on Write)

##目的
达到延迟拷贝(lazy copy)的优化目的。和延迟初始化(lazy initialization)相似, 选择在恰当的时机更加有效。

copy on write（简称COW）是一种优化策略，其核心思想是，如果有多个对象同时引用相同的资源（如内存或数据），它们会共享同一个指针指向同一个资源，直到某个对象试图修改资源的内容时，才会真正复制一份专用副本给该对象，而其他对象所引用的资源仍然保持不变。这样可以减少不必要的复制开销，提高性能和内存利用率。

在C++中，COW技术曾经被用于标准库中的std::string类，在C++98/C++03标准中是允许std::string采用COW策略的。这意味着当多个std::string对象赋值为相同的字符串时，它们实际上共享同一块字符数组的内存，只有当其中一个对象修改字符串内容时，才会分配新的内存并复制字符串。这样可以避免频繁的字符串拷贝和内存分配。

但是，在C++11标准中，为了提高并发性和避免线程安全问题，取消了std::string的COW策略。这意味着每个std::string对象都拥有自己独立的字符数组的内存，不会与其他对象共享。这样可以避免多线程环境下对同一块内存的竞争和同步。

因此，在C++中使用COW技术需要权衡其优缺点，以及考虑不同标准和编译器的实现细节。COW技术可以提高性能和内存利用率，但也可能引入写时复制开销、数据一致性问题和线程安全问题。

##别名
* COW (copy-on-write)
* Lazy copy

##动机
拷贝对象有时会带来性能损失(performance penalty)。如果对象经常拷来拷去，但是后续很少修改，使用copy-on-write就能明显地提升性能。为了实现copy-on-write, 需要使用一个智能指针将真正的对象值封装起来，每次修改时都要检查一下对象的引用计数。如果对象被多次引用，就在修改前创建一个副本。

##解决方案及示例

```
#ifndef COWPTR_HPP
#define COWPTR_HPP

#include <memory>

template<class T>
class CowPtr
{
public:
    typedef std::shared_ptr<T> RefPtr;

private:
    RefPtr m_sp;

    void detach()
    {
        T* tmp = m_sp.get();
        if( !( tmp == 0 || m_sp.unique() ) ) {
            m_sp = RefPtr( new T( *tmp ) );
        }
    }

public:
    CowPtr(T* t)
    :   m_sp(t)
    {}
    CowPtr(const RefPtr& refptr)
    	:   m_sp(refptr)
    {}
    const T& operator*() const
    {
    	return *m_sp;
    }
    T& operator*()
    {
    	detach();
    	return *m_sp;
    }
    const T* operator->() const
    {
    	return m_sp.operator->();
    }
    T* operator->()
    {
    	detach();
    	return m_sp.operator->();
    }
};

#endif
```
    译注:原文代码使用boost库，都改为std的实现了。
    const重载的知识：
    对于函数值传递的情况，因为参数传递是通过复制实参创建一个临时变量传递进函数的，函数内只能改变临时变量，但无法改变实参，则这个时候无论加不加const对实参不会产生任何影响。
    但是在引用或指针传递函数调用中，因为传进去的是一个引用或指针，这样函数内部可以改变引用或指针所指向的变量，这时const 才是实实在在地保护了实参所指向的变量。
    因为在编译阶段编译器对调用函数的选择是根据实参进行的，所以，只有引用传递和指针传递可以用是否加const来重载。
    值进函数前都会被拷贝的，const重载不重载主要是看能否改变传入的实参！！！
    参考：https://blog.csdn.net/weixin_42073412/article/details/99876532
这是一个简单的实现版本。除了必须通过智能指针解引用(dereferencing)来引用其内部对象有点不太方便外，还至少有一个缺点:类可以返回内部状态的引用:
```char& String::operator[](int)```
这样会带有一些无法预期的行为。

考虑下面的代码段:
```
CowPtr<std::string> s1 = new std::string("Hello");
char &c = s1->operator[](4); // 非常量的detach操作什么也不做
CowPtr<std::string> s2(s1); // 延迟拷贝，共享的状态
c = '!'; // 悲催啦
```
最后一行原本要修改原始的字串`s1`, 而不是它的复本`s2`，而事实上`s2`也被修改了。

一个比较好的做法是写一个自定义的copy-on-write实现，封装需要延时拷贝(lazy-copy)的类，并且保持对用户透明。为了解决上面的问题，可以标记对象为"不可共享(unshareable)"状态表示已经交出了对内存对象的引用，也就是强制进行深度拷贝。进一步优化，可以在那些不会放弃内部对象引用的non-const操作后恢复为"共享(shareable)"状态，(比如, void string::clear()))，因为客户端代码期望这些引用都会失效。

    译注:这一部分说得不清楚。标记对象为不可共享，比如上面例子中，取出字符c后设为不可共享，再建构s2时直接进行深拷贝。另外说在non-const操作没有放弃内部对象，指的是这类操作创建了一个复本，这时候的原来的对象可以更新为shareable。

再举个例子说明下COW：

假设我们有以下代码：

```cpp
#include <iostream>
#include <string>
using namespace std;

int main()
{
    string s1 = "Hello"; // 创建一个std::string对象s1，分配内存并存储字符串"Hello"
    string s2 = s1; // 创建一个std::string对象s2，赋值为s1
    cout << "s1's address: " << s1.c_str() << endl; // 输出s1的字符串地址
    cout << "s2's address: " << s2.c_str() << endl; // 输出s2的字符串地址
    s1[0] = 'h'; // 修改s1的第一个字符为'h'
    cout << "s1's address: " << s1.c_str() << endl; // 输出s1的字符串地址
    cout << "s2's address: " << s2.c_str() << endl; // 输出s2的字符串地址
    return 0;
}
```

如果我们使用C++98/C++03标准编译并运行这段代码，可能会得到以下输出：

```
s1's address: 0x55a7c9a8f0e0
s2's address: 0x55a7c9a8f0e0
s1's address: 0x55a7c9a8f100
s2's address: 0x55a7c9a8f0e0
```

我们可以看到，在赋值操作之后，s1和s2的字符串地址是相同的，说明它们共享同一块内存。这是因为std::string采用了COW策略，避免了不必要的字符串拷贝。但是，在修改操作之后，s1的字符串地址发生了变化，说明它分配了新的内存并复制了字符串。这是因为std::string在写时才进行了真正的拷贝，保证了数据的一致性。

如果我们使用C++11标准编译并运行这段代码，可能会得到以下输出：

```
s1's address: 0x55a7c9a8f0e0
s2's address: 0x55a7c9a8f100
s1's address: 0x55a7c9a8f0e0
s2's address: 0x55a7c9a8f100
```

我们可以看到，在赋值操作之后，s1和s2的字符串地址就不同了，说明它们各自拥有自己独立的内存。这是因为std::string取消了COW策略，避免了多线程环境下对同一块内存的竞争和同步。在修改操作之后，s1和s2的字符串地址没有变化，说明它们没有进行额外的拷贝。



##已知的应用

* Active Template Library
* Many Qt classes ([implicit sharing](http://doc.qt.io/qt-4.8/implicit-sharing.html))

##相关的惯用法

##参考
* Herb Sutter, More Exceptional C++, Addison-Wesley 2002 - Items 13–16
* [Wikipedia: Copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write)

