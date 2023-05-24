# 侵入式引用计数 (Counted Body / Intrusive Reference Counting)

##目的
资源或对象的逻辑共享的管理，避免代价太大的拷贝，并且对于动态分配的资源可以完整的释放。

##别名
* Reference Counting (intrusive)
* Counted Body

##动机
运用Handle/Body惯用法时，实体(Body)的拷贝的代价太大，因为拷贝涉及内存分配和拷贝构造。这种拷贝可以通过使用指针或引用来避免，但这留下了一个问题，即谁负责清理对象。通常最后一个句柄(Handle)必须负责为实体(Body)分配的内存进行释放。如果没有自动回收内存机制，就会导致内置类型和用户定义类型的行为和表现将会有所不同，这种区别可能会对程序的正确性和可维护性产生负面影响。

##解决方案及示例
解决方案是在实体(Body)类中增加引用计数来帮助内存管理，因为得名"Counted Body"(计数体)。内存管理添加到Handle类，特别是它的初始化、赋值、拷贝和析构。通过增加和减少引用计数，可以管理对象的生命周期，并确保适当的资源回收。

```
这段代码定义了两个类：StringRep和String。
StringRep是一个辅助类，用于存储字符串数据和计数器。String类是一个字符串类，它使用StringRep类来存储字符串数据。
String类有四个构造函数：一个默认构造函数，一个从另一个String对象构造的构造函数，一个从char指针构造的构造函数，以及一个赋值运算符。赋值运算符使用了拷贝并交换技巧，以确保在分配期间不会出现内存泄漏。此外，String类还有一个swap函数，用于交换两个String对象的StringRep指针.

#include <cstring>
#include <algorithm>
#include <iostream>

class StringRep {
  friend class String;

  friend std::ostream& operator<<(std::ostream& out, StringRep 										const& str) {
    out << "[" << str.data_ << ", " << str.count_ << "]";
    return out;
  }

public:
  StringRep(const char* s) : count_(1) {
    strcpy(data_ = new char[strlen(s) + 1], s);
  }

  ~StringRep() { delete[] data_; }

private:
  int count_;
  char *data_;
};

class String {
public:
  String() : rep(new StringRep("")) {
    std::cout << "empty ctor: " << *rep << "\n";
  }
  String(const String& s) : rep(s.rep) {
    rep->count_++;
    std::cout << "String ctor: " << *rep << "\n";
  }
  String(const char* s) : rep(new StringRep(s)) {
    std::cout << "char ctor:" << *rep << "\n";
  }
  String& operator=(const String& s) {
    std::cout << "before assign: " << *s.rep << " to " << *rep << 					"\n";
    String(s).swap(*this);  // copy-and-swap idiom
    std::cout << "after assign: " << *s.rep << " , " << *rep << 				"\n";
    return *this;
  }
  ~String() {  // StringRep deleted only when the last handle 								//goes out of scope.
    if (rep && --rep->count_ <= 0) {
      std::cout << "dtor: " << *rep << "\n";
      delete rep;
    }
  }

 private:
  void swap(String &s) throw() { std::swap(this->rep, s.rep); }

  StringRep* rep;
};
int main() {

  std::cout << "*** init String a with empty\n";
  String a;
  std::cout << "\n*** assign a to \"A\"\n";
  a = "A";

  std::cout << "\n*** init String b with \"B\"\n";
  String b = "B";

  std::cout << "\n*** b->a\n";
  a = b;

  std::cout << "\n*** init c with a\n";
  String c(a);

  std::cout << "\n*** init d with \"D\"\n";
  String d("D");

  return 0;
}
```
惯用法避免了不必要的拷贝，从而实现更高效的程序。这个惯用法假设程序员可以修改实体(Body)类的代码。当这不可行话，还可以使用Detached Counted Body。把引用计数放到实体类内部称为侵入式(intrusive)引用计数。如果把引用计数存储在实体(Body)类外面就称为非侵入式引用计数。这个实现是浅拷贝的变种，兼有深拷贝的语义和Smalltalk 键值对(name-value)的效率。

输出结果如下:
```
*** init String a with empty
empty ctor: [, 1]

*** assign a to "A"
char ctor:[A, 1]
before assign: [A, 1] to [, 1]
String ctor: [A, 2]
dtor: [, 0]
after assign: [A, 2] , [A, 2]

*** init String b with "B"
char ctor:[B, 1]

*** b->a
before assign: [B, 1] to [A, 1]
String ctor: [B, 2]
dtor: [A, 0]
after assign: [B, 2] , [B, 2]

*** init c with a
String ctor: [B, 3]

*** init d with "D"
char ctor:[D, 1]
dtor: [D, 0]
dtor: [B, 0]
```
##影响
创建多个引用计数会导致同一实体(Body)对象的多次删除，这是具有未知风险的。必须格外留意对同一实体(Body)对象的创建多个引用计数。侵入式的引用计数基本没有这个问题。如果是使用非侵入式的引用计数，就要要求程序员自身来避免出现重复的引用计数的情况。

##已知的应用

* boost::shared_ptr (非侵入式的引用计数)
* boost::intrusive_ptr (侵入式的引用计数)
* std::shared_ptr
* Qt toolkit, 如QString

##相关的惯用法
* Handle Body
* Detached Counted Body (非侵入式引用计数)
* Smart Pointer
* Copy-and-swap

##参考
* [Advanced C++中文版](http://product.china-pub.com/16697) 9.5垃圾收集

