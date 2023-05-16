# 能力查询 (Capability Query)
##概要

用 dynamic cast 检查 interface 设计的多继承。但是 dynamic cast 需要开启 rtti，而且没有虚函数的用不了。并且，RTTI 也是同样的需要至少一个虚函数。不过，这个限制就是当你把 dtor 写成 virtual 的时候就自动满足了。对于一些情况，与其走这个虚表查询，不如你自己做一个 virtual flag 在基类.... OOP 罪恶。可以用 double dispatch 避免 instanceof/dynamic_cast  (但是 GoF 中用双分派加强耦合是因为当时 C++11 没出，没有 RTTI)。我认为不如用一个 virtual flag/enum 了。但是 dynamic_cast 又是必不可少的，只要用到 interface based polymorphism（即多继承体系，以及连续继承等），而 type flag/virtual falg 只支持类似于 abstract class、单一 interface 多个不同子类等，即超绝耦合，父类需要知道子类的信息（当然，用一种额外的 id 的话确实可以不修改父类，总之方法有很多啦）。

##目的
在运行时检查一个对象是否支持某个接口。

##别名

##动机
将实现分隔出不同的接口是一个较好的面向对象的软件设计实践。在C++中，接口类(Interface Class)惯用法可以将实现分隔为多个接口，再运行时可以通过多态调用到公共方法。下面是一个实现多种接口的示例：
```
class Shape { // 一个接口类
   public:
    virtual ~Shape();
    virtual void draw() const = 0;
    //...
};
class Rollable { // 另一个接口类
  public:
    virtual ~Rollable();
    virtual void roll() = 0;
};
class Circle : public Shape, public Rollable { // 可以旋转的圆 - 实体类
    //...
    void draw() const;
    void roll();
    //...
};
class Square : public Shape { // 不能旋转的方形 - 实体类
    //...
    void draw() const;
    //...
};
```
对于一个指向Rollable类的指针，可以简单地用如下的方式调用`roll`函数，正是接口类(Interface Class)惯用法。
```
std::vector<Rollable*> rollables;
// 在别的地方生成一组对象。
for(vector<Rollable*>::iterator iter(rollables.begin());
     iter != rollables.end();
     ++iter)
{
  	iter->roll();
}
```
问题来了，对通过多重继承派生出来的类，在运行时并不知道一个对象是否实现某个特别的接口。这时候就需要使用这个惯用法。

##解决方案及示例
在C++里，可以通过`dynamic_cast`查询类的类型。
```
Shape *s = getSomeShape();
if (Rollable *roller = dynamic_cast<Rollable*>(s))
  roller->roll();
```
`dynamic_cast`通常被称为**cross-cast**（跨级转换）,因为它将尝试在一个继承结构进行转换，而不是简单的向上或向下转换。在这个例子里，通过`dynamic_cast`转换为`Rollable`时，   只有`Circle`类的对象可以成功。`Square`类因为没有实现`Rollable`，所以一定无法转换。

过多的使用这种方式一般表示设计有问题了。

##已知的应用
* [Acyclic Visitor Pattern (非循环访问者模式)](http://www.objectmentor.com/resources/articles/acv.pdf) - Rober C. Martin
##相关的惯用法

##相关的惯用法
* Interface Class
* Inner Class

##参考
Capability Queries - C++ Common Knowledge by Stephen C. Dewhurst
