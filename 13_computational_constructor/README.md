# 计算构造函数 (Computational Constructor)
##概要

计算构造函数（Computational  Constructor）是一种C++编程技术，用于在对象创建时执行一些计算或者初始化操作，以确保对象在创建时处于一种有效状态。与传统的构造函数不同，计算构造函数不仅仅是用于初始化对象的数据成员，它还可以执行一些额外的计算和操作，例如打开文件、分配内存、初始化全局变量等等。计算构造函数通常用于实现复杂的类或者数据结构，以确保对象在创建时能够正确地初始化和配置。需要注意的是，计算构造函数可能会对性能产生影响，因为它们在对象创建时会执行额外的计算和操作，因此应该在必要时使用，并进行必要的优化。

##目的

* 优化按值返回(return-by-value)
* 允许不支持具名返回值优化(Named Return Value Optimization)的编译器进行返回值优化(Return Value Optimization, RVO)
##别名

##动机
在C++里返回较大的对象的代价是很高的。当在函数里通过值返回一个本地创建的对象时，会在栈上创建一个临时对象。这些临时对象的生命周期一般是很短的，因为它们要么被赋给其它对象，要么就是传递到其它函数。然后在创建它们的语句执行完成后就会被释放。

过去几年，编译器应用了若干的优化方法来避免这些临时对象的创建，毕竟它们有点浪费，而且对性能不利。其中两个常见的技术是返回值优化(Return Value Optimization, RVO)和具名返回值优化(NRVO), 以将临时变量优化掉，也被称为避免拷贝(copy elision)。按顺序，先介绍RVO。

### Return Value Optimization
下面的例子展示了清除一个或者两个拷贝操作所产生的副本的场景，其中的构造函数明显有一个副作用（打印文本）。第一个要被清除的拷贝是当`Data(c)`复制到函数`func`的返回值。第二次拷贝发生在`func`函数返回临时对象给`d1`.更多的解释，可以参考[Wikipedia上的解释](http://en.wikipedia.org/wiki/Return_value_optimization)。
```
struct Data {
  Data(char c = 0)
  {
    std::fill(bytes, bytes + 16, c);
  }
  Data(const Data& d)
  {
    std::copy(d.bytes, d.bytes+16, this->bytes);
    std::cout << "A copy was made.\n";
  }
private:
  char bytes[16];
};

Data func(char c) {
  return Data(c);
}

int main(void) {
   Data d1 = func(‘A’);
}
```

下面的伪代码展示了两次`Data`的拷贝是如何被清除掉的。

我们回忆一下placement new的语法：

```
语法：new (pointer) Type(arguments);
其中，pointer 是指向要创建对象的内存地址的指针，Type 是要创建的对象类型，arguments 是传递给对象构造函数的参数。使用 placement new 创建对象时，不会调用默认的 new 操作符，因为指定了对象的内存地址，需要保证这个地址是有效的，并且没有被其他对象使用。
```

```
void func(Data* target, char c)
{
  new (target) Data(c);  // placement-new 语法 (这里没有动态分配)

  return;                 // 注意void返回值.
}
int main (void)
{
   char bytes[sizeof(Data)];  //保存数据Data对象的栈空间
   func(reinterpret_cast<Data*>(bytes), 'A'); //Data的两个拷贝副本被												消除了
   reinterpret_cast<Data*>(bytes)->~Data();   //析构
}
```
### Named Return Value Optimization (NRVO)

NRVO是更高级的技术，也不是所有的编译器都能支持。注意上面函数`func`没有给它创建的局部对象命名。实际上大部分函数会比它复杂，它们会创建局部对象，操作它们的状态，然后返回更改后的对象。要想清除这一类的对象就需要应用NRVO。考虑一下的例子，特别是做了许多处理(运算, Computational)的那部分。
```
class File {
private:
  std::string str_;
public:
  File() {}
  
  void path(const std::string& path) {
    str_ = path;
  }
  
  void name(const std::string& name)  {
    str_ += "/";
    str_ += name;
  }
  
  void ext(const std::string& ext) {
    str_ += ".";
    str_ += ext;
  }
};

File getfile(void) {
  File f;
  f.path("/lib");
  f.name("libc");
  f.ext("so");
  f.ext("6");

  // 因为命名为f, RVO就无能为力了。
  // 这里需要应用NRVO, 但不是所有的编译器都能做到。
  return f;
}

int main (void) {
  File  f = getfile();
}
```
上面的例子里，函数`getfile`在返回对象`f`前，会对`f`做一系列的操作。因为`f`对象有它的名字，所以RVO不可用，只能指望NRVO，但它又没被普遍的实现。这个惯用法就是针对这种场景提供返回值优化的效果。

##解决方案及示例
这个惯用法背后的逻辑是为了利用RVO,将处理操作移到一个建构函数中，这样编译器就更原意做些优化。新增一个建构函数就可以让RVO生效，这个建构函数带四个参数，在下面例子里就是类`File`中的处理用(computational)构造函数。`getFile`函数现在就简单多了，编译器也可以执行RVO了。

```
class File
{
private:
  std::string str_;
public:
  File() {}

  // The following constructor is a computational constructor
  File(const std::string & path,
       const std::string & name,
       const std::string & ext1,
       const std::string & ext2)
    : str_(path + "/" + name + "." + ext1 + "." + ext2) { }

  void path(const std::string & path);
  void name(const std::string & name);
  void ext(const std::string & ext);
};

File getfile(void) {
  return File("/lib", "libc", "so", "6"); // RVO可用了
}

int main (void) {
  File  f = getfile();
}
```

##影响
对于这个惯用法的一个常见批评说这样会导致不太自然的构造函数，对于上面展示的File类而言，说的是有一定道理的。如果谨慎地使用这个惯用法，可以限值计算构造函数在一个类中的扩散，同时也能得到更好的运行时性能。

##已知的应用

##相关的惯用法

##参考
* Efficient C++; Performance Programming Techniques，Dov Bulka, David Mayhew
